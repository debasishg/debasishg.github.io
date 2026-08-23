+++
title = "The Cross-Core Contract: Memory Ordering and Single-Writer State in Lock-Free Rust"
date = 2026-07-06
description = "How Release/Acquire ordering, Relaxed counters, and single-writer UnsafeCell state make lock-free Rust ring buffers correct and efficient."
template = "page.html"

[taxonomies]
tags = [
  "rust",
  "unsafe-rust",
  "concurrency",
  "atomics",
  "memory-ordering",
  "release-acquire",
  "lock-free",
  "spsc-queue",
  "ring-buffer",
  "unsafe-cell",
  "single-writer",
  "performance"
]
[extra]
+++

# The Cross-Core Contract: Memory Ordering and Single-Writer State in Lock-Free Rust

Part 2 of **Low-Level Systems Design in Rust** - a series on writing high-throughput, low-latency systems code, using a single-producer / single-consumer (SPSC) ring buffer as the running example. 

Part 1, [*Cache-Conscious Data Layout*](https://debasishg.github.io/blog/part1-cache-conscious-data-layout-in-rust/), decided where the shared fields of a concurrent structure live in memory.

This one covers how two cores read and write those fields - correctly, and as cheaply as the hardware allows. As before, the running example is a single-producer / single-consumer (SPSC) ring buffer, but the ideas apply to any lock-free queue, counter, or reclamation scheme.

---

## Recap, and the question this post answers

In the first post we set the foundation with a ring buffer so that the two cursors, `tail` (where the producer writes next) and `head` (where the consumer reads next), sit on separate cache lines. This removes false sharing between unrelated hot fields: producer-owned updates no longer invalidate the consumer’s private cursor state, and consumer-owned updates no longer invalidate the producer’s private cursor state. The published cursors are still shared synchronization points; they are simply isolated from unrelated traffic.

```rust
#[repr(C)]
pub struct Ring<T, A: BufferAllocator = HeapAllocator> {
    // producer hot 
    tail:        CacheAligned<AtomicU64>,
    cached_head: CacheAligned<UnsafeCell<u64>>,
    // consumer hot 
    head:        CacheAligned<AtomicU64>,
    cached_tail: CacheAligned<UnsafeCell<u64>>,
    // cold
    // closed, metrics, config, buffer ...
}
```

With the above solution we solved "placement", a static problem. In this post we talk about the **dynamic problem**, which deals with the runtime. When the producer publishes a new `tail`, what guarantees that the
consumer sees the data the producer wrote before it? 

One solution could be to make the fields `AtomicU64`. But it has a cost that we can try to avoid. Can we make it cheaper ?

Note that two of the four hot fields above are `AtomicU64` and two are a bare `UnsafeCell<u64>` - that asymmetry is the subject of Part 2.

We explore 2 ideas to answer the question, and both of them distill down to the same underlying theme:

1. Minimum-sufficient memory ordering - every atomic operation is a contract with the hardware, pick the cheapest contract that's still correct.
2. Single-writer invariants - when a field has exactly one writer, the cheapest correct contract is no atomic at all.

---

## Part 1 - Memory ordering: the minimum-sufficient rule

The most common mistake with atomics is beig ultra defensive (or pessimistic) and treating the ordering parameter as a safety dial - "when in doubt, turn it up to `SeqCst`." This can give the correct emergent behavior of your code, but along with correctness, it imposes a "tax" as well, which you may not need to pay. 

Treat memory ordering as a contract with the hardware about which reorderings are forbidden.

Stronger orderings emit more fences and slower instructions on weakly-ordered ISAs (ARM, RISC-V), and forbid more compiler reordering on *all* of them. 

One general rule of thumb that you can follow is to use the *minimum* ordering that establishes the happens-before relationship your algorithm actually requires.

| Pattern                                                    | Correct ordering        |
|------------------------------------------------------------|-------------------------|
| Statistical counters (no control flow depends on value)    | `Relaxed`               |
| A same-thread atomic used like a regular variable          | `Relaxed`               |
| Publish data to another thread                             | `Release` on the store  |
| Read data published by another thread                      | `Acquire` on the load   |
| Mutually exclusive global ordering across all threads      | `SeqCst` (rare)         |

### Publication: the one pattern that carries the whole protocol

The producer writes its data into the buffer, then publishes the new `tail` with `Release`. The consumer observes `tail` with `Acquire`:

```rust
// Producer - after writing buffer slots:
self.tail.store(new_tail, Ordering::Release);   // publish

// Consumer:
let tail = self.tail.load(Ordering::Acquire);   // observe
```

The above pattern enforces the exact guarantee that everything the producer wrote **before** the `Release` store becomes visible to the consumer **after** the consumer performs an `Acquire` load that observes that published value. We need no global ordering, and hence no `SeqCst`.

Now you can draw the chain explicitly - the correctness of any Release/Acquire protocol is an audit you can do by pointing at four edges:

```
producer.write(buffer[idx])
    | sequenced-before
    v
producer.tail.store(new_tail, Release)
    | synchronizes-with
    v
consumer.tail.load(Acquire)  - observes new_tail
    | sequenced-before
    v
consumer.read(buffer[idx])   - sees the producer's write
```

The `Release` store is the **publication edge**: it's the single point that makes everything sequenced before it on the producer become visible to any thread that observes the new value via an `Acquire` load. 

"Publication edge" is a *semantic* role, not a claim about the emitted instruction - on some targets it compiles to an ordinary store (more on that below).

### Statistics: `Relaxed`, but what "relaxed" really buys us

A metrics counter doesn't gate any control flow - nobody branches on the exact running total - so it preserves atomicity for the counter itself, but establishes no happens-before relationship for surrounding memory operations.

```rust
self.messages_sent.fetch_add(n, Ordering::Relaxed);
```

`Relaxed` drops the inter-thread ordering constraints: neither the compiler nor the hardware has to establish a happens-before relationship with surrounding operations on other locations. What `Relaxed` does NOT drop is the atomicity of the read-modify-write itself. On x86 that `fetch_add` is still a `lock xadd` (~10 ns); on AArch64 it's an `ldxr`/`stxr` retry loop. 

`Relaxed` is not a free local-variable increment. If per-call atomic cost shows up on a genuinely hot path, the fix isn't a weaker ordering (there isn't one) - it's to stop touching the shared atomic every call: accumulate in a thread-local and flush periodically.

### The cost is target-dependent - and that's the whole point

Here's the subtlety that makes minimum-sufficient ordering matter in practice rather than in theory: the same source compiles to very different costs depending on the CPU's memory model.

**On x86-64, plain acquire loads and release stores are usually nearly free.** The x86-TSO model already provides the ordering those operations need, so `Release` stores and `Acquire` loads typically lower to ordinary `mov` instructions - the ordering annotations exist mostly to constrain the compiler. Do not over-generalize this to "atomics are free on x86":

- atomic read-modify-writes (`fetch_add`, `compare_exchange`) still lower to `lock`-prefixed instructions regardless of ordering
- `SeqCst` stores lower to `xchg`, or `mov` + `mfence`; - target-specific lowering can change.

**On AArch64, ARMv7, and RISC-V the cost is real and visible.** These are weakly ordered: the CPU reorders loads and stores freely unless told not to. `Release` typically selects `stlr` (store-release) on AArch64 or `dmb ish` + `str` on ARMv7; `Acquire` selects `ldar` or `ldr` + `dmb ish`. Each fence costs tens of cycles. `SeqCst` is worse - a full barrier on every operation.

This is why the rule isn't a micro-optimization. Using `Relaxed` where it suffices and Release/Acquire only at publication points keeps the fence count to the minimum the algorithm requires. That's what makes the difference between source that's fast on both an Intel server and a Graviton or Apple-silicon core, and source that's correct but stalls the pipeline on half your fleet. On a weakly ordered machine, minimum-sufficient ordering is the line between an optimally working protocol and a stalled one.

### "Once per lifetime" is not a reason to reach for `SeqCst`

In many related scenarios I have seen developers reaching out to a tempting rationalization: "this atomic is only touched once, at startup, so `SeqCst` is fine - who cares about the cost?" That reasons from the wrong premise. The cost isn't the point; **correctness intent** is (sometimes we call this "convenience over correctness"). Pick the ordering the algorithm actually needs. If registering a new participant just needs a unique slot index followed by a separate `Release` publication, then `Relaxed` or `AcqRel` is the correct ordering, and `SeqCst` is noise that misleads the next reader into thinking a global total order matters here.

Reach for `SeqCst` only when the protocol genuinely requires a single global total order across unrelated atomics - and when it does, document which total order and why. "Two threads must observe these two unrelated events in the same order" is a justification. "It's only called once" is not.

### Documenting the protocol clearly can be a good idea

Release/Acquire discipline is fragile precisely because the ordering arguments are non-local - the correctness of a store here depends on a load over there. So write the protocol down once, next to the type definition, as a numbered sequence:

```text
Producer (write path):                 Consumer (read path):
1. Load  tail  Relaxed                 1. Load  head  Relaxed
2. Read  cached_head (UnsafeCell)      2. Read  cached_tail (UnsafeCell)
3. If short: Load head  Acquire        3. If short: Load tail  Acquire
4. Write buffer slots                  4. Read  buffer slots
5. Store tail  Release  (publish)      5. Store head  Release  (publish)
```

Now anyone editing step 4 can see that step 5 is the publication that makes their write visible, and that step 3 is the matching acquire on the other side. The two columns are mirror images - which is exactly what you'd expect of a symmetric producer/consumer protocol, and a good sign you got it right.

Finally: pair every non-trivial ordering decision with a model-checked concurrency test (e.g. a Loom test that exhaustively explores interleavings).  Ordering bugs don't reproduce under casual testing - they need a tool that enumerates the reorderings the hardware is allowed to perform.

---

## Part 2 - Single-writer invariants and `UnsafeCell`

Part 1 was about choosing the cheapest possible atomic. Part 2 is about noticing when you don't need an atomic at all.

Even `Relaxed` atomics aren't free in the way people assume. Beyond the instruction cost, an `Atomic*` access is opaque to the optimizer: the compiler may not reuse a loaded value across an opaque function call, nor merge adjacent loads of the same atomic, nor hoist one out of a loop. Those are real optimizations you forfeit. When a memory location has exactly one writer, you can reclaim all of them.

**The pattern:** if you can prove - and document - that a location is written by exactly one thread, store it in `UnsafeCell<T>` instead of `Atomic<T>`. Reads from the writer thread are plain reads; any reads from other threads need a separate synchronization channel (in the ring, that channel is the Release/Acquire protocol on the cursors from Part 1).

Recall the two cursor caches from the struct at the top. Each core keeps a private snapshot of the other core's cursor, to avoid a cross-core read on the fast path. The producer's snapshot of `head` has exactly one writer - the producer - so it isn't an atomic:

```rust
cached_head: CacheAligned<UnsafeCell<u64>>,
```

```rust
// SAFETY: cached_head is only ever written by the (single) producer thread.
let cached_head = unsafe { *self.cached_head.get() };
```

The matching write is just as plain - a dereference and a store, no fences, no `compare_exchange`, no `fetch_*`:

```rust
// SAFETY: same single-writer guarantee - only the producer writes here.
unsafe { *self.cached_head.get() = head; }
```

**Why this is worth the `unsafe`:** the compiler can now treat `cached_head` like an ordinary variable - keep it in a register across function calls, hoist it out of the reservation loop, fold it into the surrounding capacity arithmetic. An atomic would have barred every one of those.

### The discipline: enforce single-writer outside the type

The soundness of all this rests on one claim - *exactly one thread ever writes this field* - and that claim cannot be enforced by the `UnsafeCell` itself. It has to be guaranteed by the surrounding API. The cleanest way is to make the single-writer property *structural* rather than a rule humans must remember:

> Give out a non-`Clone` producer handle. If the `Producer` type doesn't implement `Clone`, it can't be duplicated and handed to a second thread, so only one thread can ever hold the right to call `reserve()` - and therefore only one thread can ever write `cached_head`.

The type system now carries the invariant. There's no comment to forget and no runtime check to pay for; constructing a second writer simply doesn't compile.

### You must write `unsafe impl Sync` by hand - and justify it

A type containing `UnsafeCell<T>` is deliberately not `Sync`. The standard library makes `UnsafeCell` opt out of `Sync` because unsynchronized interior mutability is the entire reason `UnsafeCell` exists. But sharing the ring across threads is the whole point of the structure, so you have to supply the impl yourself - and the value of doing it by hand is that it forces you to write down the proof:

```rust
// SAFETY:
// - `tail` and `cached_head` are written only by the producer thread.
// - `head` and `cached_tail` are written only by the consumer thread.
// - Cross-thread visibility of buffer slots is established only via the
//   Release store on `tail` (producer) and the Acquire load on `tail`
//   (consumer); the symmetric protocol applies to `head`.
// - Buffer-slot ownership is transferred by the [head, tail) range
//   invariant: outside that range, slots are exclusively the producer's
//   to write; inside it, exclusively the consumer's to read.
// - The backing storage is valid for cross-thread access and deallocation
//   under this producer/consumer protocol.
// - `T` may move across threads only when `T: Send`.
unsafe impl<T: Send, A: BufferAllocator + Sync> Sync for Ring<T, A> {}
unsafe impl<T: Send, A: BufferAllocator + Send> Send for Ring<T, A> {}
```

A few things about this block are important for reasoning:

- **The `T: Send` bound is mandatory.** If `T` can't cross threads, neither can a ring of `T`. Dropping the bound would let you transport a `!Send` type across a thread boundary inside the ring.
- **Forgetting the impls is a compile error.** Forgetting the justification is a correctness bomb that compiles fine. Treat the `SAFETY:` comment as part of the code, not commentary on it.
- **Push proof obligations down to where they're owned.** The clause about backing-storage validity really belongs to the allocator's contract, not to the ring. Stating it as an allocator-trait guarantee lets this impl rely on it instead of re-deriving allocator semantics inline.

And the general discipline that ties Parts 1 and 2 together: **every `unsafe` block gets a `// SAFETY:` comment naming the specific invariant it relies on.** Better still, give those invariants stable names (e.g. an `INV-…` identifier in a spec) and cite the name in the comment, so the chain from "this code is sound" to "because this property holds" is auditable end to end rather than folklore.

### Where single-writer state shows up beyond ring buffers

- per-thread accumulators in scatter-gather pipelines (write locally, reduce once);
- per-thread caches of read-mostly global state, refreshed only on notification;
- exclusive slot access between a `reserve()` and its matching `commit()`.

---

## The two ideas are one idea

Minimum-sufficient ordering and single-writer state are the same principle applied at two levels. Ordering asks: *given that I need an atomic, what's the cheapest correct contract with the hardware?* Single-writer asks the question one level up: *do I need the atomic at all?* In both cases the move is to pay for exactly the synchronization the algorithm requires and not one fence more. Loosely this can correspond to the more general principle of using the least powerful abstraction when it comes to software engineering.

There's a third level to this same principle: sometimes you don't need the cross-core *read* either, because a private, conservative snapshot of the other core's cursor is good enough on the fast path. That snapshot is exactly the single-writer `UnsafeCell` cache introduced here. In Part 3 we will turn it into a throughput win.

---

## Errata

*Added 2026-08-23.*

**`UnsafeCell` is no longer the only reason `Ring` opts out of `Sync`.** The text above says a type containing `UnsafeCell<T>` is deliberately not `Sync`, which is still true. But after the fix in [ringmpsc-rs#5](https://github.com/debasishg/ringmpsc-rs/issues/5) / [PR #6](https://github.com/debasishg/ringmpsc-rs/pull/6), `Ring` also holds a `base: *mut MaybeUninit<T>`, and raw pointers are independently `!Send + !Sync`. Either field alone would force the hand-written impl; the conclusion of this section is unchanged, and if anything the case is now overdetermined.

**The `SAFETY` block needs one more clause.** As written it covers the cursors, the visibility protocol, the `[head, tail)` ownership range, and `T: Send`. It should also record what the new pointer asserts:

```rust
// - `base` is derived once, before the ring is shared, and never
//   re-derived. It points into the allocation owned by `buffer`, which
//   outlives every use of `base`.
```

**A postscript worth more than the correction.** This block contains the clause:

> Buffer-slot ownership is transferred by the `[head, tail)` range invariant: outside that range, slots are exclusively the producer's to write; inside it, exclusively the consumer's to read.

That clause was, and remains, exactly right. Issue #5 was a data race that violated it anyway - not because the protocol was wrong, but because the *code* formed references spanning the whole buffer before narrowing to the slot the protocol granted, and under Rust's aliasing model creating a reference is itself an access to everything it covers. The proof was sound and the implementation claimed more than the proof allowed.

That is the argument for writing these proofs down, and also its limit: a `SAFETY` block is checked by a human against the code's *intent*, never against its actual memory effects. [Part 7](https://debasishg.github.io/blog/part7-safety-as-performance-moves-borrow-checker-debug-assert/) takes this up under *"A reference is wider than you think"*, and Part 8 is about the tooling - Miri, Loom - that checks what the comment cannot.
