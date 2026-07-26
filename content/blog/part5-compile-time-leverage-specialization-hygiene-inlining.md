+++
title = "Compile-Time Leverage: Specialization, Branch & Arithmetic Hygiene, and Inlining"
date = 2026-07-27
description = "How const generics, zero-sized types, branch and arithmetic hygiene, and disciplined inlining hand the Rust compiler enough compile-time facts to make sizes immediate, unused features free, and redundant checks disappear in a lock-free SPSC ring buffer."
template = "page.html"

[taxonomies]
tags = [
  "rust",
  "unsafe-rust",
  "monomorphization",
  "const-generics",
  "zero-sized-types",
  "inlining",
  "bounds-check-elimination",
  "branch-prediction",
  "lock-free",
  "spsc-queue",
  "ring-buffer",
  "performance"
]
+++

# Compile-Time Leverage: Specialization, Branch & Arithmetic Hygiene, and Inlining

Part 5 of **Low-Level Systems Design in Rust** - a series on writing high-throughput, low-latency systems code, using a single-producer / single-consumer (SPSC) ring buffer as the running example. 

[Part 1](https://debasishg.github.io/blog/part1-cache-conscious-data-layout-in-rust/) decided **where** the shared cursors of a concurrent structure live in memory.

[Part 2](https://debasishg.github.io/blog/part2-cross-core-contract-memory-ordering-and-single-writer-state/) covered **how** two cores read and write them correctly and at minimum cost.

[Part 3](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/) made those reads and writes *rare*. 

[Part 4](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/) talked about the **shape** of the operation itself - moving data without copying it, and structuring each call so the common case is a branch the CPU can predict and pipeline. 

The running example is still that ring buffer, but none of the three levers below is specific to ring buffers.

There are code snippets as part of the post. If you want to take a deeper dive into the ring buffer project, take a look at the code, tests and Quint specifications at the [repo](https://github.com/debasishg/ringmpsc-rs/tree/main/crates/ringmpsc).

---

## The compiler is your collaborator - feed it facts

Every technique in this post is the same move at a different scale: **hand the compiler a fact - a size, an invariant, the absence of a feature - and it optimizes on that fact, often by generating no code at all.** A capacity that's a compile-time constant folds into the instruction that uses it. A feature that's a zero-sized type compiles to nothing. A bounds check the compiler can *prove* redundant is deleted.

Three levers, from coarse to fine:

1. **Specialization** - bake known sizes and policies into types, so monomorphization generates code with the constants already substituted in.
2. **Branch & arithmetic hygiene** - phrase math and indexing so branches and bounds checks fold away instead of stalling the pipeline.
3. **Inlining** - let small things vanish at the call site, so cross-procedure optimization can fire across the seam.

---

## Lever 1 - Compile-time specialization

Rust *monomorphizes* generics: every concrete instantiation gets its own specialized machine code, the way C++ templates do, but without ad-hoc macros.  That's a performance tool, not just an ergonomic one - use it deliberately.

### Const generics bake sizes into the type

```rust
pub struct StackRing<T, const N: usize> {
    buffer: [UnsafeCell<MaybeUninit<T>>; N],
    // ...
}

impl<T, const N: usize> StackRing<T, N> {
    const MASK: usize = N - 1;     // computed at compile time
}
```

With `N` known at compile time, the buffer lives *inline* in the struct (no pointer indirection), `MASK` is a literal, and `seq & MASK` const-folds to a single `AND` against a constant encoded directly in the instruction.

**Why "no indirection" matters, concretely.** A heap-backed ring stores its buffer behind a pointer, so reaching a slot is two *data-dependent* loads: `self` → `Box<[T]>` → element. The second load can't even *start* until the first resolves - and even when both pointers are L1-resident, each costs ~4–5 cycles of latency you can't hide. With an inline `[T; N]`, `buffer[idx]` is a single base-plus-offset address the out-of-order engine resolves in parallel with surrounding work.

In one benchmarked implementation on an Apple M2, the inline-buffer variant ran **2–3× faster** than the heap variant on an otherwise identical algorithm. Where that comes from is worth pinning down, because the obvious explanation is the weakest one: measured single-threaded, the same comparison narrows to roughly 1.1×, and the gap only opens up in the threaded SPSC and MPSC runs. So the bulk of it is better spatial locality between the cursors and the buffer, *plus* a const-propagated capacity and mask, *plus* easier bounds-check elimination because the length is statically known. The one fewer indirection per slot access is real, but it's the smallest of the four. The right mental model isn't "inline is a faster knob" - it's "**inline hands the optimizer many more facts to work with**."

### `const fn new()` puts the structure in the binary

```rust
impl<T, const N: usize> StackRing<T, N> {
    // An associated const is evaluated when the type is monomorphized - not
    // when `new()` runs - so a bad `N` is caught at every call site.
    const ASSERT_POW2: () = assert!(N.is_power_of_two(), "N must be a power of 2");

    pub const fn new() -> Self {
        let () = Self::ASSERT_POW2;          // force evaluation
        Self {
            tail: CacheAligned::new(AtomicU64::new(0)),
            // ...
            buffer: unsafe { MaybeUninit::uninit().assume_init() },
        }
    }
}
```

A `const fn new()` lets you write `static RING: StackRing<u64, 4096> = StackRing::new();`. The whole structure is laid out in the binary's BSS segment - no heap allocation, no initialization cost at startup. Note the `ASSERT_POW2` assertion runs *at compile time*: a bad `N` fails the build, not the program.

### Zero-sized types make unused parameters free

When a generic parameter has a "do nothing" default, encode that default as a zero-sized type so it adds *zero bytes*:

```rust
pub struct HeapAllocator;                    // ZST: occupies 0 bytes

const _: () = assert!(
    std::mem::size_of::<HeapAllocator>() == 0,
    "HeapAllocator must be a zero-sized type",
);

pub struct Ring<T, A: BufferAllocator = HeapAllocator> {
    // ... `A` contributes 0 bytes when defaulted ...
}
```

The `const _: () = assert!(...)` is a **static assertion**: if a refactor ever gives `HeapAllocator` a field with a non-zero size, the build fails instead of silently growing the struct. A zero-sized field like `PhantomData<A>` still passes, which is what you want - it's the *bytes* the assertion guards, not the field count.

One ergonomic note while we're here: don't make callers spell out long generic lists. A type alias like `pub type StackRing4K<T> = StackRing<T, 4096>;` costs nothing at runtime and documents the common configuration.

### Static dispatch on the hot path: generics or `enum`, not `dyn`

The same monomorphization that makes const generics fast also makes *policy* parameters fast. The rule:

> Use `dyn Trait` at architectural boundaries; use generics or `enum` dispatch
> inside per-item loops.

A `dyn Trait` call is one indirect function-pointer load plus a branch - not slow in isolation, but it **blocks inlining**, and inlining is the gateway to constant propagation, dead-code elimination, and vectorization. On a per-item path, the *missed inline* usually costs far more than the vtable load itself.

This is exactly the lesson behind [Part 4's boxed-closure refactor](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/): replacing `Box<dyn FnOnce>` with a lifetime-tied `NonNull` removed both the allocation *and* the dynamic dispatch, and let the concrete method inline into the publish path. Generalized:


```rust
// Architectural boundary - a trait object is fine. One vtable load per
// channel construction; never touched on the hot path.
pub fn build_channel(backoff: Box<dyn BackoffPolicy>) -> Channel { /* ... */ }

// Hot path - a generic parameter. Each instantiation gets its own inlined
// body; no vtable lookup per spin.
pub struct Ring<T, B: BackoffPolicy = SpinThenYield> {
    backoff: B,
    // ...
}
```

When the set of implementations is small and closed, an `enum` + `match` is often the right middle ground - one binary handles every variant, the `match` predicts well, and the compiler can sometimes specialize each arm:

```rust
enum Transport {
    Tcp(TcpTransport),
    Unix(UnixTransport),
}

impl Transport {
    #[inline]
    fn send(&mut self, bytes: &[u8]) -> io::Result<usize> {
        match self {
            Transport::Tcp(t)  => t.send(bytes),
            Transport::Unix(u) => u.send(bytes),
        }
    }
}
```

Rule of thumb: for type fixed at construction and called per item, choose  **generic**, when chosen from a small closed set and called per item, choose **enum** and when a boundary is crossed rarely enough that one indirect call is noise, choose **`dyn Trait`**.

### Feature modes as types: make absence visible to the compiler

The ZST trick generalizes from *storage* parameters to *behavior* parameters: encode an optional feature as a type that compiles to nothing in the no-op case.

```rust
pub trait MetricsMode {
    fn sent(&self, n: u64);
}

pub struct NoMetrics;                         // ZST: zero bytes, no atomics
impl MetricsMode for NoMetrics {
    #[inline] fn sent(&self, _: u64) {}
}

pub struct AtomicMetrics {
    sent: AtomicU64,
}
impl MetricsMode for AtomicMetrics {
    #[inline]
    fn sent(&self, n: u64) { self.sent.fetch_add(n, Ordering::Relaxed); }
}

pub struct Ring<T, M: MetricsMode = NoMetrics> {
    metrics: M,
    // ...
}
```

`Ring<T, NoMetrics>` has a `sent()` call that inlines to *literally nothing* - the call site vanishes: no branch, no atomic read-modify-write, not even a counter byte on the cache line. `Ring<T, AtomicMetrics>` pays for the `fetch_add` exactly as if written inline - roughly 2 ns per call uncontended on Apple silicon, and considerably more if that counter shares a cache line with anything the other core touches. And the cost is visible in the **type**, so a reader can tell from the declaration which overheads are present.

Compare the familiar runtime-flag form:

```rust
if self.config.enable_metrics {
    self.metrics.add_messages_sent(n as u64);
}
```

That branch is well-predicted, but it's still a branch - and the counter field, the metrics code, and the config byte all sit in the binary and on the cache line even when the feature is off. The type-level version disappears entirely. Use the type-level form when the choice is *per-deployment* (a build either has metrics or doesn't); use the runtime flag when *one binary* must serve both modes. Same philosophy as the ZST allocator: **make the absence of a feature observable to the compiler.**

---

## Lever 2 - Branch & arithmetic hygiene

Every conditional branch is a potential pipeline stall, and every overflow check is a hidden branch. The hot path should be branch-light - and, more importantly, phrased so the optimizer can *prove* the branches and checks away.

**`wrapping_*` documents "wrapping here is harmless."** The default release profile doesn't trap on `+` anyway - though `overflow-checks = true` is a defensible hardening choice, and then it does - so the value of `wrapping_add` is the intent it records: *if* this ever wrapped, the algorithm's masking absorbs it (see [Part 3](https://debasishg.github.io/blog/part3-amortizing-cross-core-coordination-caching-and-batching/) on monotonic cursors):

```rust
let new_tail = tail.wrapping_add(n as u64);
```

**`saturating_sub` makes the free-space clamp unmissable.** This is the exact line from Part 3's `reserve()` fast path:

```rust
let space = self.capacity()
    .saturating_sub(tail.wrapping_sub(cached_head) as usize);
```

`tail.wrapping_sub(cached_head)` is the number of *occupied* slots, and free space is `capacity - occupied`. Write that guard by hand instead:

```rust
let used = tail.wrapping_sub(cached_head) as usize;
let space = if used <= self.capacity() {
    self.capacity() - used
} else {
    0
};
```

Both versions compile to the same machine code. LLVM recognizes the idiom and emits a branchless `subs` + conditional-select either way. So the argument for `saturating_sub` isn't codegen. It's that the clamp is stated outright instead of buried in a conditional. And a stated clamp is harder to break: nobody can "simplify" it away and reintroduce the underflow.

**Bounds-check elimination by giving the compiler a provable invariant.** A masked index carries its own proof: `idx = seq & mask` with `mask == capacity - 1` gives `idx <= mask < capacity`, and LLVM tracks that relationship. Indexing a single element with `buffer[seq & MASK]` on a `[T; N]` compiles with no check at all - no `unsafe` required.

Be precise about what that invariant covers, though: it bounds `idx`, not `idx + contig`.

```rust
// No bounds check: `seq & (N - 1)` is provably < N.
let value = buffer[seq & (N - 1)];

let idx = seq & (N - 1);

// Bounds check stays: nothing bounds `idx + n`.
let slice = &buffer[idx..idx + n];

// Bounds check gone: `contiguous` can't reach past the end.
let contiguous = n.min(N - idx);
let slice = &buffer[idx..idx + contiguous];
```

The `min` is what makes the third form provable - and it's the same `min` the ring already needs for correctness, since a reservation has to stop at the wrap point anyway. That's the shape to aim for: the bound falls out of code you had to write regardless. `StackRing::make_reservation` in [`stack_ring.rs`](https://github.com/debasishg/ringmpsc-rs/blob/main/crates/ringmpsc/src/stack_ring.rs) is exactly this - `n.min(N - idx)`, and nothing more.

Reach for `get_unchecked` only when profiling shows a check survived *and* matters, and keep the `// SAFETY:` justification tight.

**Don't panic on the hot path.** `unwrap()` and `expect()` compile to a branch plus a panic arm - and for `Result::unwrap` that arm pulls in `Debug` formatting for the error type. That formatting never executes on the happy path, so the cost isn't cycles spent. It's the branch, plus code the compiler has to keep reachable and lay out near your hot loop. On the hot path, prefer `Option`/`Result`-returning methods and let the caller handle the empty case; reserve `expect("reason")` for tests and `.expect("fatal: …")` for binary entry points.

**Shape the slice before you reach for `unsafe`.** A lot of "I had to drop to `unsafe` for speed" is really "I picked the wrong slice API." The standard library has methods that *prove shape facts* - to the optimizer and the borrow checker - that would otherwise need `unsafe` to assert by hand:

```rust
// Safe: proves disjointness. `left` and `right` cannot alias.
let (left, right) = buf.split_at_mut(mid);

// Safe: proves a fixed chunk size, so bounds checks fold away inside the loop.
for chunk in bytes.chunks_exact(64) {
    process_cache_line_sized_chunk(chunk);
}
let tail = bytes.chunks_exact(64).remainder();

// Unsafe: `align_to` is an `unsafe fn` because it reinterprets the element
// type. The (prefix, aligned, suffix) shape it returns is correct; the
// type-validity of reading bytes as u64 is the caller's responsibility.
// SAFETY: every byte pattern is a valid `u64`.
let (prefix, aligned, suffix) = unsafe { bytes.align_to::<u64>() };
```

The first two encode a fact (disjoint, fixed-size) entirely in *safe* code, and the optimizer uses it to elide bounds checks, unroll, and vectorize. This is why [Part 4's reservation returns a single contiguous `&mut [MaybeUninit<T>]`](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/) rather than a scatter/gather pair. A contiguous slice of known length is already the shape `copy_from_slice`, `io::Write`, and SIMD intrinsics want. Give the consumer that, and nobody has to reach for raw pointers to "make it fast." The rule: **express the fact you need in the slice API first, drop to `unsafe` only when the safe shape genuinely can't encode the proof - and then keep the unsafe region small enough that the shape fact is the only thing it relies on.**

---

## Lever 3 - Inlining discipline

`#[inline]` is a *hint*, not an order - and the compiler's own heuristics are usually right. Use the annotations where they buy something the heuristics can't see:

| Marker | Use it on |
|--------|-----------|
| `#[inline]` | Small accessors that cross a **crate boundary**; trivial wrappers (`is_empty`, `len`, `capacity`); convenience methods that should vanish at the call site. |
| `#[inline(always)]` | Tiny, branch-free helpers in the very hottest path - *sparingly*. Forcing inline can hurt the instruction cache. |
| `#[inline(never)]` / `#[cold]` | Slow-path branches, error formatters, panicking helpers. Keeps the hot path's I-cache footprint small. |

The crate-boundary case is the one the heuristics can't resolve on their own, because cross-crate inlining needs the function body available in the caller's compilation unit:

```rust
#[inline]
pub fn is_empty(&self) -> bool {
    self.tail.load(Ordering::Relaxed) == self.head.load(Ordering::Relaxed)
}
```

To inline a function, the compiler has to see its body. Inside one crate it always can. Across crates it usually can't, because a compiled crate ships machine code, not source. `#[inline]` is what tells the compiler to ship the body too. Leave it off and the caller emits a real function call, plus it loses whatever it could have folded away around that call.

Two things narrow where this actually matters. Generic functions already ship their bodies - they have to, since every caller stamps out its own copy - so `#[inline]` adds nothing there. And if you build with LTO (`lto = "thin"` or `"fat"`), cross-crate inlining happens anyway. What's left is the one case that really wants the annotation: a small, non-generic, public function, in a build without LTO.

The `#[cold]` half pairs naturally with [Part 4's fast-path/slow-path split](https://debasishg.github.io/blog/part4-zero-copy-reserve-commit-and-fast-slow-path-splitting/): mark the slow-path helper `#[cold]` so the compiler lays its code *away* from the hot path and predicts the branch into it as not-taken, shrinking the hot path's instruction footprint.

**The anti-pattern:** sprinkling `#[inline]` on large or rarely-called functions "just because." That bloats compile times, the instruction cache, and the binary with no payoff. Inline what should *disappear*; leave everything else to the optimizer.

---

## What this buys, and where the series goes next

The first four posts shaped what runs and how often. This one is about handing the compiler enough compile-time truth that some of it stops running at all: sizes become immediates, unused features become zero bytes, redundant checks get deleted, and small calls dissolve into their callers. None of it requires exotic tricks - just phrasing each fact in a form the optimizer can act on.

So far the whole series has assumed the data structure has the resources it needs and a consumer that keeps up. The next post relaxes both: what a producer should do when the ring is *full* (adaptive backoff instead of a naive spin), and where the backing memory actually comes from (custom allocators, NUMA placement, and huge pages). That's where Part 6, *Backoff & Memory Provisioning*, goes.
