+++
title = "Safety as Performance: Moves, the Borrow Checker, debug_assert!, and Cold-Path Hygiene"
date = 2026-08-24
description = "Rust's safety machinery is not a tax on performance but a tool for it: move semantics that delete copies, a borrow checker that deletes runtime guards, debug_assert! that vanishes in release, and cold-path hygiene that keeps the instruction cache clean."
template = "page.html"

[taxonomies]
tags = [
  "rust",
  "unsafe-rust",
  "ownership",
  "move-semantics",
  "borrow-checker",
  "debug-assert",
  "error-handling",
  "cold-path",
  "lock-free",
  "spsc-queue",
  "ring-buffer",
  "performance"
]
+++

# Safety as Performance: Moves, the Borrow Checker, `debug_assert!`, and Cold-Path Hygiene

Part 7 of **Low-Level Systems Design in Rust** - a series on writing high-throughput, low-latency systems code, using a single-producer / single-consumer (SPSC) ring buffer as the running example. 

[Part 1](https://debasishg.github.io/blog/part1-cache-conscious-data-layout-in-rust/) decided **where** the shared cursors of a concurrent structure live in memory.

[Part 2](https://debasishg.github.io/blog/part2-cross-core-contract-memory-ordering-and-single-writer-state/) covered **how** two cores read and write them correctly and at minimum cost.

[Part 3](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/) made those reads and writes *rare*. 

[Part 4](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/) talked about the **shape** of the operation itself - moving data without copying it, and structuring each call so the common case is a branch the CPU can predict and pipeline. 

[Part 5](https://debasishg.github.io/blog/part5-compile-time-leverage-specialization-hygiene-inlining/) pushed the remaining decisions to **compile time** - specializing on what the compiler already knows about a type, and inlining so the abstractions cost nothing once the code is built. 

[Part 6](https://debasishg.github.io/blog/part6-backoff-and-memory-provisioning-allocators-numa-huge-pages/) provisioned the two resources that steady state quietly assumed: **time**, by waiting well when the ring is full, and **memory**, by controlling where the backing buffer comes from and where on the machine it lives. 

The above 6 parts looked at a single lock-free structure from the hardware's side: cache lines, memory ordering, allocation, NUMA. This post changes the lens to the safety aspects of the *language*. The claim is that Rust's safety machinery - move semantics, the borrow checker, `debug_assert!`, and its error model - is not a tax you pay for safety but a set of tools you can spend on speed. 

The running example is still that ring buffer, but none of the four levers below is specific to ring buffers.

There are code snippets as part of the post. If you want to take a deeper dive into the ring buffer project, take a look at the code, tests and Quint specifications at the [repo](https://github.com/debasishg/ringmpsc-rs/tree/main/crates/ringmpsc).

---

## The tradeoff that mostly isn't

In a lot of systems languages, safety and performance pull against each other. A bounds check costs cycles, so the fast version drops it and hopes. A runtime guard prevents misuse, so the fast version trusts the caller instead. The mental model is "every safety check is a tax, performance means not paying it."

Rust's distinctive feature, powered by its type system, is that several of its safety mechanisms cost *nothing* at runtime - and a few of them let you *delete* work the unchecked version would still have to do. Encode a protocol rule in the type system and you remove the `if` that enforced it. Transfer ownership instead of copying and you remove the `memcpy`. Express an invariant as a `debug_assert!` and it vanishes from the release binary entirely. The following sections discuss four of these as explicit levers in your repertoire of low level system design.

---

## Lever 1 - Move semantics over cloning

Cloning is *not* free, even for "cheap" types. A clone dirties a cache line the original already owns, and for anything holding a heap allocation (`String`, `Vec`, `Box`, `HashMap`) it doubles allocation traffic - an absolute anti-pattern on hot paths. For the hot path, design so as to move a value *out* of the queue and into the consumer's storage with no copy at all. And Rust's affine type system makes this safe: once a value is moved, the compiler forbids touching the source again.

So as a designer of the consumer API, make sure you expose the *ownership contract* in the name of the API. Publish different contracts explicitly, do not conflate ownership semantics between them or you may land up into subtle, difficult-to-debug drop bugs: 

```rust
// 1. Visit: items stay in the ring; `head` is NOT advanced. A panic in the
//    handler leaves the item in place for next time. (peek / inspect / observe)
pub fn for_each_ref<F: FnMut(&T)>(&self, mut handler: F) -> usize { /* ... */ }

// 2. Drain by reference: the handler sees `&T` while the item is still in the
//    slot, then the item is dropped and `head` advances.
pub fn drain_for_each_ref<F: FnMut(&T)>(&self, mut handler: F) -> usize { /* below */ }

// 3. Drain by value: the handler receives `T`; ownership transfers and the
//    handler owns the drop (or stores the value). No clone, ever.
pub fn drain_for_each_owned<F: FnMut(T)>(&self, mut handler: F) -> usize {
    // ...
    // `slot()` is raw-pointer arithmetic off a base captured at construction;
    // see the sidebar below for why it is not `&*self.buffer.get()`.
    let item = unsafe { self.slot(idx).read().assume_init() };  // MOVES T out
    handler(item);                          // ownership transfers; no clone
    // ...
}
```

The by-value drain is the one that works best for `String`/`Vec`/`Box`/`HashMap`, because cloning would re-allocate. For `T: Copy` (like `u64`) the by-reference variant is fine - the compiler copies the bits cheaply where needed.

### The by-reference drain has one extra hazard

The by-value drain is essentially the panic-safe batch consumer from [Part 3](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/). Advance the cursor before calling the handler, and a `Drop` guard publishes progress on unwind. The *by-reference* drain has one more state to manage: the slot is **still initialized** while the handler borrows it, so if the handler panics, the guard has to know which slot is currently borrowed and drop it. Make that state machine explicit with an `armed_slot` field:

```rust
struct RefDrainGuard<'a, T, A: BufferAllocator> {
    ring:       &'a Ring<T, A>,
    current:    u64,             // last successfully consumed (exclusive)
    armed_slot: Option<*mut T>,  // Some while the handler borrows the slot
}

impl<T, A: BufferAllocator> Drop for RefDrainGuard<'_, T, A> {
    fn drop(&mut self) {
        // We are already unwinding from the handler. Advance past the slot
        // before dropping it, so the cursor we publish below never points at
        // a slot we have begun to destroy. (If `T::drop` itself panics here
        // that is a panic during unwind, which aborts - nothing below runs,
        // and no ordering could have saved us.)
        if let Some(ptr) = self.armed_slot.take() {
            self.current = self.current.wrapping_add(1);
            // SAFETY: the handler had `&T` to this slot; we are the unique
            // consumer and now reclaim ownership to drop it.
            unsafe { core::ptr::drop_in_place(ptr); }
        }
        self.ring.head.store(self.current, Ordering::Release);   // publish progress
    }
}

pub fn drain_for_each_ref<F: FnMut(&T)>(&self, mut handler: F) -> usize {
    let head = self.head.load(Ordering::Relaxed);
    let tail = self.tail.load(Ordering::Acquire);
    let avail = tail.wrapping_sub(head) as usize;
    if avail == 0 { return 0; }

    let mut guard = RefDrainGuard { ring: self, current: head, armed_slot: None };

    while guard.current != tail {
        let idx = (guard.current as usize) & self.mask();
        // SAFETY: the consumer is the unique writer of [head, tail) storage
        // per the SPSC protocol; the producer can't touch this slot until
        // `head` advances past it. `slot()` forms no reference, so nothing
        // is retagged beyond this one slot - see the sidebar below.
        let slot_ptr = unsafe { self.slot(idx).cast::<T>() };

        guard.armed_slot = Some(slot_ptr);          // arm: Drop owns it on panic
        handler(unsafe { &*slot_ptr });             // SAFETY: slot is initialized

        // Handler returned. Disarm and advance *before* running drop, so an
        // unwind from `T::drop` publishes the already-advanced cursor and the
        // slot is never double-dropped.
        guard.armed_slot = None;
        guard.current = guard.current.wrapping_add(1);
        // SAFETY: handler no longer borrows the slot; take ownership and drop.
        unsafe { core::ptr::drop_in_place(slot_ptr); }
    }

    avail
}
```

And remember that reading a slot *moves* the value out: read a slot twice - or let `Ring::drop` process a slot you already moved out of - and that's a double-drop.

### A reference is wider than you think

That `slot()` call deserves an explanation, because the obvious way to write it is wrong, and wrong in a way that compiles, passes every test, and is invisible to the borrow checker.

The obvious way is to reach into the buffer and index it:

```rust
let buf = unsafe { &mut *self.buffer.get() };   // DON'T
let slot_ptr = buf[idx].as_mut_ptr();
```

The backing store derefs to `[MaybeUninit<T>]`, so `buf` is a reference spanning the **entire** ring - every slot, including the ones the producer is writing right now. You narrow to a single slot on the very next line, but that is one line too late: the wide reference already existed. And under Rust's aliasing model, *creating* a reference is itself an access to everything it covers. Miri says so directly:

> retags occur on all (re)borrows as well as when references are copied or moved [...] therefore from the perspective of data races, a retag has the same implications as a read or write

The above obvious (but incorrect) way to reach into the buffer existed for a long time till Miri decides to hit one Sunday afternoon, while I was authoring this blog post.

So the consumer merely *forming* that reference counts as reading all `capacity` slots, which races with the producer's write. The protocol is impeccable - the slots each side actually touches are disjoint - and it is still undefined behaviour, because the reference claimed more than the protocol granted. This is not theoretical: I found exactly this bug in the ring buffer backing this series, on the ordinary safe `push` path, and Miri flags it under both Stacked and Tree Borrows.

The fix is to never form the wide reference. Capture the base pointer once at construction, while the ring is still exclusively yours, and do arithmetic on it thereafter:

```rust
#[inline]
unsafe fn slot(&self, idx: usize) -> *mut MaybeUninit<T> {
    debug_assert!(idx < self.capacity());
    // SAFETY: `base` covers `capacity()` contiguous slots; `idx` is in bounds.
    unsafe { self.base.add(idx) }
}
```

Raw pointers carry no aliasing claim, so nothing is retagged beyond the slot you go on to touch. Where you genuinely need a slice - handing a reservation to the producer - build it with `slice::from_raw_parts_mut(self.slot(idx), n)` over exactly your own region, rather than slicing a wide one down.

The general lesson is worth more than the fix. `unsafe` blocks are usually audited for the *access* - is this index in bounds, is this slot initialized - and the SAFETY comment above the offending line said all the right things about `[head, tail)`. What it missed was the reference's *extent*, which no amount of care about indices will catch. Bounds are about where you read, extent is about what you claimed the right to read. Miri checks the second; you and the compiler mostly check the first.

### The destructor must walk the live range *exactly*

Just to reiterate what to drop: items still in `[head, tail)` when the ring is dropped must be dropped exactly once, and items the consumer already drained must *not* be touched again:

```rust
impl<T, A: BufferAllocator> Drop for Ring<T, A> {
    fn drop(&mut self) {
        let head = self.head.load(Ordering::Relaxed);
        let tail = self.tail.load(Ordering::Relaxed);
        let count = tail.wrapping_sub(head) as usize;
        if count > 0 {
            // ... drop_in_place for each slot in [head, tail) ...
        }
    }
}
```

It's extremely important you get the range right or else you have a double-free. Skip any part of it and you have a leak. This is precisely the kind of property a `DropTracker` test - a `T` that counts its own constructions and destructions and asserts they balance - exists to pin down.

---

## Lever 2 - The borrow checker as a protocol enforcer

The borrow checker is usually sold as a *memory-safety* tool. However it can also be used towards enforcing your *protocol* invariants at compile time, so you can delete the runtime checks that would otherwise guard them. A somewhat similar analogy could be just like static typing lets you remove lots of tests that you need with dynamic typing, the borrow checker gives you the same leverage on top of static typing.

Look at the reservation handle from [Part 4](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/), and notice exactly which method takes which borrow:

```rust
pub struct Reservation<'a, T, A: BufferAllocator> {
    slice:   &'a mut [MaybeUninit<T>],
    ring:    NonNull<Ring<T, A>>,
    len:     usize,
    _borrow: PhantomData<&'a mut Ring<T, A>>,
}

impl<T, A: BufferAllocator> Producer<T, A> {
    pub fn reserve(&mut self, n: usize) -> Option<Reservation<'_, T, A>> {
        // One outstanding reservation per producer follows from `&mut self`.
    }
}

impl<T, A: BufferAllocator> Reservation<'_, T, A> {
    pub fn commit(self) { /* takes `self` by value: the handle is consumed */ }
}
```

The `NonNull` and the `PhantomData` are both [Part 4](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/)'s doing, and it makes the case for them there. What matters here is the receiver on `reserve`.

Three protocol rules are enforced here **without a single runtime check**, and it is worth being precise about which piece of the signature buys which:

| Guarantee | Bought by |
|---|---|
| At most one outstanding `Reservation` per producer | `reserve` taking `&mut self` |
| The ring cannot be dropped while a reservation lives | the returned `Reservation<'_, …>` borrowing at all |
| Using a reservation after `commit()` is a compile error | `commit` taking `self` **by value** |

Only the first genuinely needs `&mut`. The second holds for `&self` just as well - *any* borrow keeps the borrowed thing alive, uniqueness has nothing to do with it. The third is pure affine typing: `commit(self)` consumes the handle, so the use-after-commit error would still be there if `reserve` took `&self` and never changed.

That leaves `&mut self` doing exactly one job, and it is the important one. Had `reserve` been `&self`, two calls would borrow-check happily and hand out two `&mut` slices over the *same* slots - the reservation cursor doesn't move until `commit`. You would need an explicit reservation flag or a runtime guard to prevent the overlap, which is precisely the `if` the receiver type deletes.

A separate mechanism closes another door and enforces another part of the protocol: `Producer` deliberately does not implement `Clone`. That is what keeps one ring to one producer, and the receiver on `reserve` has nothing to do with it. Producers come only from a registration call that mints a fresh id and a fresh ring, so a non-cloneable handle means that id can never be duplicated. No runtime guard, no CAS to claim exclusive write access - the type system is the enforcement.

Two mechanisms, two invariants: `&mut self` stops one producer from overlapping itself; `!Clone` stops a second producer from existing. Neither substitutes for the other.

The lesson learnt:

> **Every protocol invariant you can encode in the type system is one fewer `if` on the hot path.**

It's the same family as typestate APIs (`TcpListener` -> `TcpStream`), affine resource handles (`File`, mutex guards), and sealed traits - all of which move a runtime check into the compiler.

---

## Lever 3 - Invariants that vanish in release: `debug_assert!`

Lock-free code is easy to break and miserable to debug. One of the practices that I follow is to write down the invariants you depend on as `debug_assert!`s: they run in tests and debug builds and compile to *nothing* in release, so they cost zero on the hot path you actually ship.

The trick that makes them maintainable is to keep them in one module of named macros, each citing the spec invariant it enforces, so a reviewer can audit code against specification:

```rust
macro_rules! debug_assert_bounded_count {
    ($count:expr, $capacity:expr) => {
        debug_assert!(
            $count <= $capacity,
            "INV-SEQ-01 violated: count {} exceeds capacity {}",
            $count, $capacity
        )
    };
}

macro_rules! debug_assert_monotonic {
    ($name:literal, $old:expr, $new:expr) => {
        debug_assert!(
            $new >= $old,
            "INV-SEQ-02 violated: {} decreased from {} to {}",
            $name, $old, $new
        )
    };
}

// `macro_rules!` is textually scoped, not path-scoped: defining these in a
// module does not make them visible anywhere else in the crate. Re-export
// them, or every call site below fails to resolve.
pub(crate) use debug_assert_bounded_count;
pub(crate) use debug_assert_monotonic;
```

Call sites then read like prose, right next to the operation they protect:

```rust
debug_assert_bounded_count!(new_tail.wrapping_sub(head) as usize, self.capacity());
debug_assert_monotonic!("tail", tail, new_tail);
self.tail.store(new_tail, Ordering::Release);
```

Three properties make this the right way to verify a hot path: 
- zero release cost (the assert is removed entirely), 
- named invariants that tie code to a spec, and 
- immediate failure under the default `cargo test` (which is a debug build). 

And for invariants checkable *statically*, escalate from `debug_assert!` to the compile-time `const _: () = assert!(...)` from [Part 5](https://debasishg.github.io/blog/part5-compile-time-leverage-specialization-hygiene-inlining/) - then the build fails on violation and no test is needed at all.

### There is a rung above that

The best outcome for an assertion is not that it gets cheaper. It is that it becomes unrepresentable, and you delete it.

The ring buffer had an invariant called INV-RES-03, "the reservation's pointer back to its ring is never null," enforced the obvious way:

```rust
macro_rules! debug_assert_valid_ring_ptr {
    ($ptr:expr) => {
        debug_assert!(!$ptr.is_null(), "INV-RES-03 violated: null ring pointer")
    };
}
```

Textbook by the standards of this section: named, spec-linked, free in release. It fired on every commit in debug builds for as long as the field was declared `ring_ptr: *const Ring<T, A>`.

Then that field became `NonNull<Ring<T, A>>` - for unrelated reasons, in a change about documenting the invariant rather than checking it. The compiler noticed immediately:

```
warning: returned pointer of `as_ptr` call is never null,
         so checking it for null will always return false
```

The assertion had not become cheap. It had become *impossible to fail*, because null was no longer a value the field could hold. The right response was to delete the macro and leave a comment where it stood, recording that INV-RES-03 is now enforced by the type rather than by a check.

That is the whole ladder in one invariant:

| Rung | Enforcement | Cost |
|---|---|---|
| Runtime `assert!` | every build | cycles on the hot path |
| `debug_assert!` | debug and test builds | zero in release |
| `const _: () = assert!(...)` | compile time | zero, and the build fails |
| Make it unrepresentable | the type system | zero, and there is nothing left to check |

Each rung removes not just cost but a *category of failure*. So when you write a `debug_assert!`, it is worth asking one more question before moving on: is this a fact I am checking, or a fact I could have made structural? The check is the fallback, not the goal. And you will not always spot the answer yourself - here the compiler found the dead check, on a change made for another reason entirely, which is its own argument for keeping warnings loud.

---

## Lever 4 - Keep cold code out of the hot instruction cache

The last lever is about the instruction cache, which is finite. Code that *sits* in your hot loop but rarely *runs* - error formatting, panic machinery, debug-only branches - evicts the code that does run. Rust's error model gives you the tools to remove it.

**Use small, `Copy`, structured error enums.** A flat enum with a tiny payload means an error return neither allocates nor drags in formatting code unless the error is actually displayed:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Error)]
pub enum ChannelError {
    #[error("too many producers registered (max: {max})")]
    TooManyProducers { max: usize },
    #[error("channel is closed")]
    Closed,
}
```

Compare with `Box<dyn Error>` or `String`, both of which force heap traffic onto the cold path that the optimizer still has to plan around.

**Keep `format!`/`panic!` out of the hot function's reachability graph.** A `panic!("{:?}", thing)` on a branch that almost never fires still pulls the entire `Debug` chain into the function and bloats its footprint - the code has to exist and be laid out somewhere, and "somewhere" defaults to right next to the code that runs every iteration. (Genuinely unreachable code is a different story; the optimizer deletes that. It's the *rare-but-real* branch that costs you.) Hoist panic helpers behind `#[cold] #[inline(never)]`:

```rust
#[cold]
#[inline(never)]
fn report_corruption(seq: u64) -> ! {
    panic!("seq {} out of range", seq)
}
```

This is the same `#[cold]` discipline from [Part 5](https://debasishg.github.io/blog/part5-compile-time-leverage-specialization-hygiene-inlining/): the slow, big, rarely-taken code gets moved out of line, into a distant part of the binary, so the hot loop's I-cache footprint stays small. The payoff is layout, not branch prediction - the predictor learns a consistently-not-taken branch on its own within a few iterations. What it cannot do is un-evict the cache lines that the cold code displaced.

**Feature-gate observability.** Metrics, tracing spans, and audit logging belong behind a flag - and ideally a compile-time one, which connects straight back to Part 5's "feature modes as types":

```rust
if self.config.enable_metrics {
    self.metrics.add_messages_sent(n as u64);
}
```

A runtime flag is a well-predicted branch, but the *code* and the counter field are still present even when disabled. Gate it with `#[cfg(feature = …)]` (or the ZST-mode type from Part 5) and it disappears at compile time.

---

## Where this leaves us

The thread through all four levers is the same: in Rust, the safe construction is frequently *also* the fast one. Moving instead of cloning removes a copy and is enforced by affine types. `&mut self` removes the runtime guard against overlapping reservations, and a non-`Clone` handle removes the one against a second producer - two mechanisms, two invariants, neither standing in for the other. `debug_assert!` gives you invariant checking that isn't in the shipped binary. Small error types and `#[cold]` helpers keep the cold path from taxing the hot one. None of these asks you to choose between correct and fast.

The one caveat the `slot()` sidebar earns: none of this makes `unsafe` blocks audit themselves. The type system enforces what you encoded in it, and a whole-buffer `&mut` inside an `unsafe` block encodes nothing at all. That is exactly the seam Part 8 is about.

That sets up the final question we all have been looking for in the series: once you've built something this intricate, how do you *trust* it? The last post is about making performance code trustworthy - Loom and Miri for concurrency, model-checking the protocol, the supporting Rust idioms, the anti-patterns to watch for, and a final all-in-one checklist. That's where Part 8 - *Trustworthy Performance* goes.
