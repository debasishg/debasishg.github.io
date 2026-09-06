+++
title = "How Miri found a data race in a lock-free ring buffer that the compiler had no way to see"
date = 2026-09-06
description = "A lock-free MPSC ring buffer passed its test suite, a Quint model check, loom, and cargo miri test - and was still undefined behaviour. A silent Deref widened every buffer reference to span the whole allocation, and a retag counts as an access."
template = "page.html"

[taxonomies]
tags = [
  "rust",
  "unsafe-rust",
  "miri",
  "stacked-borrows",
  "tree-borrows",
  "aliasing",
  "undefined-behavior",
  "data-race",
  "lock-free",
  "spsc-queue",
  "ring-buffer",
  "testing"
]
+++

# How Miri found a data race in a lock-free ring buffer that the compiler had no way to see

### The Reference That Was Too Wide

There is the usual story Rust programmers tell about data races: the borrow checker prevents them. Aliasing XOR mutation, enforced at compile time, and the whole category disappears.

The story has a footnote, and the footnote is `unsafe`. Inside an `unsafe` block you are not asking the compiler to check your reasoning - you are asserting a conclusion it will then rely on. And the rules you are asserting compliance with, are *not* the rules of the borrow checker. They are the rules of Rust's aliasing model, which is stricter, subtler, and largely invisible in the source text.

Maybe you're wondering why does Rust's aliasing model have a different and stricter ruleset than the borrow checker. Let's unpack it a bit ..

>The borrow checker is a static analysis over safe code: one `&mut` or many `&`, lifetimes must not outlive their referent. When you write unsafe, that analysis stops applying to the raw pointers you're manipulating. But the semantics of references don't stop applying. Rust has a separate, dynamic model of what a reference means - what permissions it claims over what range of memory, and whose permissions it revokes when it comes into existence. That's the aliasing model: currently Stacked Borrows (the deployed one) and Tree Borrows (its likely successor). unsafe exempts you from the checker, not from the model. 

This unfolds the story of a bug in `ringmpsc`, a lock-free MPSC channel built on ring decomposition. The SPSC protocol at its heart was correct. The memory ordering was correct. Every slot each thread touched was genuinely disjoint from every slot the other thread touched. The code passed its full test suite, a TLA+/Quint model check, loom's exhaustive interleaving search, and `cargo miri test`.

It was still undefined behaviour, and it had been from the beginning.

---

## 1. The setup

`ringmpsc` gives each producer its own SPSC ring, so producers never contend with each other. Within a single ring the classic protocol applies: the producer owns the slots in `[tail, head + capacity)`, the consumer owns `[head, tail)`, and the two sets never intersect. Publication happens through a `Release` store on `tail`, the consumer picks it up with an `Acquire` load.

Take a look at the full implementation at the github [repo](https://github.com/debasishg/ringmpsc-rs/tree/main/crates/ringmpsc) ..

The buffer is generic over an allocator, so the ring can be backed by a plain heap allocation, huge pages, or NUMA-local memory. The trait looks like this (`crates/ringmpsc/src/allocator.rs`):

```rust
pub unsafe trait BufferAllocator: Send + Sync {
    /// The owned buffer type returned by the allocator.
    ///
    /// Must dereference to a contiguous `[MaybeUninit<T>]` slice and
    /// handle its own deallocation on drop.
    type Buffer<T>: Deref<Target = [MaybeUninit<T>]> + DerefMut;

    fn allocate<T>(&self, capacity: usize) -> Self::Buffer<T>;
}
```

tl;dr That `Deref<Target = [MaybeUninit<T>]> + DerefMut` bound is the whole bug. Hold onto that thought till we uncover the subtleties ..

The ring stored the buffer in an `UnsafeCell`, because both threads need to reach it through a shared `&self`:

```rust
pub struct Ring<T, A: BufferAllocator = HeapAllocator> {
    tail: CacheAligned<AtomicU64>,
    cached_head: CacheAligned<UnsafeCell<u64>>,
    head: CacheAligned<AtomicU64>,
    cached_tail: CacheAligned<UnsafeCell<u64>>,
    // ...
    buffer: UnsafeCell<A::Buffer<T>>,
}
```

And every access to a slot went through that cell. Producer side, in `make_reservation`:

```rust
// SAFETY: Buffer access is safe because:
// 1. idx is within bounds (masked to capacity)
// 2. These slots are not being read by consumer (they're beyond current tail)
// 3. Only the producer writes to slots between tail and tail+n
// 4. The Reservation's commit() will publish via Release store to tail
let slice = unsafe {
    let buffer = &mut *self.buffer.get();
    &mut buffer[idx..idx + contiguous]
};
```

Consumer side, in `consume_batch` and its three siblings:

```rust
// SAFETY: Buffer access is safe because:
// 1. idx is within bounds (masked to capacity)
// 2. Items in [head, tail) were fully written by producer
// 3. The Acquire load on tail synchronizes with producer's Release store
// 4. assume_init_read moves ownership out - item will be dropped after handler
// 5. Only consumer reads these slots; after head advances, slots are "empty"
//    and producer can reuse them
let item = unsafe {
    let buffer = &*self.buffer.get();
    buffer[idx].assume_init_read()
};
```

Those `// SAFETY:` comments are the reason this bug is worth writing about. In this project we mandate one on every `unsafe` block, citing the invariant that makes it sound, and these were not casually authored. Read them clause by clause and **every single one is true**. `idx` really is masked to capacity. The producer's slots really are beyond the current tail. The `Acquire`/`Release` pair really does establish the happens-before edge. `assume_init_read` really does move ownership out for a clean `Drop`.

Nine numbered clauses across the two sites, every one of them true, and the code is still undefined behaviour - because all nine are about the **slots**, and the bug is in the **reference**. Not one clause says anything about how wide a reference the code is about to create, which is the only question that mattered. The comments answered "am I allowed to touch this memory?" and never asked "what exactly am I claiming when I form this reference?"

The slots are disjoint. That was never the problem.

---

## 2. What the compiler saw, and why it said nothing

Consider what the type system actually knows at each of those sites.

`self.buffer.get()` returns `*mut A::Buffer<T>`. A raw pointer. The moment you write `.get()` on an `UnsafeCell`, you have stepped outside everything the borrow checker tracks. It will not tell you how many other threads hold the same pointer, whether the region you are about to touch overlaps someone else's, or when the reference you derive from it is wider than your claim.

`&mut *self.buffer.get()` produces a `&mut A::Buffer<T>` out of thin air, with a lifetime the compiler infers from context rather than derives from ownership. That is precisely what `unsafe` is for: you are telling the compiler *I have checked this*. It believes you. It then goes further - it *optimizes on the assumption that you were right*, which is the part that makes being wrong expensive and embarrassing.

So the compiler was never going to catch this. There is no analysis it could run. The information it would need - that another thread is concurrently inside the same allocation - is not present in a single function's types, and Rust's aliasing rules for raw pointers are deliberately not checked statically. They are a *contract*, and the tool that checks contracts at runtime is [Miri](https://github.com/rust-lang/miri/), the Undefined Behavior detection tool for Rust.

---

## 3. The actual bug: `Deref` widens the reference

Look again at the producer line, and this time trace the types rather than the intent:

```rust
let buffer = &mut *self.buffer.get();   // &mut A::Buffer<T>
&mut buffer[idx..idx + contiguous]      // ???
```

`A::Buffer<T>` is not a slice. It is `Box<[MaybeUninit<T>]>` for the default heap allocator. Indexing it requires `DerefMut`, which the compiler inserts silently:

```rust
<Box<[MaybeUninit<T>]> as DerefMut>::deref_mut(buffer)  // &mut [MaybeUninit<T>]
                                                        // ^ spanning ALL capacity slots
```

**A mutable reference covering the entire ring buffer comes into existence**, and only on the *next* step does the index expression narrow it to `[idx, idx + contiguous)`.

Written as a chain, with the step the source never shows you marked:

```text
   self.buffer.get()          *mut A::Buffer<T>            no reference yet
         |
         |  &mut *_
         v
   &mut A::Buffer<T>          reference to the Box *handle*
         |
         |  buffer[idx..idx+n]   <-- compiler inserts DerefMut here
         v
   &mut [MaybeUninit<T>]      ***** RETAG OVER ALL CAPACITY SLOTS *****
         |                          this line does not exist in the source
         |  index
         v
   &mut [MaybeUninit<T>]      narrowed to [idx, idx+n) -- one step too late
```

Every level of that chain is written on a single source line. The fourth one is where the damage happens, and it is the only one with no text of its own to point at in review.

The consumer's `&*self.buffer.get()` does the same thing through `Deref`: a shared reference over every slot in the allocation, narrowed one operation too late to slot `idx`.

That intermediate wide reference is not a compilation artifact you can wave away. In Rust's semantics it is a real event with real consequences, and the name for that event is a **retag**.

### Why a retag counts as an access

When a reference is created, copied, or moved, the aliasing model *retags* it: it records the new reference's permissions over its entire range, and revokes conflicting permissions held by others. Miri's own diagnostic explains why this is not bookkeeping you can ignore:

- retags occur on all (re)borrows as well as when references are copied or moved
- retags permit optimizations that insert speculative reads or writes
- therefore from the perspective of data races, a retag has the same implications as a read or write

That middle line is the crux. Because a `&mut [T]` asserts exclusivity over its whole range, the compiler is entitled to emit LLVM `noalias`, and `noalias` licenses speculative loads and stores anywhere in that range. The optimizer may hoist a read of a slot the source never mentions out of a branch, or sink a write, purely because you promised nothing else was looking.

So when the consumer forms a whole-buffer `&[MaybeUninit<T>]`, the model treats it as **reading every slot in the ring** - including the one the producer is writing at that instant. When the producer forms a whole-buffer `&mut [MaybeUninit<T>]`, the model treats it as **writing every slot in the ring** - including the ones the consumer is reading. The capacity does not matter; whatever it is, the reference covers all of it.

Here is the whole bug in one picture. A ring caught mid-flight - capacity is illustrative, three items live, `head` at 2 and `tail` at 5:

```text
                 0     1     2     3     4     5     6     7
              +-----+-----+-----+-----+-----+-----+-----+-----+
   buffer     |  .  |  .  |  A  |  B  |  C  |  .  |  .  |  .  |
              +-----+-----+-----+-----+-----+-----+-----+-----+
                          ^head             ^tail

   granted by the protocol  -- two disjoint sets

     consumer  [head, tail)
                          |<--------------->|

     producer  [tail, head+capacity), wrapping
                                            |<--------------->|
              |<--------->|

   retagged by `&*self.buffer.get()`  -- one set, covering both

              |<=============================================>|
```

The top half is what every `// SAFETY:` comment in section 1 was describing, and it is correct: two regions, no overlap, meeting only at the boundaries `head` and `tail` where atomics do the synchronizing. The bottom half is what the code actually asserts to the compiler. One bar, spanning everything, formed simultaneously by both threads.

A read and a write of the same location, on two threads, with no synchronization between them. That is the textbook definition of a data race, and in Rust a data race is unconditional UB.

And nothing exotic is required to reach it. `push` - the ordinary safe convenience method, the friendliest API the crate offers - goes straight through the widening deref on every call:

```rust
pub fn push(&self, item: T) -> bool {
    self.reserve(1).is_some_and(|mut r| {
        r.as_mut_slice()[0] = std::mem::MaybeUninit::new(item);   // ring.rs:686
        r.commit();
        true
    })
}
```

No hand-rolled reservation loop, no unusual entry point. Safe code, safe signature, whole-buffer retag. The SPSC protocol was not at fault. The *reference* was simply wider than the protocol permitted.

---

## 4. The test that finds it has to be small

Here is the part that should worry anyone maintaining concurrent code: `cargo +nightly miri test` **passed**. Not by luck - it would have passed forever.

The test that finally caught it is 57 lines (`crates/ringmpsc/tests/miri_overlap_test.rs`), and its most important line is a sizing decision: **the ring must be smaller than the workload**. That is what forces the producer to block and wait for the consumer to drain, and blocking is the only reason the two threads are ever live at the same instant. Twenty-four items into eight slots.

```rust
#[test]
fn spsc_producer_and_consumer_overlap() {
    const ITEMS: u64 = 24;

    // 8 slots against 24 items: the producer must block and wait for the
    // consumer to drain, which is what forces the two to be live together.
    let config = Config::new(3, 1, false);
    let channel = Arc::new(Channel::<u64>::new(config));

    let ch = Arc::clone(&channel);
    let producer = thread::spawn(move || {
        let p = ch.register().unwrap();
        for i in 0..ITEMS {
            while !p.push(i) {
                thread::yield_now();
            }
        }
    });

    let ch = Arc::clone(&channel);
    let consumer = thread::spawn(move || {
        let mut total = 0usize;
        let mut sum = 0u64;
        while total < ITEMS as usize {
            total += ch.consume_all(|item| sum += item);
            if total < ITEMS as usize {
                thread::yield_now();
            }
        }
        sum
    });

    producer.join().unwrap();
    let sum = consumer.join().unwrap();
    assert_eq!(sum, (0..ITEMS).sum::<u64>());
}
```

It runs natively in microseconds and passes. Under Miri it fails - reliably, not just occasionally, once Miri is asked to switch between threads more often than it does by default (section 5).

Three properties have to hold *at once* for a test to be able to catch this, and the existing suite never had all three in one place:

| Requirement | What the suite had instead |
|---|---|
| Two threads | `tests/miri_tests.rs` was entirely single-threaded - no number of assertions substitutes for a second thread |
| Overlapping in time | `test_fifo_ordering_multi_producer` joins every producer *before* consuming, so the two sides are never live together |
| Small enough to interpret | `test_concurrent_stress` genuinely overlaps, but at 400,000 items it never finishes under an interpreter |

Each of those is a reasonable test on its own terms. The bug lived in the gap between "we have concurrent tests" and "we have a concurrent test Miri can actually run" - and when the tool is an interpreter, "big enough to be realistic" is exactly the wrong instinct. Size for interleaving, not for throughput.

---

## 5. What Miri actually reported

Miri's threading support needs a nudge to explore interleavings aggressively. `-Zmiri-preemption-rate=0.1` raises the chance of a context switch at each step to 10%, which is what turns a theoretically-racy program into a reliably-failing one.

**Stacked Borrows** (the default model):

```console
$ MIRIFLAGS="-Zmiri-preemption-rate=0.1" \
    cargo +nightly miri test --test miri_overlap_test
```

```text
error: Undefined Behavior: Data race detected between (1) retag write on thread `unnamed-2`
and (2) retag read of type `std::boxed::Box<[std::mem::MaybeUninit<u64>]>` on thread `unnamed-3`
  --> crates/ringmpsc/src/ring.rs:457:29   (consumer)
  --> crates/ringmpsc/src/ring.rs:276:17   (producer)
```

Read the message emitted by Miri closely, because it is unusually informative. Not "data race between a write and a read" - **"data race between (1) retag write and (2) retag read."** Miri is not reporting that the two threads touched the same slot. It is reporting that the two threads' *reference creations* overlapped. The racing party is named as `Box<[MaybeUninit<u64>]>`: the whole buffer, not a slot.

**Tree Borrows** (`-Zmiri-tree-borrows`), a newer and in some respects more permissive model, rejects it too - and pins the blame even more precisely:

```text
error: Undefined Behavior: Data race detected between (1) non-atomic write on thread `unnamed-2`
and (2) retag read of type `[std::mem::MaybeUninit<u64>]` on thread `unnamed-3`

  (2) <Box<[MaybeUninit<u64>]> as Deref>::deref
      at ringmpsc_rs::Ring::<u64>::consume_batch  ->  crates/ringmpsc/src/ring.rs:458
  (1) crates/ringmpsc/src/ring.rs:686   (Ring::push)
```

There it is in the stack trace, by name: `<Box<[MaybeUninit<u64>]> as Deref>::deref`. The offending operation is a `Deref` call that appears nowhere in the source - inserted by the compiler to make `buffer[idx]` typecheck.

Two independent aliasing models, disagreeing about plenty of other things, agreeing about this one.

Was there an observed compilation failure? No. The slots really are disjoint, so the generated code did the right thing on every machine it ran on. But `noalias` on that deref'd `&mut [MaybeUninit<T>]` is a standing invitation, and "the optimizer has not exercised its rights yet" is not a safety property. It is a deadline.

---

## 6. The fix: never form a wide reference again

The problem is not the pointer arithmetic and not the protocol - it is the intermediate `&`/`&mut` that spans the allocation. So remove it. Derive the base pointer **once**, at construction, while access is still genuinely exclusive, and use pointer arithmetic forever after.

A new field:

```rust
/// Raw pointer to slot 0 of the backing buffer.
///
/// Derived exactly once in [`Ring::new_in`], before the ring is shared with
/// any other thread, and never re-derived afterwards. Every concurrent
/// access goes through this pointer instead of through `buffer`'s
/// `Deref`/`DerefMut`.
///
/// This matters for soundness, not just speed. `A::Buffer<T>` derefs to
/// `[MaybeUninit<T>]` spanning the *whole* allocation, so `&*self.buffer.get()`
/// retags every slot in the ring - including the slots the other side of the
/// SPSC protocol is concurrently writing. A retag counts as an access for the
/// purposes of the aliasing model, so that races with the producer's write
/// even though the slots each side actually touches are disjoint.
base: *mut MaybeUninit<T>,
```

And a single accessor that forms no reference at all:

```rust
/// Raw pointer to slot `idx`.
///
/// Forms no reference, so nothing is retagged beyond the single slot the
/// caller goes on to touch. This is what keeps producer and consumer from
/// racing on the buffer as a whole - see the docs on [`Ring::base`].
///
/// # Safety
///
/// `idx` must be less than `capacity()`. The caller must additionally hold
/// the SPSC protocol's claim to slot `idx` (producer: `[tail, head+capacity)`,
/// consumer: `[head, tail)`) before reading or writing through the result.
#[inline]
unsafe fn slot(&self, idx: usize) -> *mut MaybeUninit<T> {
    debug_assert!(idx < self.capacity(), "slot index {idx} out of bounds");
    // SAFETY: `base` points at `capacity()` contiguous slots and `idx` is
    // in bounds per the precondition.
    unsafe { self.base.add(idx) }
}
```

Every call site then narrows *first* and forms a reference *second* - or forms no reference whatsoever. The chain from section 3, redrawn:

```text
   self.base                  *mut MaybeUninit<T>          no reference
         |
         |  .add(idx)              <-- narrowing happens HERE, on a raw pointer
         v
   *mut MaybeUninit<T>        still no reference; nothing retagged
         |
         |  from_raw_parts_mut(_, contiguous)
         v
   &mut [MaybeUninit<T>]      retag over exactly [idx, idx+contiguous)
```

Same destination, same number of source lines, and no step anywhere in the middle that covers more than the protocol grants. Redrawn onto the ring picture, the bottom bar now sits exactly under the top one instead of spanning the whole buffer.

The producer builds its slice over exactly its own region:

```rust
// SAFETY: Buffer access is safe because:
// 1. idx is within bounds (masked to capacity)
// 2. These slots are not being read by consumer (they're beyond current tail)
// 3. Only the producer writes to slots between tail and tail+n
// 4. The Reservation's commit() will publish via Release store to tail
// 5. The slice spans exactly [idx, idx+contiguous), the producer's own
//    region, so its retag cannot overlap slots the consumer holds.
let slice = unsafe { std::slice::from_raw_parts_mut(self.slot(idx), contiguous) };
```

The consumer's `readable()` does the same on its side:

```rust
// SAFETY: Buffer access is safe because:
// 1. idx is within bounds (masked to capacity)
// 2. Items in [head, tail) were written by producer and published via Release
// 3. The Acquire load on tail synchronizes with that Release
// 4. Only consumer reads these slots; producer won't overwrite until head advances
// 5. The slice spans exactly [idx, idx+contiguous), the consumer's own
//    region, so its retag cannot overlap slots the producer is writing.
unsafe {
    Some(std::slice::from_raw_parts(
        self.slot(idx).cast::<T>(),
        contiguous,
    ))
}
```

There is still a retag here - `from_raw_parts` creates a reference - but it now covers exactly the range the protocol grants. Two retags over disjoint ranges are not a race.

And the four consume loops stop forming a reference at all, copying the one slot out by value:

```rust
// SAFETY: Buffer access is safe because:
// 1. idx is within bounds (masked to capacity)
// 2. Items in [head, tail) were fully written by producer
// 3. The Acquire load on tail synchronizes with producer's Release store
// 4. assume_init_read moves ownership out - item will be dropped after handler
// 5. Only consumer reads these slots; after head advances, slots are "empty"
//    and producer can reuse them
// 6. Reading through `slot()` copies the one slot out without
//    forming a reference that spans slots the producer owns.
let item = unsafe { self.slot(idx).read().assume_init() };
```

Notice what happened to those comments. Clauses 1 through 5 are untouched - they were true before and they are true now. The fix shows up as an *added* clause, the one that answers the question section 1 said was never asked: how wide is the reference this code creates? That is the shape a fix of this kind takes. Nothing in the old reasoning was retracted, a missing dimension of it was filled in.

`Drop` was converted too. It holds `&mut self`, so nothing can race with it - but a wide retag there would be pointless and `base` is already the right pointer:

```rust
// Safety: idx bounded by mask; slot in [head, tail) is initialized (INV-INIT-01, INV-DROP-01).
// We hold `&mut self`, so there is no concurrent access, but we still
// go through `slot()` rather than deref the buffer: a wide retag here
// would be pointless and `base` is exactly the right pointer.
unsafe {
    ptr::drop_in_place((*self.slot(idx)).as_mut_ptr());
}
```

Uniformity is worth something on its own here. If one site still derefs the buffer, the next person to copy-paste a loop will copy that one.

---

## 7. The part that took three attempts: why there is an extra `Box`

Now let me talk about something that might have got lost in the story of wide references.

The buffer field did not merely lose its `Deref` calls. It changed type:

```rust
buffer: Box<UnsafeCell<A::Buffer<T>>>,
```

An extra heap indirection, wrapping something already heap-allocated. Try removing the `Box` and you end up reintroducing UB. Why is that?

**Retagging a `Box` asserts uniqueness over its pointee.** `Box` is special in the aliasing model: it carries a `noalias`-like guarantee, and *every move of a value containing a `Box` retags that `Box`*, invalidating pointers derived from its contents.

`Ring` gets moved constantly during setup. It is returned by value from `new_in`. It is pushed into `ChannelInner::rings`. Each of those moves retags any `Box` held inline - and `A::Buffer<T>` for the default allocator *is* a `Box`. So a `base` pointer derived from a bare inline buffer handle is invalidated by the very act of returning the ring from its constructor.

Miri caught two successive attempts at this, each looking perfectly reasonable in source:

1. **Obtain `base` before wrapping in `UnsafeCell`.** The subsequent move of the buffer handle into the cell invalidated the pointer.
2. **Obtain it after, via `get_mut()`.** Returning `ring` from `new_in` invalidated it - reported as *"Unique retag (of a reference/box inside this compound value)."*

The outer `Box` fixes this by giving the buffer *handle* a fixed heap address of its own. Moving the `Ring` moves a pointer to that handle; the handle itself never moves, so it is never retagged, so `base` stays valid for the ring's entire life:

```rust
pub fn new_in(config: Config, alloc: A) -> Self {
    let capacity = config.capacity();
    let buffer = Box::new(UnsafeCell::new(alloc.allocate::<T>(capacity)));

    // The one and only wide `&mut` over the buffer, taken while the ring is
    // still exclusively ours, so it races with nothing. `buffer` is already
    // in its final heap home, and the outer `Box` keeps it there, so this
    // pointer survives every later move of the `Ring`. See the field docs.
    let base = unsafe { (*buffer.get()).as_mut_ptr() };

    Self { base, /* ... */ buffer }
}
```

Note what that comment claims and why it is true. There *is* still exactly one wide `&mut` in the codebase - `as_mut_ptr()` on the deref'd slice. It is taken inside the constructor, before the ring has been shared with anything, at a moment when no other thread can possibly exist. A retag that races with nothing is not a race.

The cost is one allocation at construction, on a path that runs once per ring and never on a hot path.

One thing that did *not* need changing, though it is worth knowing about: a raw pointer field is neither `Send` nor `Sync`, so adding `base` costs `Ring` its automatic thread-safety markers. That turned out to be free here, because `UnsafeCell<A::Buffer<T>>` had already cost it the same markers, and the explicit assertions were already in place and already correct:

```rust
// Safety: Ring is Send + Sync as long as T is Send.
// The atomic operations ensure proper synchronization.
// BufferAllocator requires Send + Sync, and Buffer<T> requires Send.
unsafe impl<T: Send, A: BufferAllocator> Send for Ring<T, A> {}
unsafe impl<T: Send, A: BufferAllocator> Sync for Ring<T, A> {}
```

The new field simply added a second reason for an `unsafe impl` that was there already. 

---

## 8. Lessons Learnt

**Data races in Rust are not only about memory ordering.** The ordering here was, and remains, correct: `Release` on `tail`, `Acquire` on the read. Every atomic was right. The race was in reference *width* - a category of bug that the vocabulary of "acquire/release/relaxed" cannot even express.

**A `// SAFETY:` comment can be entirely true and still insufficient.** Nine numbered clauses across the two sites (section 1), all of them correct, none of them wrong about anything they asserted - and the code was UB regardless, because they were reasoning about the slots rather than the reference. The safety comment asked "Am I allowed to touch this memory?" as the obvious question. It never asked "how much memory does the reference I am creating actually cover?". Worth adding to the checklist alongside bounds and initialization.

**Silent `Deref` is a real hazard in `unsafe` code.** `buffer[idx]` where `buffer: &Box<[T]>` does not read like "materialize a reference to the entire allocation." Anywhere a smart pointer or newtype meets an `UnsafeCell` on a concurrent path, the auto-deref deserves to be traced out by hand - or, better, made impossible by holding a raw pointer instead.

**`Box` in a concurrently-accessed struct has semantics beyond indirection.** Its uniqueness guarantee interacts with moves in ways that invalidate derived pointers, and the invalidating move can be as innocuous as `return ring;`. And this broke two consecutive attempts at the fix.

**The test that finds a concurrency bug has to be small.** The suite already overlapped producer and consumer under real load - it was simply 400,000 items too large for Miri to reach the race. The one that found it does 24, into a ring of 8 (section 4). Two threads, overlapping in time, small enough to interpret: all three at once, or the tool has nothing to work with.

**Run both aliasing models.** Stacked Borrows and Tree Borrows disagree on real programs, and each catches things the other permits. `-Zmiri-preemption-rate=0.1` is what makes either of them reliably surface a thread-interleaving bug rather than occasionally.

The uncomfortable conclusion: this code had passed a formal model check of its protocol, exhaustive loom testing of its interleavings, and a Miri run. Every one of those tools was doing its job. None of them was looking at the thing that was broken, because the protocol was fine, the interleavings were fine, and the Miri tests had no second thread. Verification coverage is not the union of your tools' reputations - it is the intersection of what they each actually examined.
