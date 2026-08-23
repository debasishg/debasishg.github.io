+++
title = "Amortizing Cross-Core Coordination: Cursor Caching and Batch Processing"
date = 2026-07-13
description = "How cursor caching and batch processing amortize the cost of cross-core coordination in a lock-free SPSC ring buffer, so cores touch shared atomics a fraction as often."
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
  "cursor-caching",
  "batching",
  "cache-coherence",
  "performance"
]
[extra]
diagram = true
+++

# Amortizing Cross-Core Coordination: Cursor Caching and Batch Processing

Part 3 of **Low-Level Systems Design in Rust** - a series on writing high-throughput, low-latency systems code, using a single-producer / single-consumer (SPSC) ring buffer as the running example. 

[Part 1](https://debasishg.github.io/blog/part1-cache-conscious-data-layout-in-rust/) decided **where** the shared cursors of a concurrent structure live in memory.

[Part 2](https://debasishg.github.io/blog/part2-cross-core-contract-memory-ordering-and-single-writer-state/) covered **how** two cores read and write them correctly and at minimum cost.

This post is about doing it **less often** - because the cheapest cross-core synchronization is the one you don't perform. The running example is still a single-producer / single-consumer (SPSC) ring buffer, but the techniques apply to any structure where cores coordinate through shared atomics.

There are code snippets as part of the post. If you want to take a deeper dive into the ring buffer project, take a look at the code, tests and Quint specifications at the [repo](https://github.com/debasishg/ringmpsc-rs/tree/main/crates/ringmpsc).

---

## The expensive operation, and two ways to spend less on it

Part 2 established the unit of cost in lock-free coordination: a cross-core atomic. Just as a refresher in the SPSC ring buffer we have two cursors living in shared memory:
  - `tail` - written by the producer, tracks where it will write next.
  - `head` - written by the consumer, tracks where it will read next.

Modern CPUs don't read/write main memory (RAM) directly on the hot path - they operate on cache lines (typically 64 bytes) held in each core's private L1/L2 cache. When the consumer does `head.store(...)`, that store lands in the consumer core's cache, and the cache line holding the `head` becomes owned exclusively by that core in the Modified (M) state of the MESI coherence protocol.

Now when the producer wants to read the `head` (to check "has the consumer freed any slots?"), the latest value of `head` isn't in the producer's cache, and it isn't even in RAM yet - it's sitting dirty in the consumer core's private cache.

Here's the sequence the hardware runs when the producer issues that `Acquire` load:

  1. The producer's core looks in its own L1/L2 - it's a **miss** (either it doesn't have the line, or it has a stale copy that coherence has already invalidated).
  2. The coherence protocol broadcasts a request for that cache line across the interconnect (on Intel, a ring/mesh bus, across sockets).
  3. The consumer's core, which holds the line in Modified state, has to respond and ship the line over - a cache-to-cache transfer.
  4. The line's coherence state transitions (the consumer's copy typically drops to Shared or Invalid), and the producer finally gets the value.

That's the **coherence miss** .. and the following diagram is an illustration of this ..

{% diagram() %}
sequenceDiagram
    participant P as Producer core
    participant IC as Interconnect
    participant C as Consumer core
    Note over C: holds cache line in Modified (dirty)
    P->>P: Acquire load of head — L1/L2 miss
    P->>IC: request line (read for ownership)
    IC->>C: snoop
    C->>C: line is dirty (Modified)
    C-->>IC: cache-to-cache transfer
    Note over C: line drops M → Shared/Invalid
    IC-->>P: deliver line
    Note over P: producer gets fresh head value
{% end %}

That whole round trip - request out, snoop, cache-to-cache transfer back - is what costs tens to hundreds of cycles:
  - Same core complex / shared L3: ~tens of cycles.
  - Different core, same socket: ~40-80 cycles (~40 ns).
  - Across sockets (NUMA): can be well into the hundreds.

Do that once per item and your throughput is capped at roughly `1 / cross_core_latency` items per second, no matter how fast the rest of your code is.

Note that the `Acquire` ordering isn't itself what causes the coherence miss - the miss is about where the data lives. But `Acquire` is the reason you're reading the real shared cursor at all (to get a correct, synchronized view of the producer's published writes), rather than a stale private copy.

There are exactly two ways to drive that cost down, and this post takes a look at each of them:

1. **Avoid the read** - keep a private, conservative cache of the other core's cursor and consult *that* on the fast path, paying the cross-core load only when the cache says you're stuck. That's Part 1 below and the biggest trick it employs is that you never try to keep `cached_head` fresh. You let it be stale on purpose and rely on the fact that a stale value is safe.
2. **Amortize the read** - when you must synchronize, do it once for a whole *batch* of items instead of once per item. (Part 2)

Both rest directly on machinery from the earlier posts: the cache is the single-writer `UnsafeCell` Part 2 introduced, and the batch publish is the Release/Acquire protocol Part 2 spent its first half on.

---

## Part 1 - Cursor caching: avoid the cross-core read

Most of the time, a core doesn't actually *need* a fresh view of the other core's cursor. A producer checking "is there room for `n` items?" only needs to know that `head` has advanced *far enough* - and if it had plenty of room last time it looked, it very likely still does. So instead of reading the real `head` every time, the producer keeps a **single-writer cache** of it, owned and written by the producer alone, and reads the real cursor only when the cache says it's out of space. See the next section for why it's perfectly safe and correct to read from the stale cache.

That cache is the `cached_head` field from Parts 1 and 2 - a bare `UnsafeCell<u64>` rather than an atomic, precisely because exactly one core ever writes it.

Concretely, this shows up in `reserve` function below - the producer's entry point for claiming space in the ring. A caller asks for room for `n` items; `reserve` returns a `Reservation` it can write into if the ring has space, or `None` if it's full.  The whole point of the method is to answer "is there room for `n`?" as cheaply as possible, and that question is exactly where the cached cursor does the trick: the fast path answers it from the producer's own cache with no cross-core traffic, falling back to the real `Acquire` load only when the cache reports the ring full:

```rust
pub fn reserve(&mut self, n: usize) -> Option<Reservation<'_, T, A>> {
    let tail = self.tail.load(Ordering::Relaxed);   // our own cursor: cheap

    // fast path: consult the producer-owned cache. No cross-core traffic.
    // safety: cached_head is written only by the (unique) producer thread.
    let cached_head = unsafe { *self.cached_head.get() };
    let space = self.capacity()
        .saturating_sub(tail.wrapping_sub(cached_head) as usize);

    if space >= n {
        return Some(self.make_reservation(tail, n));
    }

    // slow path: the cache says we're full. *Now* pay for the cross-core
    // Acquire load, and refresh the cache for next time.
    let head = self.head.load(Ordering::Acquire);   // potential coherence miss
    // safety: same single-writer guarantee.
    unsafe { *self.cached_head.get() = head; }

    let space = self.capacity()
        .saturating_sub(tail.wrapping_sub(head) as usize);
    (space >= n).then(|| self.make_reservation(tail, n))
}
```

The consumer side is the mirror image: it caches `tail` in a `cached_tail` `UnsafeCell` and reads the real `tail` with `Acquire` only when its cache reports the ring empty.

### Why a stale cache is always safe

The subtle part is that the cache can be *wrong* and the code is still correct. The key property: **a stale view of `head` can only ever under-report available space, never over-report it.** `head` moves in exactly one direction (the consumer only ever frees slots), so a cached `head` that lags behind reality makes the producer think it has *less* room than it really does. The worst that happens is an unnecessary trip to the slow path - never a write into a slot that overwrites existing data and that which the consumer hasn't actually released. Conservative caches are safe caches.

The control flow makes the safety visible: a stale cache can only ever divert you *down* into the slow path, and the slow path re-checks against the real `head` before it ever hands out a reservation. There is no edge that turns a stale cache into an over-report.

{% diagram() %}
flowchart TD
    A["reserve(n): load own tail (Relaxed)"] --> B{"fast path: space >= n?<br/>(computed from cached_head)"}
    B -- yes --> C["make_reservation — no cross-core read"]
    B -- "no (cache says full)" --> D["slow path: Acquire load of real head<br/>+ refresh cached_head"]
    D --> E{"real space >= n?"}
    E -- "yes (cache was stale &amp; pessimistic)" --> F["make_reservation"]
    E -- no --> G["genuinely full → None"]
{% end %}

And under steady-state operation the cache is stale *most of the time*, which is exactly when it pays off. If the consumer is keeping up, the producer's cached `head` lags by a bounded amount and the fast path keeps succeeding; the cross-core `Acquire` fires only at the moments the producer genuinely catches up to the consumer and has to re-check. The cache turns a per-call coherence miss into an occasional one.

### Cache asymmetrically - not every cross-core read wants a cache

Here's the part that's easy to overdo. Having seen the win, the instinct is to cache *every* cross-core read. That's wrong, and the reason is instructive.

Consider two different consumer-side operations:

- A **`readable()` poll** - "is there any data?" - is often called in a tight loop (e.g. an async task spinning until woken). It *should* use `cached_tail`: on an empty ring, each poll costs one `UnsafeCell` read, and the cross-core `Acquire` fires only when the cache is stale. The cross-core read happens *per check*, so caching saves a coherence miss on nearly every call.

  Now you might ask "can the `cached_tail` ever give me a wrong answer ?". Never, for the same reason the producer's stale `cached_head` is safe as explained above. The consumer caches `tail` (the producer's cursor). `tail` moves in exactly one direction - the producer only ever advances it as it publishes more data - so a stale `cached_tail` always lags behind the real `tail`:

  > cached_tail  <=  real_tail        (always)

- A **batch consume** (the subject of Part 2 below) reads `tail` *directly* with `Acquire` and skips the cache entirely. Why? Because that single load already determines how many items the *entire batch* can consume. Its cost is O(1) per batch regardless of batch size, so a cache would save ~20 ns once per batch while adding a branch to every call. Pure overhead.

The rule: **cache when the cross-core read happens *per check*; don't when it already happens *per batch*.** If an operation already amortizes synchronization across N items, an extra cache layer is decoration - it adds a branch and a maintenance burden to save a cost that's already negligible. Which is the perfect segue, because amortization is the other half of this post.

---

## Part 2 - Batch amortization: pay the atomic once per N items

Caching avoids the cross-core *read*. Batching attacks the problem from the other direction: when you genuinely have to touch the shared cursors, touch them **once for a whole run of items** instead of once each.

The model here is the LMAX Disruptor's: the consumer reads `tail` once with `Acquire`, processes everything up to it with no atomics in the loop, then writes `head` once with `Release` to publish all of that progress at once.

The interesting part isn't the atomic count - it's doing this *panic-safely*. If the user's per-item handler unwinds midway through the batch, we must still publish the items we already moved out, so that a later consume (or the ring's own destructor) doesn't read or free them a second time. A `Drop` guard handles that:

```rust
struct ConsumeGuard<'a, T, A: BufferAllocator> {
    ring:    &'a Ring<T, A>,
    current: u64,         // advanced after each successful handler call
}

impl<T, A: BufferAllocator> Drop for ConsumeGuard<'_, T, A> {
    fn drop(&mut self) {
        // Publish whatever progress we made - including on a panic unwind.
        self.ring.head.store(self.current, Ordering::Release);
    }
}

pub fn consume_batch<F>(&self, mut handler: F) -> usize
where
    F: FnMut(T),
{
    let head = self.head.load(Ordering::Relaxed);   // our own cursor: cheap
    let tail = self.tail.load(Ordering::Acquire);   // ONE cross-core load
    let avail = tail.wrapping_sub(head) as usize;
    if avail == 0 {
        return 0;
    }

    let mut guard = ConsumeGuard { ring: self, current: head };

    while guard.current != tail {                   // NO atomics in this loop
        let idx = (guard.current as usize) & self.mask();
        // SAFETY: head <= idx < tail, and the producer published this slot
        // via the Acquire load above, so it is initialized.
        let item = unsafe {
            let buf = &*self.buffer.get();
            buf[idx].assume_init_read()
        };
        // Advance the cursor *before* invoking the handler - see below.
        guard.current = guard.current.wrapping_add(1);
        handler(item);                              // by value; may panic
    }

    // Normal exit: the guard's Drop publishes the final `head` via Release.
    avail
}
```

The whole-batch path now touches atomics **three times total** - a local `Relaxed` load of `head` (hot in our own cache), one cross-core `Acquire` load of `tail` (the only potentially expensive one), and one `Release` store of `head` on the way out (local to the consumer's cache line) - instead of three times *per item*. For a batch of any meaningful size the per-item synchronization cost vanishes into the noise. In one benchmarked implementation, batched consumption brought the amortized per-item cost down to ~2 ns, versus ~6 ns when every item paid its own atomic toll; your exact numbers will depend on item size, batch size, and target.

### Why the cursor advances *before* the handler runs

This ordering looks backwards on first read, and getting it wrong is a real memory-safety bug, not a style nit.

`assume_init_read()` **moves** the value out of the slot - afterward the slot is logically uninitialized. Suppose you advanced the cursor *after* the handler instead, and the handler panicked. The slot now holds a moved-out value, but `head` still points *at* it. When the guard's `Drop` publishes `head` - or when the ring is later dropped - that slot gets `assume_init_read` called on it *again*: a use-after-move, and for non-`Copy` types a double-drop.

Advancing *before* the handler runs keeps the published `head` honest: it always means "every slot below here has already left our ownership." Whatever the handler does, the guard publishes a cursor that never points at a slot we already emptied.

### The idle-poll case is cheap by construction

A consumer polling an empty ring in a tight loop is the common case, and this shape handles it for nearly free: when `avail == 0` the function returns before constructing the guard, so *no* `Release` store is issued - the call costs exactly two atomic loads (near zero on a cache hit, ~40 ns on a cross-core miss).  An idle consumer burns minimal CPU without any special-casing.

### Don't let observability sneak back into the loop

One easy way to throw the entire optimization away: increment a metric counter inside the per-item loop. A `fetch_add` is ~10 ns on x86 (`lock xadd`). Call it 1,000 times per batch and you've spent 10 µs per batch re-introducing exactly the per-item atomic cost you just eliminated. Hoist counters out and update them once per batch:

```rust
if self.config.enable_metrics {
    self.metrics.add_messages_received(count as u64);   // ONE atomic
    self.metrics.add_batches_received(1);
}
```

The principle is general: **anything you do per item - synchronization, metrics, logging - must be cheap, or it has to move out of the loop.** Batching the data path while leaving a `lock xadd` in the loop body is a self-inflicted wound.

### The trade-off, and how to bound it

Batching trades a little latency for a lot of throughput: the *first* item in a batch can't be delivered until the batch has formed. Where tail latency matters more than peak throughput, expose a bounded variant - `consume_up_to(max, handler)` - that caps how many items a single call will accumulate, so no item waits behind an unboundedly large batch.

The same amortize-the-expensive-operation move shows up all over systems code:

- **Disk I/O:** group commit in write-ahead logs - one `fsync` per batch of records instead of one per record.
- **Network I/O:** TCP coalescing; `io_uring` submission queues that batch many operations behind a single tail update.
- **Allocators:** returning N freed blocks to a central free-list in one operation instead of N.

---

## The two halves are the same instinct

Caching and batching look like different techniques, but they're the same idea aimed at the same cost. The cross-core atomic is expensive, so: **avoid it when a conservative local snapshot will do, and amortize it across many items when you can't avoid it.** Caching removes the read on the fast path; batching removes it from the per-item path. Used together, a busy producer/consumer pair touches shared memory a tiny fraction as often as a naive one-atomic-per-item design - which is what turns the careful layout of Part 1 and the correct ordering of Part 2 into actual throughput.

There's a third move hiding in the `reserve()` example above: the producer wrote *directly* into the ring's slots via the reservation, with no intermediate copy.  That zero-copy `reserve`/`commit` protocol - and the fast-path/slow-path discipline that the caching example is one instance of - is where Part 4 goes.

---

## Errata

*Added 2026-08-23.*

**The buffer access in the batch consume loop is unsound.** The loop above reads each slot like this:

```rust
let item = unsafe {
    let buf = &*self.buffer.get();   // WRONG
    buf[idx].assume_init_read()
};
```

The backing store derefs to `[MaybeUninit<T>]` spanning the **whole** allocation, so `buf` is a reference covering every slot in the ring - including the ones the producer is writing right now. Narrowing to `buf[idx]` on the next line is too late: under Rust's aliasing model, *creating* a reference counts as an access to everything it covers, so merely forming `buf` races with the producer's write. The batching argument in this post is unaffected, and so is the range invariant the SAFETY comment appeals to - the slots each side touches really are disjoint. The reference simply claimed more than the protocol granted.

Miri reports it as a data race under both Stacked and Tree Borrows. The fix is to never form the wide reference: capture a base pointer once at construction and do raw-pointer arithmetic on the hot path, so nothing is retagged beyond the single slot being touched.

```rust
let item = unsafe { self.slot(idx).read().assume_init() };
```

- Filed as [ringmpsc-rs#5](https://github.com/debasishg/ringmpsc-rs/issues/5), fixed in [PR #6](https://github.com/debasishg/ringmpsc-rs/pull/6).
- [Part 7](https://debasishg.github.io/blog/part7-safety-as-performance-moves-borrow-checker-debug-assert/) discusses this at length under *"A reference is wider than you think"*, including why an `unsafe` block can be audited carefully for bounds and still be wrong about a reference's extent.

Everything else in this post stands, including the `reserve(&mut self)` signature - see the erratum on [Part 6](https://debasishg.github.io/blog/part6-backoff-and-memory-provisioning-allocators-numa-huge-pages/), where the same method was shown taking `&self`.
