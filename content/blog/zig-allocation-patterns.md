+++
title = "Zig Allocation Patterns"
date = 2026-05-17
description = "A practical reference on Zig allocation patterns - breaking down the aware (unmanaged) vs. owning (managed) ownership models, arenas, index arenas, and other key idioms used in porcupine-zig, with real code examples and tradeoffs"
template = "page.html"

[taxonomies]
tags = ["zig", "design-patterns", "allocator"]
[extra]
+++

# Zig Allocation Patterns

A reference for the allocation and ownership patterns used in [`porcupine-zig`](https://github.com/debasishg/porcupine-zig), plus the broader landscape of Zig allocator idioms. Examples cite types and functions in the codebase by name so the patterns can be grounded against real code. But the discussion in this post stands on its own and does not require reading those sources.

---

## 1. The two ownership models: aware vs. owning

Both patterns satisfy the project rule "every allocating function takes an explicit `std.mem.Allocator`." The split is about **whether the type also stores a copy of that allocator inside itself.**

### 1.1 Allocator-aware (Unmanaged)

The type holds **no** allocator field. Every method that allocates or frees re-receives one as a parameter, and the caller is responsible for passing the *same* allocator on `init`, `clone`, and `deinit`.

```zig
var bs = try Bitset.init(arena_alloc, 256);
defer bs.deinit(arena_alloc);          // caller re-supplies arena_alloc
const cp = try bs.clone(arena_alloc);  // and again
```

**Use it when:**

1. **Many short-lived instances share one allocator.** The custom `Bitset` used by the checker clones constantly inside the DFS - one per cache entry, often hundreds or thousands per partition. All of them live and die under the same per-worker arena. Storing a 16-byte `std.mem.Allocator` (vtable pointer + state pointer) on every instance would inflate the type for zero information gain - it's the same allocator every time.

2. **The type is small and SBO-sensitive.** `Bitset` deliberately keeps a `[4]u64` inline buffer to dodge heap allocation for histories ≤ 256 ops.  Adding 16 bytes of allocator metadata to a struct whose whole point is staying compact would defeat the optimization.

3. **The instance is embedded inside another struct that already owns its own lifetime story.** Putting an `ArrayListUnmanaged` inside a struct lets the outer struct drive `deinit` cleanup with whichever allocator it already has on hand.

4. **You want the type to be trivially copyable as a value.** A `Bitset` can be assigned by value (with the documented caveat that the `heap` slice is a shallow alias - both copies will point at the same heap buffer). A managed type with an embedded allocator complicates the question of who owns what after a copy.

The standard library's `ArrayListUnmanaged`, `HashMapUnmanaged`, `BufMap` family all follow this rule - and as of Zig 0.14+ the unmanaged variants are the default; the managed wrappers were largely removed.

### 1.2 Allocator-owning (Managed)

The type stores its allocator at construction time and exposes a parameterless `deinit()`.

```zig
var arena = try NodeArenaOf(M).fromEntries(gpa, entries);
defer arena.deinit();   // no allocator argument needed
```

**Use it when:**

1. **There's exactly one (or a small handful of) long-lived instances.** The checker's `NodeArena` is created once per partition, holds one bulk `[]Node` allocation, and has one well-defined teardown site. The 16-byte cost is paid once; the duplication argument doesn't apply.

2. **The deinit site is far from the init site.** When a struct travels through several layers of code (returned from a builder, stashed in a context, eventually torn down by a `defer` near the top), threading the right allocator back to the deinit call is a footgun. Storing it eliminates the question "which allocator did this come from?" entirely.

3. **The deinit shape needs to be compatible with `defer` without a closure.** `defer arena.deinit()` is a one-liner; `defer arena.deinit(...)` requires the allocator to still be in scope and unambiguous.

4. **The owned resource is non-trivial enough that mismatched-allocator bugs would be expensive.** Freeing a `[]Node` with the wrong allocator is undefined behavior; storing the allocator removes the chance.

Standard-library precedent: `std.heap.ArenaAllocator` itself is managed (it stores the child allocator). `std.process.ArgIterator` on most platforms stores its allocator. `std.Build` stores allocators throughout.

### 1.3 The mental model: cost vs. risk

The choice reduces to a tradeoff between two costs:

| Cost                                   | Aware              | Owning           |
|----------------------------------------|--------------------|-------------------|
| Per-instance memory (allocator field)  | 0 bytes            | 16 bytes          |
| Risk of mismatched allocator at deinit | Caller's burden    | Eliminated        |
| Trivially copyable as a value          | Yes (with caveats) | No (semantic gotcha) |
| Deinit ergonomics                      | `x.deinit(alloc)`  | `x.deinit()`      |

Aware wins on memory and copy semantics. Owning wins on safety and ergonomics.  Population size and lifetime usually decide it: **many short-lived → aware, few long-lived → owning.**

---

## 2. Other Zig allocation patterns

The aware/owning axis is just one dimension. Several other patterns are in active use across the standard library and this codebase.

### 2.1 Arena allocation (lifetime-bundled)

`std.heap.ArenaAllocator` - many objects share one allocator, and freeing happens *only* in bulk (`deinit` or `reset(.retain_capacity)`). Per-object `free` is a no-op. Used pervasively here: `WorkerCtx.run` holds one arena per worker, resets it between partitions, throws it away on exit. The arena itself is allocator-owning (it stores its child allocator); the things it allocates for are usually allocator-aware (`Bitset`, hashmaps).

This is orthogonal to aware/owning: an arena is a *strategy*, while aware/owning is a *contract*.

**When arenas are wrong:** when individual objects need to outlive each other in unpredictable orders, or when objects hold non-memory resources (file handles, locks) that need explicit release. Arenas reclaim memory only.

### 2.2 Fixed-buffer / stack-backed allocators

`std.heap.FixedBufferAllocator` wraps a `[N]u8` and bump-allocates inside it.  No heap touch at all. Useful for parsers, formatters, comptime-bounded scratch - not used here, but the SBO pattern in `Bitset` is the manual cousin: a `[4]u64` inline buffer that dodges allocation entirely until you exceed it.

`FixedBufferAllocator` returns `error.OutOfMemory` when the buffer is exhausted, so callers still need to handle the fallible path; the win is no syscall, no heap fragmentation, and no thread-safety concerns.

### 2.3 Allocator-free types

Types that allocate nothing: pure value types or types backed by a caller-provided slice. `Bitset.data()` is a borrowed view over the active chunks; the project's `Operation`, `Event`, and `CheckResult` types are POD.  No `init`/`deinit` ceremony, no allocator threading. Always prefer this when feasible - the best allocation is no allocation.

### 2.4 Caller-provided buffer ("BYO storage")

The function takes a `[]T` from the caller and writes into it; ownership stays with the caller. `std.fmt.bufPrint`, `std.fs.File.read`. The function never allocates. Useful when the caller already has a buffer or wants tight control. `Bitset.dataMut()` exposes this shape on the read side.

The pattern composes well with fixed buffers: a caller can stack-allocate a `[1024]u8`, hand a slice of it to a library, and never touch the heap.

### 2.5 Bulk single-allocation containers (the "index arena")

`NodeArenaOf` is itself a pattern: instead of N small `Node` allocations linked by pointers, do **one** `alloc([]Node)` and link nodes by `u32` indices. Halves per-node footprint, makes free a single call, plays nicely with arena resets, and keeps everything cache-local. A `none_ref = std.math.maxInt(u32)` sentinel replaces `?u32` at zero cost - `u32` and `?u32` have different sizes in Zig (the latter carries a discriminator), so picking a value out of the valid range and treating it as "null" recovers the tighter packing while keeping the semantics.

This is closer to a layout pattern than an allocator pattern, but it exists *because* of how Zig encourages explicit allocator thinking. In a language that hides allocation, a pointer-linked list and an index-linked list look the same; in Zig the cost difference is right in the source.

### 2.6 Two-phase reservation (capacity hints + assume-capacity)

`ensureTotalCapacity` (fallible) followed by `appendAssumeCapacity` (infallible). Lets you front-load OOM into one reservation and then write a tight loop that can't fail. The codebase uses it where the final size is known up front - for example, building the per-partition entry array and op-id list inside `checkOperations`, and the event-renumbering loop in the model layer.

This is allocator-discipline rather than ownership, but it's a distinct idiom: it splits "can this fail?" from "will this run?", which composes better with `errdefer` than a sequence of fallible appends.

### 2.7 Scratch / two-allocator split

Pass two allocators: one for the long-lived output, one for the temporary working set. Useful whenever a function builds a result via a transient hashmap or buffer that is discarded before return - e.g., a function that renumbers events through an id-remap hashmap and then emits a new slice: the slice must outlive the call, the hashmap must not.

The shape is `fn foo(out_alloc: Allocator, scratch: Allocator, ...)`. Callers who have an arena handy pass it as `scratch`; callers who don't can pass the same allocator to both arguments and pay the per-allocation cost. The win is that callers who *do* have an arena get bulk reclamation of the scratch state for free, without the function having to know an arena exists.

### 2.8 Failing allocator / testing wrappers

`std.testing.allocator` (leak detector), `std.testing.FailingAllocator` (OOM injection at the Nth allocation), `std.heap.GeneralPurposeAllocator(.{})` with `.retain_metadata = true`. These wrap any other allocator and observe its calls. Their value evaporates if any code on the hot path bypasses the allocator the test handed in - for instance, a worker that hardcodes `std.heap.page_allocator` instead of taking the allocator from its parent will silently skip leak detection for everything it allocates. The discipline pairs with §3.1 below: every layer must accept an allocator parameter, even when the "obvious" choice would be a global.

`FailingAllocator` is the standard way to test OOM paths: configure it to fail on the Nth allocation, run the function, assert it returns `error.OutOfMemory` cleanly with no leaks. The `HashMapKvModel` OOM tests in this project use exactly this pattern to exercise the `errdefer` unwinding inside `step` and `clone`.

### 2.9 Concrete backing allocators (page / C / smp)

The patterns above are *strategies*; at the bottom of the stack sits a concrete allocator that actually returns memory. Three from the standard library show up in real code, each picked along a different axis:

- **`std.heap.page_allocator`** goes straight to `mmap` / `VirtualAlloc`.  No fragmentation tracking, allocations rounded up to page size (typically 4 KiB). Cheap for one big request, expensive per call. Almost always wrong for small, frequent requests - that's what `GeneralPurposeAllocator` is for - but it shines as the backing store for arenas: the arena amortizes the page-grain cost across many small client requests, so `page_allocator` is only called when the arena actually grows. The per-worker arenas in this codebase use it for exactly that reason.

- **`std.heap.c_allocator`** calls `malloc` / `free`. Available only when linking libc. Useful for FFI scenarios where Zig-allocated memory crosses into C code, since C code expects to call `free` on it. Outside FFI, no edge over `GeneralPurposeAllocator`.

- **`std.heap.smp_allocator`** (Zig 0.14+) - a thread-safe, general-purpose allocator suitable for multi-threaded code. The natural fit for "I have N worker threads and I want each to be free to allocate without coordinating": callers can pass `smp_allocator` directly without wrapping their own GPA in a thread-safe shim. In this project's parallel partition dispatch, the worker threads each hold their own arena, but the *backing* allocator for those arenas needs to be safe to call concurrently - that's the slot `smp_allocator` fills.

---

## 3. Anti-patterns and pitfalls

### 3.1 Hidden global allocators

Resist the urge to define `const gpa = std.heap.page_allocator;` at module scope and reach for it from every function. The whole point of Zig's explicit-allocator convention is that *the caller* decides allocation strategy: arena, GPA, fixed buffer, leak-detecting test wrapper. A hidden global silently defeats all of those.

The one defensible exception is a top-level `main` that picks an allocator and threads it down - but even that is one decision, not a pattern.

### 3.2 Mismatched allocators on aware types

`Bitset.deinit(other_alloc)` when `init` ran with `arena_alloc` is undefined behavior. The aware contract puts the burden on the caller, and Zig has no compile-time check that catches it. Mitigations:

- Pair `defer` with the allocation site so they're visually adjacent.
- Don't pass aware types across module boundaries without documenting the allocator contract.
- When in doubt, prefer owning: the cost is 16 bytes, the benefit is the whole class of bug going away.

### 3.3 Shallow copies of aware types with heap state

`Bitset` is a value type; `var b2 = b1;` shares the `heap` slice. Freeing either invalidates the other. The codebase deliberately holds bitsets through pointers - for example, iterating with `for (entries.items) |*e|` rather than `|e|` - to avoid making an aliased copy on every loop step.  `Bitset`'s doc comment flags the hazard; without that comment, the bug class is invisible at the call site.

### 3.4 Per-element deinit under an arena

If an arena is going to reclaim everything in one call, walking each element and `free`ing it first is pure waste - and worse, it's misleading documentation: a reader sees the loop and assumes the elements have non-trivial cleanup, which forces them to reason about ordering, partial failure, and reentrancy. The fix is either to delete the loop and document the arena assumption at the allocation site, or - if the type genuinely might hold non-memory resources like file handles - gate the loop behind a comptime marker on the type so the dead version is removed in builds where the elements really are arena-safe.

### 3.5 Storing an allocator inside a thread-shared struct

If multiple threads share a struct that owns its allocator, and the allocator isn't thread-safe (`GeneralPurposeAllocator(.{})` defaults are single-threaded), every allocation is a race. Either pick a thread-safe allocator (`smp_allocator`, `c_allocator`) or give each thread its own arena, as `WorkerCtx` does.

### 3.6 Returning slices from a function whose allocator goes out of scope

The classic: a function creates a `FixedBufferAllocator` on its stack frame and returns a slice allocated from it. The frame unwinds, the buffer is gone, the slice is dangling. Either return ownership with `toOwnedSlice` (backed by a heap allocator that outlives the call), or have the caller pass in the buffer.

---

## 4. Quick decision table

| Pattern                       | When it fits                                                                 |
|-------------------------------|------------------------------------------------------------------------------|
| Allocator-aware (Unmanaged)   | Many small instances; SBO-sensitive; embedded in a parent that drives deinit |
| Allocator-owning (Managed)    | Few long-lived owners; deinit site distant from init; mismatch is dangerous  |
| Arena                         | Many small lifetimes coterminous with a single phase                         |
| Fixed-buffer / SBO            | Bounded size known at comptime or by domain heuristic                        |
| Allocator-free                | POD / borrowed views                                                         |
| BYO buffer                    | Caller already has storage; library doesn't need policy                      |
| Index-arena (one bulk alloc)  | Graphs / linked structures of homogeneous nodes                              |
| Two-phase reserve + assume    | Hot-loop appends where OOM should be front-loaded                            |
| Scratch + output split        | Function with transient working set + persistent result                      |
| Failing/leak-detecting wrapper| Tests; OOM-path coverage                                                     |
| Page allocator                | Backing store for arenas; rare direct use                                    |
| C allocator                   | FFI boundaries with C code                                                   |
| Smp allocator                 | Multi-threaded code without a per-thread arena                               |

---

## 5. How the patterns compose in `porcupine-zig`

A single `checkOperations` call exercises most of the table:

1. **Caller's allocator** (any) is handed in for the entry array and partition slices - the long-lived outputs that survive the call.
2. **Per-worker arena** wraps `page_allocator` and serves every DFS-internal allocation: bitsets, hashmap nodes, cloned states, linearization records.
3. **Allocator-aware `Bitset`** is allocated inside the arena, cloned thousands of times during DFS, and never individually freed - the arena reclaims it all on `reset` between partitions.
4. **Allocator-owning `NodeArena`** holds the doubly-linked-list of call/return nodes for the whole partition, allocated from the caller's allocator (one bulk alloc) and freed once at the end.
5. **Two-phase reserve + assume** in `checkOperations` and `dedupe` pre-sizes the entry array and op-id list so the inner loop can't OOM.
6. **Index arena** layout in `NodeArenaOf` keeps the linked-list traversal cache-friendly with `u32` links and a `none_ref` sentinel.
7. **Allocator-free types** (`Operation`, `Event`, `CheckResult`) flow through the API boundary without any allocation ceremony.

Each pattern is doing the job it's best at, and the boundaries between them match natural lifetime boundaries in the algorithm. That's the goal: pattern selection should fall out of "what lives how long?" rather than being imposed top-down.

---

## 6. Further reading

- Zig standard library source: `lib/std/heap.zig` and `lib/std/mem/Allocator.zig` for the canonical implementations of every allocator mentioned here. The `Allocator` interface in particular is worth reading end-to-end - it is short, and seeing how the vtable is laid out makes the "16 bytes" cost in §1 concrete.
- The Zig language reference's "Memory" chapter, for the rationale behind explicit-allocator passing as a language design choice.
