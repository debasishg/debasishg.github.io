+++
title = "Cache-Conscious Data Layout in Rust: Field Zoning, False Sharing, and the 128-Byte Rule"
date = 2026-06-29
description = "How to lay out a shared Rust struct by who-touches-what - zoning fields by write owner and frequency, then padding the cross-core ones to avoid false sharing and prefetcher-induced contention."
template = "page.html"

[taxonomies]
tags = ["rust", "concurrency", "lock-free", "cache-coherence", "false-sharing", "data-layout", "memory-alignment", "ring-buffer", "systems-programming", "performance"]
[extra]
+++

# Cache-Conscious Data Layout: Field Zoning, False Sharing, and the 128-Byte Rule

Part 1 of **Low-Level Systems Design in Rust** - a series on writing high-throughput, low-latency systems code, using a single-producer / single-consumer (SPSC) ring buffer as the running example.

[Part 0 - *Architectural Decomposition*](https://debasishg.github.io/blog/part0-architectural-decomposition-remove-contention-by-design/) made the highest-leverage decision (remove contention structurally, so every writer gets its own ring) and holds the full **series index** and guiding principles. 

This post starts the micro-level work: given one such SPSC ring, how should it be laid out in memory? The patterns aren't specific to ring buffers - they apply to any structure more than one core touches.

---

## The thing nobody tells you about multi-threaded structs

When you write a single-threaded data structure, the only layout question that matters is "does it fit in cache." When you write a *multi-threaded* one, a second, relevant and a bit subtle question appears:

> For each field, **which core touches it, and how often?**

Getting this wrong will make you pay in a way that no profiler flags as a single hot line. Two cores writing to *different* fields that happen to share a 64-byte cache line will quietly serialize each other through the hardware coherence protocol - a pathology called **false sharing**. The code looks lock-free. The benchmark says otherwise.

This post is about designing the layout deliberately. It comes in two parts:

1. **Field zoning** - grouping fields by `(write owner, frequency)` so that one core's hot writes don't evict another core's hot working set.
2. **Alignment and padding** - the mechanics that make zoning real: why `#[repr(C)]` is load-bearing, why the magic number is often 128 and not 64, and why *adding* prefetch hints can make things worse.

The running example is a single-producer / single-consumer (SPSC) ring buffer.  One core (the producer) appends, another core (the consumer) drains. They coordinate through two monotonic cursors - `tail` (where the producer writes next) and `head` (where the consumer reads next).

Though the following discussion and the design principles are general enough, you can refer to this [ring buffer](https://github.com/debasishg/ringmpsc-rs/tree/main/crates/ringmpsc) implementation as a reference point that honors these guiding principles.

---

## Part 1 - Field zoning: design based on "who touches what"

The cache line - commonly 64 bytes on x86-64 and many AArch64 cores - is the unit of currency for inter-core communication. Coherence traffic is accounted *per line*, not per byte or per field. So the first design move for any shared structure is to sort its fields into zones by access pattern:

| Zone           | Fields                          | Write owner             | Cross-core access             |
|----------------|---------------------------------|-------------------------|-------------------------------|
| Producer-hot   | `tail`, `cached_head`           | producer                | consumer samples `tail`       |
| Consumer-hot   | `head`, `cached_tail`           | consumer                | producer samples `head`       |
| Cold           | `closed`, `config`, `metrics`   | lifecycle/observability | rare, not hot-path polling    |

"Producer-hot" does **not** mean "producer-only." The producer owns writes to `tail`, but the consumer still reads `tail` when its cached view runs dry. The important distinction is __write ownership__: the producer is the core that keeps making the line exclusive, so the fields near that write must be chosen deliberately.

First, the vocabulary, since the rest of the post leans on it. The **hot path** is the code that runs on (almost) every operation - here, the send and receive loops that each core executes millions of times a second. **Hot fields** are the ones those loops touch, like `tail`, `head`, and the two cursor caches. The **cold path**, by contrast, is everything that runs rarely - construction, shutdown, an occasional metrics read - and **cold fields** are the ones only the cold path touches, like `config` and `metrics`. "Hot" and "cold" are about *frequency of access*, and they're what the zones in the table above are sorted by.

The rule is then simple to state: **place each zone on its own set of cache lines.** A hot field written by one core must not share a line with hot fields another core reads or writes frequently - a producer store to `tail` should not invalidate the line holding the consumer's `head` or `cached_tail`, and vice versa. Cold fields, since nobody contends over them on the hot path, can stay packed together at the back.

Here is a ring buffer that encodes those zones directly in its declaration. The type parameter `A` is just a pluggable allocator for the backing buffer, ignore it if you like - what matters is the *order and grouping* of the fields:

```rust
#[repr(C)]
pub struct Ring<T, A: BufferAllocator = HeapAllocator> {
    // PRODUCER HOT 
    tail:        CacheAligned<AtomicU64>,
    cached_head: CacheAligned<UnsafeCell<u64>>,

    // CONSUMER HOT
    head:        CacheAligned<AtomicU64>,
    cached_tail: CacheAligned<UnsafeCell<u64>>,

    // COLD
    closed:      AtomicBool,
    metrics:     Metrics,
    config:      Config,
    buffer:      UnsafeCell<A::Buffer<T>>,
}
```

A few things are worth reading carefully here, because each one is a deliberate choice rather than an accident of style.

**The two `cached_*` fields live in the hot zones, not the cold one.** A producer that wants to know "is there space?" must compare `tail` against `head`. But `head` is written by the *other* core, so reading it directly is a potential cross-core coherence miss - tens to hundreds of cycles. Instead, the producer keeps `cached_head`: a single-writer, producer-owned snapshot of where the consumer was *last time we looked*. Because a stale view of `head` can only ever *under-report* available space (never over-report it), the cache is always safe to trust on the fast path, and the expensive `Acquire` read of the real `head` fires only when the cache says we're out of room. That snapshot is touched on *every* send, so it belongs in the producer-hot zone - and it is deliberately a plain `UnsafeCell<u64>`, not an atomic, precisely because only one core ever writes it. The consumer has the mirror image: `cached_tail` lets it avoid reading the producer-owned `tail` until the local snapshot is exhausted.

**Cold fields share lines on purpose.** `closed`, `metrics`, and `config` are packed together with no padding between them. That's not laziness - it's the point. The goal of zoning is not to align everything; it's to align *only the things that ping between cores*. Padding a cold field just wastes a cache line you could have spent keeping hot data resident. If a lifecycle flag is polled on every send or receive, it is no longer cold; promote it into its own hot or lifecycle zone instead of hiding it behind the word "shutdown."

### Why `#[repr(C)]` is doing real work

Notice the `#[repr(C)]` on the struct. It is not decorative, and removing it would silently undermine the entire layout strategy.

By default, Rust uses `repr(Rust)`, and the [Rust Reference is explicit][repr] that `repr(Rust)` guarantees only field *alignment*, that fields don't overlap, and that the aggregate is sufficiently aligned. It explicitly does **not** guarantee that fields are laid out in declaration order - the compiler is free to reorder them (typically to minimize padding). For a normal struct that's a feature. For a struct whose correctness-of-performance depends on `tail` and `head` sitting in *different* zones, a reorder could collapse your carefully separated zones back onto shared lines.

`#[repr(C)]` pins fields in declaration order, so the zone layout you wrote is the zone layout you get.

One subtlety that's easy to miss: **`repr(C)` on the outer struct does not recursively freeze the layout of nested `repr(Rust)` fields.** It fixes the order of `Ring`'s own fields, but the internal ordering of, say, `Metrics` or `Config` is still `repr(Rust)` unless those types carry their own `repr(C)`. If a nested struct has its *own* hot/cold split that matters, give it `repr(C)` too.  Here it doesn't matter, because `Metrics` and `Config` are entirely cold.

---

## Part 2 - Alignment and false sharing: making zoning real

Zoning is the *intent*. Alignment is the *mechanism*. Declaring that `tail` and `head` belong to different zones accomplishes nothing unless the bytes actually land on different cache lines. That's the job of `CacheAligned<T>`.

### The one-line helper that pays for itself

```rust
/// Aligns a value to a 128-byte boundary so that cross-core fields don't
/// share a cache line. The 128 (vs. 64) is a target policy: it also leaves
/// room for adjacent-line/spatial-prefetch effects on CPUs where those matter.
#[repr(C, align(128))]
struct CacheAligned<T> {
    value: T,
}

impl<T> CacheAligned<T> {
    const fn new(value: T) -> Self {
        Self { value }
    }
}

impl<T> std::ops::Deref for CacheAligned<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.value
    }
}
```

That's the whole thing - a `#[repr(C, align(128))]` newtype with a `Deref` so it's transparent at the call site (`self.tail.load(...)` just works). No library dependency required. Because `Ring` is `repr(C)`, the compiler lays fields out in declaration order and inserts whatever padding is needed to satisfy each field's alignment. Because a Rust type's size is a multiple of its alignment, each small `CacheAligned<T>` also consumes a full 128-byte slot. No two hot fields can share a line, and - critically - neither can they share a line with the *cold* fields that follow.

### Why 128 bytes, when the cache line is 64?

This is the part that trips people up. If the cache line is 64 bytes, why pad to 128?

Because strict line-level separation can still be too optimistic. Some CPUs have adjacent-line or spatial prefetch behavior: when you touch one 64-byte line, the core may speculatively pull in its neighbor. If producer-hot data sits on the line immediately adjacent to consumer-hot data, the prefetcher can drag the two into each other's caches even though, strictly speaking, they never shared a line. The result is **prefetcher-induced false sharing**: the lines are technically separate, the contention is real, and line-granularity tools may not point directly at it. You see it as throughput that's lower than the algorithm predicts.

For small cursor fields, aligning each hot field to 128 bytes makes it start on an even 64-byte-line boundary, so an adjacent-line prefetch usually pulls in padding rather than another core's hot data.

Treat the exact width as a **target-specific policy, not a universal constant**.
Crossbeam's current [`CachePadded`][cachepadded] policy is:

- **128 bytes** on x86-64, AArch64, and powerpc64,
- **256 bytes** on s390x,
- **32 bytes** on arm, mips, mips64, sparc, and hexagon,
- **16 bytes** on m68k,
- **64 bytes** on everything else.

Crossbeam also documents that these are reasonable guesses, not a guarantee that they match the physical cache line of the machine you're running on. A local `CacheAligned<T>` with a fixed `align(128)` is appropriate when you specifically want to *pin* the policy (e.g. you've benchmarked your target and want it frozen).  Either way: **verify with benchmarks**, don't trust any single number - including 128.

### Putting it together: the layout the CPU actually sees

With the zones and the alignment combined, construction is unremarkable - the layout work has already been done by the types:

```rust
Ring {
    tail:        CacheAligned::new(AtomicU64::new(0)),
    cached_head: CacheAligned::new(UnsafeCell::new(0)),
    head:        CacheAligned::new(AtomicU64::new(0)),
    cached_tail: CacheAligned::new(UnsafeCell::new(0)),
    closed:      AtomicBool::new(false),
    // metrics, config, buffer ...
}
```

The producer core writes the `tail` line and uses the `cached_head` line. The consumer core writes the `head` line and uses the `cached_tail` line. The published cursors still create the unavoidable cross-core communication - the consumer must sometimes observe `tail`, and the producer must sometimes observe `head` - but those invalidations no longer drag unrelated hot fields with them.  That is the payoff, and it costs roughly one 128-byte slot per hot field: a good trade for a structure hammered by two cores, and one you'd never make for a cold field.

---

## Two counterintuitive corollaries

Once you're thinking at the level of cache lines and prefetchers, two pieces of "obvious" performance folklore turn out to be wrong.

### Don't sprinkle in software prefetch hints

A common instinct, having learned that the prefetcher exists, is to "help" it: drop `_mm_prefetch` (or Rust's `core::intrinsics::prefetch_*`) into the hot loop to pull the next slots in early. On a ring buffer - i.e. **stride-1, linear access** - this is almost always a *loss*.

Modern L1/L2 hardware prefetchers detect linear strides within the first 2–3 accesses and stream cache lines in ahead of the CPU automatically. A manual software prefetch doesn't add to that; it *competes* with it - consuming instruction issue slots and memory bandwidth that the demand fetches actually need, while the hardware was already going to bring those lines in. It's a mistake I've watched ship more than once: a hot loop seeded with manual prefetch hints, A/B benchmarked, and found to *hurt* throughput - the hints come back out.

The honest rule: software prefetches earn their keep only for **non-linear** access patterns - graph walks, hash-table probe sequences, pointer chasing - where the hardware prefetcher *can't* predict the next address. And even then, only add them if a profile-guided experiment proves they help on your workload. For anything stride-1, get out of the prefetcher's way.

### Pick a capacity that fits a cache level - deliberately

The ring's *buffer* has its own cache story, separate from the cursors. Capacity doesn't guarantee residency in a particular cache level, but it does set the working-set budget you are asking the hierarchy to carry. Choose it consciously rather than typing a round number and letting the hardware discover the trade:

- **Sized to stay near L1.** For 8-byte elements, 4,096 slots is 32 KiB. That can be a reasonable starting point for a *cache-resident* ring on cores with a 32 KiB-or-larger L1D, but it leaves no room for other hot data on a 32 KiB L1.  L1D sizes vary, and fitting capacity-wise does **not** guarantee zero L2/L3 misses - conflict misses, prefetch behavior, coherence traffic, and other hot data on the same core all still matter. Confirm with hardware counters, not arithmetic.
- **Sized to absorb bursts.** Sizing at 256K slots (2 MiB for 8-byte elements) deliberately stops pretending the buffer is L1-resident and accepts lower-cache or LLC latency in exchange for soaking up much larger bursts without backpressure.

Both are legitimate. What isn't legitimate is letting "a nice round number like one million" sneak into your config and *accidentally* deciding the working set your hot path has to carry.

---

## Where this generalizes

None of this is specific to ring buffers. The "zone by `(write owner, frequency)`, then pad the cross-core fields" pattern shows up wherever a structure is shared across cores:

- the `tail`/`head` cursors of **SPSC** queues, and the contended cursors of **MPSC** queues (padding prevents false sharing there; it does not remove the shared-tail contention),
- per-thread counters in **sharded metrics** (an `AtomicU64` per core, summed on read),
- per-core **epoch counters** in EBR / hazard-pointer memory reclamation,
- per-CPU **scheduler run queues**,
- resize-epoch counters in **lock-free hash tables**.

In every case the recipe is the same:

1. For each field, name the write owner, the readers, and the frequency.
2. Group fields into hot zones (one per write owner, plus isolated zones for any unavoidable multi-writer atomics) and a cold zone.
3. Lock the order with `#[repr(C)]`.
4. Pad the hot, cross-core fields to a target-appropriate cache boundary - 128 bytes on x86-64/AArch64/powerpc64 in Crossbeam's policy, or let `CachePadded` pick.
5. Leave the cold fields packed, and *don't* add prefetch hints to linear loops.
6. Verify the win with hardware counters such as `perf c2c` where available, and with a throughput benchmark - because the bugs this prevents are exactly the ones a profiler won't point at for you.

The structure looks lock-free either way. The difference between the two versions is whether the hardware agrees.

[repr]: https://doc.rust-lang.org/reference/type-layout.html#the-rust-representation
[cachepadded]: https://docs.rs/crossbeam-utils/latest/crossbeam_utils/struct.CachePadded.html
