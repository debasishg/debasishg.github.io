+++
title = "Zero-Copy on the Hot Path: Reserve/Commit and Fast-Path/Slow-Path Splitting"
date = 2026-07-20
description = "How a reserve/commit protocol eliminates per-item copies in a lock-free SPSC ring buffer, and how fast-path/slow-path splitting keeps the common case branch-predictable."
template = "page.html"

[taxonomies]
tags = [
  "rust",
  "unsafe-rust",
  "zero-copy",
  "maybe-uninit",
  "panic-safety",
  "lock-free",
  "spsc-queue",
  "ring-buffer",
  "branch-prediction",
  "fast-path",
  "performance"
]
[extra]
diagram = true
+++

# Zero-Copy on the Hot Path: Reserve/Commit and Fast-Path/Slow-Path Splitting

Part 4 of **Low-Level Systems Design in Rust** - a series on writing high-throughput, low-latency systems code, using a single-producer / single-consumer (SPSC) ring buffer as the running example. 

[Part 1](https://debasishg.github.io/blog/part1-cache-conscious-data-layout-in-rust/) decided **where** the shared cursors of a concurrent structure live in memory.

[Part 2](https://debasishg.github.io/blog/part2-cross-core-contract-memory-ordering-and-single-writer-state/) covered **how** two cores read and write them correctly and at minimum cost.

[Part 3](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/) made those reads and writes *rare*. 

This post is about the shape of the operation itself - moving data without copying it, and structuring each call so the common case is a branch the CPU can predict and pipeline. The running example is still a single-producer / single-consumer (SPSC) ring buffer, but neither
technique is specific to ring buffers.

There are code snippets as part of the post. If you want to take a deeper dive into the ring buffer project, take a look at the code, tests and Quint specifications at the [repo](https://github.com/debasishg/ringmpsc-rs/tree/main/crates/ringmpsc).

---

## Two taxes that have nothing to do with atomics

Parts 1 through 3 were all about the cost of *coordination* - cache lines, memory ordering, and how often cores have to talk. But a naive hot path pays two more taxes that no amount of clever atomics will fix:

1. **Copying.** In the obvious queue API the producer builds an item somewhere (on its stack), then `push(item)` copies it into a ring slot - that's the *copy-in*. Later the consumer calls `pop()`, which returns the item *by value*, meaning it gets copied back out of the slot into the consumer's local variable - that's the *copy-out*. For anything larger than a machine word that's two `memcpy`s per item the design can avoid. This post presents the reserve/commit protocol that builds the producer side, which eliminates the copy-in. The consumer's copy-out is really the mirror-image protocol - borrow the initialized slots in place, process them, then release.
2. **Mixing the rare case with the common one.** If every call runs the expensive path (synchronize, allocate, syscall) inline with the cheap one, the branch predictor can't learn the common case and the CPU can't pipeline past it.

This post removes both: **zero-copy via reserve/commit**, then **fast-path/slow-path splitting**.

---

## Technique 1 - Zero-copy via reserve/commit

The principle is simple: *the owner of the destination memory should write directly into it.* Instead of building an item on the stack and handing it to the queue to copy, let the producer construct it in place, inside the slot it will live in. That turns a write-then-copy into a single write.

Mechanically, sending splits into two phases:

1. **Reserve** - the queue advances its cursor to claim space and hands back a mutable slice of *uninitialized* slots: `&mut [MaybeUninit<T>]`.
2. **Commit** - the producer signals "I've initialized the first `K` slots; publish them" - and *that* store is the Release publication from Part 2.

Here is the protocol end to end. The thing to watch is the window between reserve and commit: the slots exist and the producer is writing into them, but nothing is visible to the consumer until the Release store closes it.

{% diagram() %}
sequenceDiagram
    participant P as Producer
    participant R as Ring slots
    participant C as Consumer
    P->>R: reserve(n) - claim space
    R-->>P: &amp;mut [MaybeUninit&lt;T&gt;]
    Note over P,R: uninitialized window opens -<br/>slots claimed, invisible to the consumer
    P->>R: write slot 0 in place
    P->>R: write slots 1 .. k-1 in place
    P->>R: commit() - Release store of cursor
    Note over P,R: window closes - exactly k slots published
    C->>R: Acquire load of cursor
    R-->>C: k initialized items visible
{% end %}

The whole game is in getting the safety of that handoff right, and the type system covers only half of it. `MaybeUninit<T>` polices the *read* side: a slot cannot be read as a `T` without an explicit (and `unsafe`) `assume_init`, so you can't stumble into garbage by accident. What it cannot police is the *write* side: a `&mut [MaybeUninit<T>]` has exactly the same type whether you initialized all `n` slots, the first `k`, or none - initialization is a runtime fact the slice type doesn't record. So when commit declares "the first `k` slots are real `T`s now", no compiler check can confirm the claim. Closing that gap is what the API shapes below are about.

### A safe `commit` has to be paid for

Here's the trap. If `commit()` is a safe method that just does `tail += n` - no bookkeeping, no proof - a caller can commit *without* having initialized every slot, and then the consumer calls `assume_init_read` on uninitialized memory, which is undefined behavior. So the safety has to be paid for somewhere in the API shape: either `commit` is `unsafe` and the *caller* carries the proof of initialization, or the reservation *tracks* initialization itself and publishes only what it tracked. There are three sound shapes; pick one and commit to it:

```rust
// Option A - commit is `unsafe`; the caller proves every slot is written.
impl<'a, T, A: BufferAllocator> Reservation<'a, T, A> {
    pub fn as_mut_slice(&mut self) -> &mut [MaybeUninit<T>];
    /// SAFETY: every slot from `as_mut_slice()` was initialized via
    /// `MaybeUninit::write`.
    pub unsafe fn commit(self);
}

// Option B - the caller passes the initialized count; commit is `unsafe`
//            because the API cannot verify that count itself.
impl<'a, T, A: BufferAllocator> Reservation<'a, T, A> {
    /// SAFETY: the first `initialized` slots are fully initialized.
    pub unsafe fn commit_initialized(self, initialized: usize);
}

// Option C - safe builder. The reservation owns the init counter and
//            publishes exactly what it tracked. Zero `unsafe` at the call site.
impl<'a, T, A: BufferAllocator> Reservation<'a, T, A> {
    pub fn push(&mut self, value: T) -> Result<(), T>; // bumps `written`
    pub fn commit(self);                               // safe
}
```

**Option C is the right default for a public API**: the safety proof lives inside the type, once, instead of in every caller. Options A and B justify their `unsafe` only when the caller does *bulk* initialization - a SIMD copy or `copy_from_slice` over the whole slice - where threading a per-item counter would cost more than the discipline saves.

With the safe builder, the call site has no `unsafe` at all:

```rust
if let Some(mut r) = producer.reserve(n) {        // &mut self - see Part 2
    for i in 0..r.capacity() {
        if r.push(make_item(i)).is_err() { break; }
    }
    r.commit();                                   // publishes only what was written
}
```

The unsafe form trades that safety for the ability to write the slots however you
like:

```rust
if let Some(mut r) = producer.reserve(n) {
    let slice = r.as_mut_slice();                 // &mut [MaybeUninit<T>]
    for slot in slice.iter_mut() {
        slot.write(make_item());                  // .write(), NOT assignment through as_mut_ptr()
    }
    // SAFETY: the loop above initialized every slot in the slice.
    unsafe { r.commit(); }
}
```

That `slot.write(value)` is not a stylistic choice. Consider the tempting alternatives: `*slot.as_mut_ptr() = value`, or `*slot.assume_init_mut() = value`. Both are ordinary assignments through a `T`-typed place. And an ordinary assignment *drops the previous value first*. But here there is no previous value. The destructor runs on uninitialized garbage - undefined behavior. For a `String` it blows up loudly. For a type with no drop glue it stays silent - until a refactor gives it some. `MaybeUninit::write` initializes the slot without running any prior drop. (`*slot = MaybeUninit::new(value)` is also fine - `MaybeUninit` itself has no drop glue - but `.write()` says what you mean.) No mistake introduces UB into a reserve/commit loop more often than this one.

### Panic safety: disarm the destructor *before* you publish

Now the subtle bug, the one that survives code review. Suppose `make_item` panics partway through the loop - some slots written, `commit()` not yet reached. Two things must hold:

- the partially-written slots must be cleaned up (dropped in place, or leaked) - but they must **not** be published, because no commit happened and the consumer must still see the old cursor
- and here's the trap: `commit(self)` takes the reservation *by value*, which means Rust **still runs `Drop` on `self` at the end of `commit`'s body**.  If `commit` advances the cursor and *then* `Drop` walks the same slots calling `assume_init_drop`, those slots get dropped while the consumer also owns them - a classic double-drop.

The fix is two lines: zero the tracked count *before* publishing, so the destructor that runs at the end of `commit` finds nothing to do.

```rust
impl<'a, T, A: BufferAllocator> Reservation<'a, T, A> {
    /// Safe commit (Option C): publishes exactly the `written` count.
    pub fn commit(mut self) {
        let written = self.written;
        // Disarm Drop *before* publishing: from here on the slots belong
        // to the consumer and the reservation owns nothing.
        self.written = 0;
        // SAFETY: `written` slots were initialized via `push`.
        unsafe { self.ring.as_ref().publish(written); }
        // `self` is dropped here; Drop sees written == 0 and does nothing.
    }
}

impl<T, A: BufferAllocator> Drop for Reservation<'_, T, A> {
    fn drop(&mut self) {
        // Runs on a panic unwind (uncommitted) AND on the no-op tail of commit().
        for i in 0..self.written {
            // SAFETY: slots [0, written) were initialized via `push` and
            // have not been published.
            unsafe { self.slice[i].assume_init_drop(); }
        }
    }
}
```

A reservation with `written == k` has exactly two legitimate exits, and one forbidden one. The diagram below shows why the ordering inside `commit` is the entire fix - disarm first, and the destructor that inevitably runs has nothing left to do:

{% diagram() %}
flowchart TD
    A["Reservation alive: written = k"] --> B["panic during make_item()"]
    A --> C["commit()"]
    B --> B1["unwind runs Drop:<br/>assume_init_drop on slots [0, k)"]
    B1 --> B2["nothing published - consumer<br/>still sees the old cursor ✓"]
    C --> C1["written = 0 (disarm Drop)"]
    C1 --> C2["publish(k) - Release store"]
    C2 --> C3["Drop at end of commit sees<br/>written == 0 → no-op ✓"]
    C -.->|"mis-ordered commit:<br/>publish first, disarm late"| X["Drop walks the same k slots<br/>the consumer now owns → double-drop ✗"]
{% end %}

This "disarm the destructor before transferring ownership" is the job `ManuallyDrop` and `mem::forget` normally do - but neither fits cleanly here. A type that implements `Drop` can't be partially moved out of, so `mem::forget(self)` would first need `ManuallyDrop` gymnastics to extract `written` and the ring pointer. Zeroing the counter achieves the same disarming in one line. The only way to keep a regression out is a test that panics partway through `make_item` and asserts each value is dropped exactly once.

### The boxed-closure trap - and a 38% throughput win

When you first sketch a `Reservation`, the natural instinct is to give it a closure to call on commit:

```rust
// DON'T - heap allocation + vtable dispatch on every reserve()
pub struct Reservation<'a, T> {
    slice:     &'a mut [MaybeUninit<T>],
    on_commit: Box<dyn FnOnce(usize)>,
}
```

This puts a `Box::new` and a dynamic dispatch on *every single* `reserve()` call - ~5–10 ns of allocation and an indirect jump - which quietly defeats the entire zero-copy story you just built. The right shape stores a non-null pointer back to the parent structure (lifetime-tied) and calls a concrete method:

```rust
use std::marker::PhantomData;
use std::ptr::NonNull;

pub struct Reservation<'a, T, A: BufferAllocator> {
    slice:   &'a mut [MaybeUninit<T>],
    ring:    NonNull<Ring<T, A>>,
    written: usize,
    _borrow: PhantomData<&'a mut Ring<T, A>>,    // documents the logical borrow
}
```

In one implementation, replacing the boxed closure with this pointer form was the single largest throughput improvement in the project's history - about **38%** on the zero-copy benchmark. The lesson generalizes: any time the "trait object" temptation shows up on a hot path, ask whether the lifetime relationship can be encoded so that a plain pointer suffices instead.

Two idioms in that struct are worth calling out:

- **`NonNull<T>` over `*const T`.** The pointer is known non-null (it came from a reference), so `NonNull` is the honest type: it documents the invariant and enables the null-pointer niche optimization in enclosing enums.
- **`PhantomData<&'a mut Ring<T, A>>` even though `slice` already carries `'a`.** It looks redundant, but it documents the *logical* borrow against the ring and survives refactors that might restructure or remove the slice field later. A one-line insurance policy.

### `Send`/`Sync` falls out for free - and that's the design

A reasonable worry on seeing a raw-ish `NonNull<Ring<T, A>>` in a struct: doesn't that risk making `Reservation` accidentally `Send`/`Sync`? The opposite is true.  `NonNull` (like raw pointers) is `!Send + !Sync`, so its *presence* means the compiler never infers those auto traits for `Reservation` at all. To be precise about what protects what: the single-producer invariant is really preserved by the exclusive `&'a mut` borrow of the producer - even a `Send` reservation moved to another thread couldn't race the producer, because the borrow pins it until the reservation dies. `!Send` is defense-in-depth on top of that, keeping the raw pointer from crossing threads even under refactors that loosen the borrow; and the `'a` lifetime forbids the reservation from outliving the ring. No `unsafe impl` needed anywhere. The pattern is: lean on auto-trait inference for safety, and reach for `unsafe impl` only when you have a positive reason to *grant* `Send` or `Sync` (as Part 2 did for the ring itself).

### Reservations can shrink - and slices stay contiguous

One ergonomic wrinkle: at the end of a circular buffer, a request for `n` slots may wrap around. The reservation returns the *contiguous* run up to the buffer boundary, which can be **fewer than `n`**. Callers must loop:

```rust
let mut remaining = n;
while remaining > 0 {
    if let Some(mut r) = producer.reserve(remaining) {
        let got = r.capacity(); // MAY BE < remaining at wrap-around
        for _ in 0..got {
            let _ = r.push(next_item()); // cannot fail: we stay within capacity()
        }
        r.commit();
        remaining -= got;
    } else {
        // backpressure: ring full - back off and retry
    }
}
```

It's tempting to "fix" this by returning *two* slices (a scatter-gather pair) so a single reserve always satisfies `n`. Don't. Every caller would then have to handle the split case, breaking interop with everything that expects contiguous memory - `io::Write`, `memcpy`, SIMD. The contiguous-only rule pushes one bit of complexity (the retry loop) onto the producer and removes a much larger pile from every consumer of the slice. If you want the contract to be self-documenting, name the method for it: `reserve_contiguous(n)` reads as "may be shorter than `n`". (A count-taking `commit_written(k)` is the Option B spelling from above - pair it only with that API shape, not with the safe builder.)

### You already use this pattern: `Vec::spare_capacity_mut`

If reserve/commit feels exotic, it isn't - the standard library exposes the exact same protocol on `Vec`, for the case where you just need a one-shot buffer to feed a syscall, parser, or decompressor. Save the length, fill the spare region, publish with `set_len`:

```rust
let old_len = buf.len();
let spare   = buf.spare_capacity_mut();      // &mut [MaybeUninit<u8>]  ← reserve
let written = read_into(spare);              // initialize spare[..written]

// SAFETY:
// - `written <= spare.len()`,
// - `spare[..written]` is now initialized,
// - `old_len + written <= buf.capacity()` (spare len == capacity - old_len).
unsafe { buf.set_len(old_len + written); }   // ← commit
```

`spare_capacity_mut` is the reserve step (hand back uninitialized slots *after* the live elements); `set_len` is the commit step. `set_len` is `unsafe` for the very same reason Option A's raw `commit` is: the API can't verify the count you pass. And note the bug this pattern quietly avoids - the tempting shortcut `set_len(written)` is correct *only* when `old_len == 0`; on an already-populated buffer it truncates the live prefix, and `set_len` won't stop you, because it does exactly what you tell it. The lesson reserve/commit keeps teaching, in `Vec` and in the ring alike: **initialization tracking is the hard part, not getting a pointer.**

### Bonus: skip the teardown when `T` has no drop glue

A generic container has wildly different destruction costs at `Ring<u64>` versus `Ring<String>`. `std::mem::needs_drop::<T>()` is a `const fn` that resolves to a compile-time-known bool for a monomorphic `T`, so you can gate the per-slot walk on it and let the trivial case compile to nothing:

```rust
impl<T, A: BufferAllocator> Drop for Ring<T, A> {
    fn drop(&mut self) {
        if !std::mem::needs_drop::<T>() {
            return; // Ring<u64>, Ring<[u8; 16]>, …: no-op.
        }
        // SAFETY: only the [head, tail) range is initialized.
        unsafe { self.drop_initialized_slots(); }
    }
}
```

For `Ring<u64>` this collapses to an empty `drop`; for `Ring<String>` the walk runs. Any time a generic container does per-element teardown over `MaybeUninit` storage, gate it on `needs_drop::<T>()` so the trivially-droppable case doesn't pay for loop machinery it can't use.

---

## Technique 2 - Fast-path/slow-path splitting

Zero-copy removed the per-item *copy*. Fast/slow-path splitting removes the per-item *rare-case cost*.

The observation: most operations on a healthy data structure succeed without contention, allocation, or a system call. So structure every operation so the 99%-of-calls case is a short, predictable branch over local data, and let the rare case pay for correctness off to the side:

```rust
pub fn op(&self, /* … */) -> Result {
    // 1. Cheap argument validation.
    if invalid { return early; }

    // 2. Fast path - local data only, no cross-core traffic.
    if local_check_says_ok {
        return Ok(do_it());
    }

    // 3. Slow path - synchronize, refresh caches, retry.
    self.refresh_caches();
    if still_no_good { return Err(/* … */); }
    Ok(do_it())
}
```

You've already seen a concrete instance of this: [Part 3's `reserve()`](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/) checks the producer-owned cached `head` on the fast path and issues a cross-core `Acquire` load *only* when the cache says it's out of room. The caching technique from Part 3 is just this pattern with "the cache" as the local check and "refresh the cache" as the slow path.

**Why the split is worth the extra branch.** Branch predictors learn a consistently-taken path within a handful of iterations, after which the fast path is predicted essentially for free. Modern out-of-order cores then speculate *past* that predicted branch and overlap the whole fast-path computation with surrounding instructions. The slow path still costs what it costs - but it costs that *only when actually taken*, instead of taxing every call.

The same shape recurs everywhere once you look for it:

- lock-free `compare_exchange_weak` retry loops (fast: the CAS succeeds; slow: contention, reload and retry);
- `Vec::push` (fast: spare capacity remains; slow: reallocate and move);
- allocators (fast: a thread-local cache hit; slow: lock the central free-list).

**Help the compiler lay it out.** Inline the fast path aggressively (`#[inline]`) and keep the slow path out-of-line so it doesn't bloat the hot instruction footprint. The optimizer usually arranges this on its own, but marking the slow-path helper `#[cold]` nudges it to place that code away from the fast path and to predict the branch into it as not-taken.

---

## Where Parts 1–4 leave us

Four posts in, the hot path of a lock-free structure has been shaped from four independent directions: 
* its fields are laid out so cores don't fight over cache lines (Part 1)
* its cross-core reads and writes use the minimum-sufficient ordering (Part 2)
* those reads and writes happen as rarely as possible (Part 3)
* and each operation copies nothing it doesn't have to and branches predictably (Part 4)

None of these is a micro-optimization in isolation - together they're the difference between a structure that *looks* lock-free and one the hardware actually runs at full speed.

What's left is leverage the compiler can give you *for free* if you ask in the right language: const generics that bake sizes in at compile time, arithmetic and branch hygiene that lets the optimizer prove what it needs, and inlining discipline. We discuss that compile-time leverage in Part 5 - *Compile-Time Leverage*.
