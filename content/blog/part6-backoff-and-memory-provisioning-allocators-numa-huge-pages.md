+++
title = "Backoff & Memory Provisioning: Adaptive Spinning, Custom Allocators, NUMA & Huge Pages"
date = 2026-08-03
description = "Two resources a lock-free ring buffer takes for granted: time and memory. Adaptive backoff for waiting well, and a pluggable allocator trait for NUMA placement and huge pages."
template = "page.html"

[taxonomies]
tags = [
  "rust",
  "unsafe-rust",
  "backoff",
  "spin-loop",
  "allocators",
  "numa",
  "huge-pages",
  "lock-free",
  "spsc-queue",
  "ring-buffer",
  "performance"
]

[extra]
diagram = true
+++

# Backoff & Memory Provisioning: Adaptive Spinning, Custom Allocators, NUMA & Huge Pages

Part 6 of **Low-Level Systems Design in Rust** - a series on writing high-throughput, low-latency systems code, using a single-producer / single-consumer (SPSC) ring buffer as the running example. 

[Part 1](https://debasishg.github.io/blog/part1-cache-conscious-data-layout-in-rust/) decided **where** the shared cursors of a concurrent structure live in memory.

[Part 2](https://debasishg.github.io/blog/part2-cross-core-contract-memory-ordering-and-single-writer-state/) covered **how** two cores read and write them correctly and at minimum cost.

[Part 3](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/) made those reads and writes *rare*. 

[Part 4](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/) talked about the **shape** of the operation itself - moving data without copying it, and structuring each call so the common case is a branch the CPU can predict and pipeline. 

[Part 5](https://debasishg.github.io/blog/part5-compile-time-leverage-specialization-hygiene-inlining/) pushed the remaining decisions to **compile time** - specializing on what the compiler already knows about a type, and inlining so the abstractions cost nothing once the code is built. 

The above 5 parts tuned the *steady state* - data flowing, the consumer keeping up, the buffer simply there. This post handles the two resources that steady state quietly assumed: **time** (what a producer does when the ring is full and it has to wait) and **memory** (where the backing buffer comes from, and where on the machine it lives). 

The running example is still that ring buffer, but none of the three levers below is specific to ring buffers.

There are code snippets as part of the post. If you want to take a deeper dive into the ring buffer project, take a look at the code, tests and Quint specifications at the [repo](https://github.com/debasishg/ringmpsc-rs/tree/main/crates/ringmpsc).

---

## The two things the fast path took for granted

Every earlier post optimized the case where the operation *succeeds*: a slot is free, the data flows, nobody blocks. But two assumptions were hiding underneath:

1. **That the producer never has to wait.** [Part 4](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/)'s `reserve()` returns `None` when the ring is full - and the loop there had a placeholder comment, *"back off and retry."* What goes in that branch matters: waiting *badly* wastes power and can starve the very consumer you're waiting on.
2. **That the buffer just exists.** It has to be allocated somewhere, and two things about that somewhere matter. On a multi-socket server, *where* it lives changes access latency by 1.5–3×. And the page size it's built from changes how many TLB entries it takes to keep the buffer translated.

Two assumptions, but three levers as axes of variations - because the memory one splits. Spend waiting time well. Control *where the buffer comes from*. And then, separately, control *where on the machine it physically lives*. The second is an abstraction problem, the third a placement problem, and they're worth taking a look one at a time.

---

## Lever 1 - Adaptive backoff: how to wait

When the ring is full, the producer has to wait for the consumer to drain a slot.  The two obvious approaches are both wrong:

- A **pure spin loop** wastes power, starves other threads, and on a single-core (or heavily oversubscribed) system can prevent the contending thread from ever running - so you spin forever waiting for progress you're actively preventing.
- A **naked `thread::yield_now()`** is far too slow when the wait is only a handful of nanoseconds: you pay a full context switch to wait for something that was about to happen anyway.

The answer is to **escalate**: spin briefly, then yield, then give up and let the caller decide.

```rust
pub struct Backoff { step: u32 }

impl Backoff {
    const SPIN_LIMIT: u32 = 6;     // 2^6 = 64 spin iters max
    const YIELD_LIMIT: u32 = 10;   // then give up

    pub fn new() -> Self { Self { step: 0 } }

    /// Light spin with PAUSE hints.
    pub fn spin(&mut self) {
        let spins: usize = 1 << self.step.min(Self::SPIN_LIMIT);
        for _ in 0..spins {
            std::hint::spin_loop();   // emits PAUSE on x86, YIELD on ARM
        }
        if self.step <= Self::SPIN_LIMIT { self.step += 1; }
    }

    /// Heavier backoff: spin, then yield.
    pub fn snooze(&mut self) {
        if self.step <= Self::SPIN_LIMIT {
            self.spin();
        } else {
            std::thread::yield_now();
            if self.step <= Self::YIELD_LIMIT { self.step += 1; }
        }
    }

    /// Have we exhausted our patience? This is the whole point - see below.
    pub fn is_completed(&self) -> bool { self.step > Self::YIELD_LIMIT }

    /// Reset for the next wait cycle.
    pub fn reset(&mut self) { self.step = 0; }
}

impl Default for Backoff {
    fn default() -> Self { Self::new() }
}

// INV-BACKOFF-01: the shift below must not overflow.
const _: () = assert!(
    Backoff::SPIN_LIMIT < usize::BITS,
    "SPIN_LIMIT must be < usize::BITS"
);
```

**Why `let spins: usize` and not just `let spins`?** Because the untyped literal would infer `i32`, and `SPIN_LIMIT` is a knob people reach for. At `SPIN_LIMIT = 31`, `1i32 << 31` is `i32::MIN` - so `0..spins` becomes an *empty range* and `spin()` performs zero iterations. Someone raising the limit to spin harder would get a backoff that stops spinning entirely, with no panic and no warning, visible only as a throughput regression under contention. The annotation removes the sign; the `const` assert catches the one case the annotation can't (`SPIN_LIMIT` at or beyond `usize::BITS`, where the shift itself overflows). Both cost nothing at runtime - this loop is dominated by `PAUSE`, so the counter's width is irrelevant.

It's a small thing, but it's the shape of bug worth designing against: a tuning constant whose failure mode is silence rather than a crash. Pushing the constraint into the type and then into a compile-time assertion is the cheapest way to make sure the next person to touch that number finds out immediately.

`is_completed()` and `reset()` are not for decoration. Without the first, `snooze()` saturates at its final step and keeps calling `yield_now()` forever - the exact naked-yield behaviour rejected two paragraphs ago. Without the second, a caller that successfully acquires after a long wait carries a maxed-out `Backoff` into the next wait and gives up almost immediately.

**`spin()` or `snooze()`?** Both are public and the choice matters. Use `spin()` when you know the other thread is actively making progress and will finish shortly - you don't want to surrender your timeslice to wait a few dozen nanoseconds. Use `snooze()` when the wait might be long or the other thread might itself be descheduled; giving up the CPU is what lets it run. In a ring buffer: `spin()` for a failed `compare_exchange` retry, `snooze()` for a full ring.

`std::hint::spin_loop()` is the important primitive. It compiles down to the architecture's spin-wait hint - `PAUSE` on x86, `YIELD` on ARM. That hint is how you tell the CPU that this loop is a busy-wait, not real work.

Saying so buys two things. First, the core stops pushing the loop through the pipeline at full speed, which frees execution resources for the SMT sibling sharing that core - quite possibly the thread you are waiting on. Second, it makes leaving the loop cheaper. A tight loop that keeps re-reading a shared variable fills the pipeline with speculative loads; when another core finally writes to that variable, the CPU spots the memory-order violation and throws the speculated work away. The hint keeps far fewer loads in flight, so there is much less to throw away. Without it, you are burning full execution throughput on a loop that does nothing.

The three phases have very different cost profiles, which is the whole reason to distinguish them:

| Phase   | Step | Action                              | Cost per call                  |
|---------|------|-------------------------------------|--------------------------------|
| Spin    | 0–6  | `spin_loop()` hint (PAUSE / YIELD)  | (1–45 ns) × `2^step` iters     |
| Yield   | 7–10 | `thread::yield_now()`               | ~1 µs if it actually switches  |
| Give up | >10  | `is_completed()`; caller decides    | (depends on caller)            |

The spin phase **doubles** its iteration count each step - 1, 2, 4, 8, 16, 32, 64 - classic exponential backoff: short waits stay short, and long waits ramp up quickly so the contending thread gets room to make progress before you spend more cycles. Seven `snooze()` calls burn 127 `spin_loop()` iterations in total before the first yield.

**That per-iteration range is not a hedge - it's a portability problem.** `PAUSE` cost varies by an order of magnitude across microarchitectures: roughly 10 cycles on pre-Skylake Intel, but ~140 cycles from Skylake-SP through Ice Lake (Intel lengthened it deliberately to make spin-wait loops cheaper for the sibling thread), back down to ~40 on recent cores. ARM's `YIELD` is frequently a near no-op. So the full 127-iteration ramp costs ~600 ns on an older desktop part and closer to 6 µs on a Skylake-SP server - where it now exceeds the yield it was supposed to be cheaper than. `SPIN_LIMIT = 6` is a reasonable default, not a portable constant. If you are tuning for one machine, measure `PAUSE` there before trusting the shape of this table.

**Why not jump straight to `park()`?** `std::thread::park` is cheaper than it looks - it's a per-thread token over a futex, needs no user-supplied mutex/condvar, and skips the syscall entirely when the token is already available. What it can't avoid is the kernel transition on the *contended* path, where the parked thread must actually be woken and rescheduled. For waits in the tens-to-hundreds of nanoseconds, that wake-up costs more than simply spinning through the wait. For *longer* waits - milliseconds and beyond - spinning wastes power and parking is clearly right.

Which is why the key design decision is: **keep the backoff bounded and let the caller own the parking choice.** `is_completed()` goes true past the yield limit and hands control back, so the caller decides whether to retry, park, or report backpressure upstream. Don't bury a `park()` inside the backoff loop - that steals a policy decision from the one piece of code that has the context to make it.

### Backoff in the reserve loop

That is exactly the shape the `reserve()` retry loop needs. Part 4 left a `// back off and retry` comment in its backpressure branch; `reserve_with_backoff` is the comment cashed in:

```rust
/// Reserve with adaptive backoff. Spins, yields, then gives up.
pub fn reserve_with_backoff(&self, n: usize) -> Option<Reservation<'_, T, A>> {
    let mut backoff = Backoff::new();
    while !backoff.is_completed() {
        if let Some(r) = self.reserve(n) {
            return Some(r);
        }
        if self.is_closed() {
            return None;
        }
        backoff.snooze();
    }
    None
}
```

Three things worth noticing. The loop is bounded by `is_completed()`, so it *always* terminates - returning `None` is backpressure the caller can act on, not a failure. The closed check inside the loop means a shutdown doesn't have to wait out the full ramp. And the zero-cost fast path is preserved: if the first `reserve(n)` succeeds, the `Backoff` is constructed and dropped without a single `spin_loop()` executing.

**When to use it:** any try-fail-retry loop where the expected wait is *microseconds*, not milliseconds - full ring, a failed `compare_exchange` retry, a network send that returned `EWOULDBLOCK`. If the wait is genuinely long, reach for an OS primitive instead. (This is the same shape Crossbeam's `Backoff` implements, and the same exponential-backoff logic network stacks use on retry.)

---

## Lever 2 - Allocator abstraction: where the buffer comes from

The global allocator is perfectly fine for cold paths. But a *hot* data structure often wants something more specific: page-aligned memory, huge pages, NUMA-bound pages, or an arena. The way to offer that without taxing the people who don't need it is to parameterize on an allocator trait with a **zero-sized default** - the same ZST-default trick from [Part 5](https://debasishg.github.io/blog/part5-compile-time-leverage-specialization-hygiene-inlining/), so that *not* opting in costs literally zero bytes and zero indirection.

```rust
/// # Safety
///
/// Implementations must guarantee:
/// 1. `allocate(capacity)` returns a buffer of exactly `capacity` elements.
/// 2. The memory is valid for reads and writes for the buffer's lifetime.
/// 3. The buffer's `Deref`/`DerefMut` targets are contiguous slices.
/// 4. The buffer's `Drop` correctly deallocates the memory.
pub unsafe trait BufferAllocator: Send + Sync {
    type Buffer<T>: Deref<Target = [MaybeUninit<T>]> + DerefMut;
    fn allocate<T>(&self, capacity: usize) -> Self::Buffer<T>;
}

pub struct HeapAllocator;                    // ZST default

pub struct Ring<T, A: BufferAllocator = HeapAllocator> {
    // ... `A` adds 0 bytes when defaulted ...
}

// Send/Sync are claimed on the Ring, not on the buffer:
unsafe impl<T: Send, A: BufferAllocator> Send for Ring<T, A> {}
unsafe impl<T: Send, A: BufferAllocator> Sync for Ring<T, A> {}
```

A few design points are doing real work here.

**The trait is `unsafe` because of clause 1, mostly.** The natural guess is that the `unsafe` is there for thread-safety. It isn't. That question is already settled, in two other places. The `Send + Sync` bound sits on the *allocator* itself. And the cross-thread claim for the ring is made by [Part 2's `unsafe impl Sync for Ring`](https://debasishg.github.io/blog/part2-cross-core-contract-memory-ordering-and-single-writer-state/), which asks for `T: Send` and asks nothing at all of the buffer.

What `unsafe` actually buys is the length guarantee in clause 1. To reach a slot, the ring masks an index against its capacity and writes through a raw pointer. There is no bounds check anywhere in that path. The mask *is* the bounds check, and it only works if the buffer really is `capacity` elements long. Hand back a shorter slice and every one of those writes lands outside the allocation.

Nothing in the type system prevents that. `Deref` can return any slice the implementor likes, and the compiler has no way to compare its length against the number the ring asked for. Clause 3 is unenforceable for the same reason: the ring assumes one contiguous region at a stable address, so a buffer that splits its storage, or returns a different pointer on each `Deref`, breaks the same arithmetic just as quietly.

Those are the obligations whose violation is undefined behaviour, which is why they are the ones named on the trait. The proof obligation lives where it's owned.

That covers what the trait *says*. What it deliberately doesn't say matters too: there is no `Send` bound on `Buffer<T>`.

It looks harmless to add. Buffers do get handed between threads, so why not put it in the trait? The problem is that a bound on an associated type has to hold for *every* `T`, unconditionally. `Box<[MaybeUninit<T>]>` cannot make that promise: it is `Send` only when `T` is. Write `type Buffer<T>: Deref<Target = [MaybeUninit<T>]> + DerefMut + Send` and the `HeapAllocator` impl stops compiling immediately - not later, at some awkward call site, but in the impl block itself, because the compiler cannot prove `Send` for a `T` that carries no bound.

There are two ways to patch that, and neither is worth it. Put `where T: Send` on the associated type and the bound now holds in precisely the cases where `Box` was already `Send`, which buys nothing. Put `T: Send` on the trait instead and every single-threaded user pays for a guarantee they never asked for. Leaving the bound off is the version that actually works: the buffer is as `Send` as whatever it holds, and the cross-thread claim stays where Part 2 made it.

**Deallocation is RAII, not a trait method.** The trait has an `allocate`, but there is no matching `deallocate`. Clause 4 puts that job in `Buffer<T>`'s own `Drop` instead.

The asymmetry is deliberate, and it falls out of the two operations needing different information. Allocating needs the *allocator*: which NUMA node to bind to, what alignment to round up to, whether to ask the kernel for huge pages. That is policy, and it lives in `A`. Freeing needs none of it. It needs facts about the one buffer in hand - its base pointer, and the exact length that was mapped - and those are known only to the buffer itself. `Box` already tracks its own length and frees on drop. The NUMA buffer records the length its `mmap` was rounded up to, and hands that same number back to `munmap`.

That second example is why the distinction is useful. As Lever 3 notes below, `munmap` with an unrounded length fails with `EINVAL` and leaks the mapping. A `deallocate(&self, buf, len)` on the trait would turn that length into something the caller has to store and re-supply correctly. Kept as a private field - written once at creation, read once at destruction - the two can't disagree. `Drop` also runs on every exit path, unwinding from a panic included, so there is no route by which the ring simply forgets to free. And the ring's own `Drop` stays short: drop whatever elements are still initialized, then let the buffer release its own memory.

**The cost is opt-in, and visible in the type.** Users who don't care write `Ring::<u64>::new(config)` and pay nothing - and, as with `SPIN_LIMIT` above, the claim is worth asserting rather than asserting *about*, so a `const` block checks it at compile time:

```rust
const _: () = assert!(
    std::mem::size_of::<HeapAllocator>() == 0,
    "HeapAllocator must be a zero-sized type"
);
```

Users who do care write `Ring::<u64, AlignedAllocator<128>>::new_in(config, AlignedAllocator::<128>)` or supply their own. Typical implementations:

- `HeapAllocator` - the default, a `Box<[MaybeUninit<T>]>`.
- `AlignedAllocator<const ALIGN: usize>` - over-allocates a byte buffer and hands back a slice starting at the aligned offset, keeping the original allocation alive for `Drop`.
- `NumaAllocator` - `mmap` + `mbind` on Linux (Lever 3, below).
- `StdAllocator` - bridges the nightly `std::alloc::Allocator`.

**`allocate` returns a buffer, not a `Result` - on purpose.** This is worth being explicit about, because it looks like it contradicts the degrade-gracefully rule in Lever 3. It doesn't, because two different failures are in play. 
- *Allocation* failure (out of memory, a failed `mmap`) panics, exactly as every other Rust allocation does; a ring that cannot obtain its backing store has nothing sensible to return. 
- *Capability* failure - asking for NUMA placement on a machine that has no NUMA, or huge pages the OS hasn't reserved - is not an error at all, and is handled by silently producing correct memory with a weaker guarantee.

Only one of those two paths can end without a buffer:

{% diagram() %}
flowchart TD
    A["allocate(capacity)"] --> B{"Capability present?<br/>NUMA node, reserved huge pages"}
    B -- yes --> C["Place as asked<br/>mmap + mbind + MAP_HUGETLB"]
    B -- no --> D["Capability failure<br/>use plain pages instead, warn once"]
    C --> E{"Memory obtained?"}
    D --> E
    E -- yes --> F["Buffer of capacity elements<br/>correct either way, only the speed differs"]
    E -- no --> G["Allocation failure<br/>panic, as any Rust allocation does"]
{% end %}

Notice that the capability branch rejoins the happy path. Both routes through it end at the same buffer, equally correct and equally safe to use; what differs is only how fast it will be.

So imagine `allocate` did return a `Result`. What would the error case actually mean? Not "out of memory" - that path panics. It would mean *"here is your memory, it works fine, but it isn't on the node you asked for."* Almost no caller can do anything with that. They cannot conjure a NUMA node into existence, and they are already holding a perfectly usable buffer. The honest response at nearly every call site would be `.unwrap()`, so the `Result` buys nothing and taxes the happy path with error handling that never fires.

Keeping the signature infallible puts the decision in the one place with enough context to make it. The allocator knows what it asked the kernel for, what it got back, and whether the gap is worth a warning. The caller gets a flat guarantee instead - a buffer of `capacity` elements, always safe to use - and the placement it ended up with is a performance property, not a correctness one.

**Why a custom trait rather than `std::alloc::Allocator`?** Because `Allocator` is nightly-only. Keeping the abstraction local means the code works on stable Rust today while leaving room for a nightly bridge - the unsafety is held by the implementor, never the user.

---

## Lever 3 - NUMA placement and huge pages

On a multi-socket server, reaching memory on a *remote* NUMA node costs 1.5–3× the latency of local-node access, and scattering a large structure across physical pages thrashes the TLB. For a big, long-lived backing array - exactly what a ring buffer is - both are worth controlling. `NumaAllocator` is just a `BufferAllocator` that makes those choices.

### Placement

Bind the allocation to a node, by an explicit policy:

```rust
pub enum NumaPolicy {
    Fixed(u16),        // all allocations on one node
    RoundRobin,        // cycle across nodes
    ProducerLocal,     // the current thread's node
}
```

On Linux, `mmap` + `mbind(MPOL_BIND, …)` does the placement.

### Huge pages

`MAP_HUGETLB` requests 2 MiB pages instead of the default 4 KiB:

```rust
let mut flags = libc::MAP_PRIVATE | libc::MAP_ANONYMOUS;
if huge_pages {
    flags |= libc::MAP_HUGETLB;
}
```

The payoff is fewer TLB entries. But it is worth being honest about how much that buys for *this* structure, because the obvious worked example doesn't survive scrutiny: a 64K-slot ring of `u64` is 512 KiB, which needs 128 4 KiB-page translations - and 128 entries fit comfortably in a modern L2 STLB of ~1500+.  There was no TLB problem to solve. Worse, a ring buffer is walked *sequentially*, which is the access pattern hardware prefetchers and the TLB handle best: one miss per 4 KiB page, amortised across 512 slots.

So the honest claim is narrower than "rings want huge pages." Huge pages start paying when the *working set of mappings* is large - a channel with many rings, each buffer in the megabytes, interleaved with the rest of the process's memory, where translations for one ring get evicted before that ring is touched again.  That's a genuinely TLB-bound workload, and there the improvement is real. A single sequentially-drained ring is not one, and adding `MAP_HUGETLB` to it will do nothing you can measure.

Three caveats decide whether any of this helps:

- **Huge pages must be pre-reserved by the OS** (`vm.nr_hugepages`); the `mmap` fails otherwise. This is also the reason to consider transparent huge pages - `madvise(MADV_HUGEPAGE)` on a normal mapping - first: no reservation, no configuration, no failure mode, and the kernel backs the region with huge pages when it can. `MAP_HUGETLB` is what you reach for when you need the guarantee rather than the hint.
- **A huge-page mapping is rounded up to 2 MiB.** That 512 KiB ring now occupies a full 2 MiB - 4× the memory for no benefit at that size. It also means the length you hand to `munmap` must be the *rounded* length; passing the unrounded one fails with `EINVAL` and leaks the mapping.
- **NUMA placement only matters when threads are pinned.** If the scheduler is free to migrate the producer to another node, it defeats your careful local placement. Pair NUMA-aware allocation with `pthread_setaffinity_np` (or equivalent) or don't bother.

One ordering detail makes `mbind` work at all: anonymous `mmap` pages aren't physically allocated until first touch, so binding the range *before* anything writes to it is what lets `MPOL_BIND` decide placement rather than merely record a preference. Allocate, bind, then touch.

### Degrade gracefully - never *require* the optimization

`mbind` and `MAP_HUGETLB` are Linux-specific; macOS, Windows, FreeBSD, and most embedded targets have neither the same APIs nor the same NUMA semantics. The right shape is to gate the platform-specific path behind a `cfg` and fall back to the default heap allocator everywhere else, with a *one-time* warning so the operator knows the hint had no effect:

```rust
#[cfg(target_os = "linux")]
mod platform { /* mmap + mbind path */ }

#[cfg(not(target_os = "linux"))]
mod platform {
    // Falls back to HeapAllocator; warns once via std::sync::Once.
}
```

This keeps the **API** portable - the same `NumaAllocator::new(policy)` call compiles everywhere - and the **performance** honest, with no silent pretending that placement took effect. The principle generalizes to *any* platform-specific optimization: **feature-detect, accept the fallback, warn once, never panic.** Code that needs NUMA or huge pages to *function*, rather than merely to run faster, is almost never what you want - a performance hint that becomes a portability requirement has been mis-designed.

The same provisioning concerns apply to anything with large, long-lived backing arrays: write-ahead log segments, columnar in-memory stores, JIT code caches.

---

## Where this leaves the series

Parts 1–5 made the steady state fast; Part 6 provisioned the two resources that steady state assumed - *time*, by waiting well when the structure is full, and *memory*, by controlling where the buffer comes from and where it physically lives. Notice how much of this reuses machinery from earlier posts: backoff fills in Part 4's backpressure branch, and the allocator trait leans on Part 5's ZST default and Part 2's `Sync` proof. The pieces compose.

Six posts in, we've examined a single structure from nearly every hardware angle.  The remaining posts change the lens. The next one is about a distinctly *Rust* idea that's run quietly underneath everything so far: that the language's own safety machinery - move semantics, the borrow checker, `debug_assert!` - is not a tax on performance but a *tool* for it. That's where Part 7, *Safety as Performance*, goes.

---

## Errata

*Added 2026-08-23.*

**`reserve_with_backoff` takes `&mut self`, not `&self`.** The signature shown above is:

```rust
pub fn reserve_with_backoff(&self, n: usize) -> Option<Reservation<'_, T, A>> {
```

It should be `&mut self`, matching `reserve` as shown in [Part 3](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/) and [Part 4](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/). This is not cosmetic. With `&self`, two calls borrow-check happily, and because the cursor does not advance until `commit()`, both are handed `&mut` slices over the *same* slots - two live `&mut` aliasing the same memory, with no concurrency required to trigger it. The `&mut` receiver is what makes "one outstanding reservation per producer" a compile-time property instead of an unwritten rule.

The backoff argument in this post - spin, then yield, then give up, and the `Backoff` state machine itself - is entirely unaffected. Only the receiver is wrong.

- Filed as [ringmpsc-rs#7](https://github.com/debasishg/ringmpsc-rs/issues/7), fixed in [PR #8](https://github.com/debasishg/ringmpsc-rs/pull/8).
- [Part 7](https://debasishg.github.io/blog/part7-safety-as-performance-moves-borrow-checker-debug-assert/) works through which protocol guarantee each part of the signature actually buys, since `&mut self`, the borrow's mere existence, and `commit(self)` are easily conflated.

**A note on the allocator trait.** Nothing in the `BufferAllocator` design above is wrong, but it is worth recording that the `type Buffer<T>: Deref<Target = [MaybeUninit<T>]> + DerefMut` bound is what made a separate bug possible ([#5](https://github.com/debasishg/ringmpsc-rs/issues/5)): every deref through it yields a slice spanning the entire allocation, and forming that reference is itself an access to every slot. The trait's convenience was the vector. NUMA binding, huge pages, and RAII deallocation are all unaffected - the fix caches a base pointer and stops dereferencing the handle on hot paths.
