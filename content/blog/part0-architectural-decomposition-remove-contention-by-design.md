+++
title = "Architectural Decomposition: Remove Contention by Design"
date = 2026-06-22
description = "The opening post of a Low-Level Systems Design in Rust series, arguing that the biggest performance win is structural: replace a shared multi-producer cursor that serializes cores through cache-coherence traffic with one private single-writer SPSC ring per producer, converting quadratic writer contention into a cheap O(N) consumer sweep."
template = "page.html"

[taxonomies]
tags = ["rust", "concurrency", "lock-free", "spsc", "ring-buffer", "cache-coherence", "false-sharing", "systems-programming", "performance", "scalability", "optimization"]
[extra]
+++

# Architectural Decomposition: Remove Contention by Design

Across the Rust projects I've worked on, the ones that demanded high throughput and low latency were almost always built around a few architectural principles that remained invariant across all of them. Many of these principles are not limited to Rust, but apply equally, with implementation-specific variations, to lower-level languages like C++ or Zig.

In this series of blog posts I plan to mine those design principles and prepare a collection of patterns that can be applied in similar contexts. I call the series **Low-Level Systems Design in Rust** - this is the opening installment of it. Most of the posts work at the micro level - cache lines, memory ordering, allocation - using a single-producer / single-consumer (SPSC) ring buffer as the running example.

But before any of that pays off, there's one decision that dwarfs all the others, and it's structural. This post is about that decision. The patterns aren't specific to ring buffers: they apply to any system where many writers feed a shared reader.

Three principles run underneath everything that follows, and the first one *is* this post:

1. **The biggest wins are architectural, not microscopic.** Eliminating a shared write cursor is worth orders of magnitude more than picking the "right" `Ordering`. Cache-line padding, atomic tuning, and inlining only matter *after* you've made the structural choice that removes contention in the first place.
2. **Optimize what the hardware actually does, not what the source looks like.** Modern CPUs speculate past branches, reorder loads, and bounce cache lines between cores under contention. Many also have adjacent-line or spatial prefetch behavior that can make a nominally "separate" cache line relevant to performance. Source-level cleverness that ignores those realities buys nothing.
3. **Measure, don't guess.** Every technique in this series earns its place against a benchmark - and benchmark the code you *think* you're benchmarking: Rust's optimizer is happy to delete work whose result is unused, so reach for `std::hint::black_box` and read the generated assembly before trusting any number.

Here's a rough idea of what I plan to cover as part of this series:

0. **Architectural Decomposition** *(this post)* - remove contention by design.
1. **Cache-Conscious Data Layout** - field zoning, false sharing, and cache-line and adjacent-line placement heuristics.
2. **The Cross-Core Contract** - memory ordering and single-writer state.
3. **Amortizing Cross-Core Coordination** - cursor caching and batch processing.
4. **Zero-Copy on the Hot Path** - reserve/commit and fast-path/slow-path splitting.
5. **Compile-Time Leverage** - specialization, branch & arithmetic hygiene, and inlining.
6. **Backoff & Memory Provisioning** - adaptive spinning, custom allocators, NUMA & huge pages.
7. **Safety as Performance** - moves, the borrow checker, `debug_assert!`, and cold-path hygiene.
8. **Trustworthy Performance** - verification, idioms, anti-patterns, and knowing when to stop.

Start here; the rest builds on the foundation this post lays.

---

## The most expensive instruction is the one two cores fight over

Here's the scenario that drives the whole design. You have *N* independent producers - threads, tasks, cores - that all need to hand work to a single consumer. Assume a bounded in-memory handoff path where per-producer FIFO is acceptable and the consumer can poll. The obvious implementation is one shared multi-producer queue: a single buffer with shared write-side state that every producer must update, often through an atomic read-modify-write such as a compare-and-swap or `fetch_add`.

That obvious implementation has a problem that gets *worse* as you add cores. Every producer is hammering the same `tail` cache line, or whatever cache line holds the shared write-side state. On a CAS or other read-modify-write, the line has to be in the **exclusive** coherence state on the writing core - which means other cached copies of that line must be **invalidated** before the write can complete. Two producers that want to append at the same instant don't run in parallel; the hardware coherence protocol serializes them, ping-ponging ownership of that one line from core to core. Add a third producer and it's worse. The shared cursor is a contention point, and the cost of touching it scales with the number of cores touching it.

This is the trap: the code *looks* lock-free - there's no mutex anywhere - but the shared atomic cursor reintroduces serialization at the hardware level. You've moved the lock from the OS into the cache-coherence fabric, where it's harder to see and you can't even name it.

## The decomposition: give every writer its own lane

The fix is to refuse the premise. Don't make independent writers share a structure at all. Give **each writer its own private single-writer structure**, and let the consumer poll across them:

```
Producer 0 ──► [ SPSC Ring 0 ] ──┐
Producer 1 ──► [ SPSC Ring 1 ] ──┼──► Consumer (sweeps the rings in turn)
Producer 2 ──► [ SPSC Ring 2 ] ──┘
        … one dedicated ring per producer …
```

Now each producer writes only to *its own* ring. A producer's append updates a cursor no other producer ever touches. In Rust that cursor is still an atomic value, because the consumer reads it from another thread, but the hot update is a simple atomic `store`, not an atomic read-modify-write. The result: **no CAS, no spin loop, and crucially no producer-producer cache-line bouncing.** Two producers appending at the same instant genuinely run in parallel, because they're writing to different cache lines on different rings. The defining property:

> *N* independent single-writer structures have **zero** coherence traffic on the hot cursor path *between the writers*.

The contention didn't vanish into thin air - it was *converted*. Instead of *N* cores serializing on one hot cursor, you have one consumer paying an **O(N) sweep** to poll the rings in turn. That's a fundamentally better trade when producer throughput dominates: a single thread doing a little more sequential work, in exchange for eliminating the quadratic-feeling coherence storm among the writers. Each individual ring is then a clean single-producer / single-consumer problem - which is exactly the structure the rest of this series spends its time optimizing.

This decomposition also changes the semantics. Each ring preserves FIFO order for one producer, but the system no longer has a natural global enqueue order across all producers. If the consumer needs a total order, you need sequence numbers, timestamps, or a merge step - and that reconciliation has its own cost.

## Be honest about the trade-offs

Decomposition isn't free, and the costs are worth stating plainly:

- **Memory grows with N.** One buffer per producer instead of one shared buffer.  For a few dozen producers this is negligible; for tens of thousands it's a real budget line.
- **Single-message latency can rise.** The consumer has to *poll* - a message sitting in ring 3 isn't seen until the sweep reaches ring 3. For small, hot sets of rings that overhead is often tiny compared with the contention it eliminates, but the exact number is hardware- and workload-dependent. For very large N you may need a smarter consumer: a ready-set, per-core consumers, or an event mechanism, all of which I will return to in later posts.
- **Backpressure becomes per-lane.** If one producer's ring is full, that producer has to block, drop, or spill somewhere else, while other producers may continue normally. That isolation is often useful, but it is a policy decision, not a free consequence.
- **Fairness is now explicit.** A consumer that drains one busy ring for too long can starve quieter rings. The sweep policy - round-robin, bounded drain, priority, batching - becomes part of the design.
- **Global ordering is no longer free.** Per-producer FIFO is cheap. Cross-producer total order requires extra metadata and merge work.

**When to reach for it:** any time the asymmetry between writers and readers is large - *many* writers, *one* (or few) readers - and the readers can tolerate an O(N) sweep. That's precisely the shape where a shared cursor hurts most and a per-writer structure helps most.

## You've seen this pattern before

Architectural decomposition is one of the most widely reinvented ideas in systems software, because the writer/reader asymmetry is everywhere:

- **Allocator sharding** - jemalloc uses arenas and thread caches, while mimalloc shards free lists and separates thread-local from concurrent free paths, so allocation and free traffic doesn't collapse onto one global structure.
- **Per-worker run queues** - schedulers like Tokio and Go keep local queues around workers, with global queues and work-stealing as reconciliation mechanisms instead of putting every operation through one central queue.
- **Sharded counters** - one `AtomicU64` per core, summed on read, so the hot increment path never contends. (The thread-local metrics aggregation in Part 8 is the same idea taken one step further.)
- **Per-writer log segments** - write-ahead logs that give each writer its own segment and merge at commit time.

In every case the move is identical: take a shared mutable thing that *N* agents fight over, and replace it with *N* private things plus a cheap reconciliation step on the read side.

## Why this comes first

It's tempting to open a performance series with the satisfying micro-techniques - cache-line padding, the `Relaxed` versus `Acquire` decision, the zero-copy reservation. But those are refinements *within* a structure, and refining a structure that's architecturally contended is polishing a part that shouldn't exist. The padding in Part 1 matters *because* there's no longer a shared cursor competing for the same cache lines; the minimum-sufficient ordering in Part 2 is tractable *because* each ring has exactly one writer and one reader.

So this is the decision that makes the rest of the series worth doing. Get it right and every later technique compounds on a clean foundation. Get it wrong - keep the shared cursor - and no amount of cache-line tuning will save you from the coherence traffic you designed in.

From here on, the running example is a single one of those SPSC rings, examined up close. Part 1 starts where it should: how to lay that ring out in memory so two cores never fight over a cache line.
