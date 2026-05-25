+++
title = "Building a custom Bitset - A study in access-pattern-driven data structure choice"
date = 2026-05-25
description = "Shows that data-structure choice must be driven by access pattern, not category: a custom bitset with SBO and virtual-mutation hashing is strongly justified for a hot-path DFS that repeatedly clones, hashes, and probes without committing, while the standard-library DynamicBitSet is the right tool for a simple one-shot debug assertion"
template = "page.html"

[taxonomies]
tags = ["zig", "design-patterns", "data structure"]
[extra]
+++

# Building a custom Bitset - A study in access-pattern-driven data structure choice

I was porting `porcupine-rust` to Zig - the core of the implementation is a Bitset data structure, which is used in the implementation of a cache (see below for the details of why we need the cache).

[`porcupine-zig`](https://github.com/debasishg/porcupine-zig) carries two bitset implementations and uses them in two different places:

| Site | Type | Mode | Purpose |
| --- | --- | --- | --- |
| `src/bitset.zig` (custom) | `Bitset` | Release + Debug | Inner DFS in `checker.zig` - cache-probed, cloned per miss, hashed every step |
| `assertPartitionIndependent` (std) | `std.DynamicBitSet` | Debug only | One-shot validation that a partitioning is disjoint, complete, and in-bounds |

That is not duplication or inconsistency. It is the most basic discipline in performance engineering: **pick the data structure that fits the access pattern of the call site, not the one that fits "bitset" as a category.** The two sites have effectively zero overlap in what they ask of a bitset, and collapsing them onto a single type would either pessimise the hot path or over-fit the cold one.

This document walks through both call sites, derives the requirements from the algorithm rather than from intuition, and shows why the resulting choices are strongly justified - not stylistic. A wrapper around `std.DynamicBitSetUnmanaged` could in principle furnish part of the hot-path surface (its allocator-on-call-site shape, `clone`, and `eql` are already there), but it would still lack SBO and the virtual-mutation hash/equality protocol; closing that gap is most of the work, which is why the project carries a project-local type. For a complete explanation we need to start with a basic understanding of the linearizability-checker algorithm.

---

## 1. The hot path: linearizability DFS in `checker.zig`

### 1.1 What the algorithm actually does

The Wing & Gong / Lowe linearizability check explored in `checkSingle` is a depth-first search over the lattice of partial linearizations of an operation history. At each frame the algorithm:

1. Picks an unlinearized call `op_id` from a doubly-linked candidate list.
2. Steps the user-supplied model: `step(state, input, output) -> ?State`.
3. If the step succeeds, asks the cache: *have we already explored the `(linearized ∪ {op_id}, new_state)` pair from any other ordering?*
4. On cache miss: commit. Push the old state onto a backtrack stack, set the bit, store the new `(bitset, state)` in the cache, descend.
5. On cache hit: prune. Backtrack and try the next candidate.

The cache is the entire point of the algorithm. Without it the search is super-exponential in history length; with it the redundant subtrees collapse and the search is tractable for histories of a few hundred operations. The cache key is the pair `(set of linearized op ids, model state)`, with the bitset standing in for the set.

This puts the bitset squarely in the DFS inner loop - touched multiple times per frame:

- **Hashed** every probe (`linearized.hashWithBit(op_id)`).
- **Compared for equality** every probe whose hash collides (`bs.eqlWithBit(...)`, inside `cacheContainsWithBit`).
- **Cloned** every cache miss, before the bit is actually set on the clone (`linearized.clone(arena_alloc)`).
- **Mutated** on every step forward and every backtrack (`linearized.set(op_id)`).

On workloads shaped like the Rust/Go etcd and KV fixtures, cache hits are expected to dominate mid-search (the upstream Go and Rust ports report high hit rates on these histories; a measured Zig hit-rate table belongs here once the corresponding fixtures land in `porcupine-zig`). That distribution matters: any cost that lands on a *probe* is paid many times more often than any cost that lands on a *commit*.

### 1.2 Requirements derived from the access pattern

From the algorithm we get a precise contract for the bitset:

1. **Cheap clone in the common size range.** Histories the project actually targets are 50-256 ops; cloning must not allocate in this range, because misses still happen at every level of the search tree.
2. **`hash` that admits an O(1) update under single-bit mutation.** Every probe asks "what would the hash be if I set bit `p`?". If answering that requires materializing the mutated bitset, the probe pays the cost of the commit even when it's a hit.
3. **`eq` that can be evaluated against a virtually-mutated `self`.** Symmetric to (2): hash collisions need to be resolved without realising the mutation either.
4. **Allocator passed at the call site, not stored.** The DFS uses a per-worker arena (constructed inside `checkSingle`) and tears the whole partition down with one `arena.deinit()`. A bitset that hides an allocator pointer inside its struct fights that lifetime model.
5. **Stable inline layout, predictable indexing.** Hot inner loops want the chunk pointer in a register and bounds checks elided in release.

The standard library's `std.DynamicBitSet` (and `DynamicBitSetUnmanaged`) satisfy *none* of (1)-(3) and only partially (4)-(5). `DynamicBitSetUnmanaged` already takes the allocator at the call site and exposes `clone`/`eql`, so the lifetime and basic-comparison story can be borrowed; but SBO and the virtual-mutation hash/equality protocol have no counterpart in the standard library, and adding them is most of the work of writing a project-local type.

### 1.3 How the custom `Bitset` answers each requirement

#### 1.3.1 Small-buffer optimization (SBO)

`Bitset` stores an `inline_buf: [4]u64` field directly in the struct - 256 bits, aligned, contiguous, no indirection. The `heap` slice is consulted only when the bit count exceeds 256, which on the project's workloads happens for exactly one fixture (and even there only for the un-partitioned variant).

What this buys, concretely:

- `clone` is a struct copy plus an *optional* `allocator.dupe` that almost never runs. In the inline case it is one `[4]u64` move + a couple of scalar fields. No allocator vtable dispatch, no free-list traversal, no fragmentation.
- The active storage tends to live in 1-4 consecutive cache lines, often prefetched as part of the surrounding `CacheEntry` row, rather than being a heap pointer that may sit on a cold line.
- The arena allocator never sees most bitsets at all. On the partitions that fit inline the arena reset at end-of-partition is reclaiming state-clones and hash-table buckets, not bitset bodies.

The 256-bit ceiling is not arbitrary: 256 was chosen because (a) it covers the histories `porcupine-zig` is benchmarked against (etcd ~170 ops → 3 chunks; KV per-partition ≤ 50 ops → 1 chunk, per the workload notes at the top of `bitset.zig`), and (b) `n / 64` is a single shift in code that indexes chunks, with `inline_cap = 4` keeping the struct under one cache line on common architectures (`Bitset` is `4*8 + 16 + 8 = 56` bytes, plus padding).

`std.DynamicBitSet` has no small-buffer optimization; for any nonzero runtime size its mask storage is heap-backed. The type is designed for arbitrary runtime sizes where the average case is large enough to not care about the allocation. For the DFS this is the wrong default: the average case is *small enough that the allocation would dominate the operation cost*.

(One Zig-specific ownership caveat worth flagging while we're on the SBO discussion: `Bitset` contains a slice field for the heap-spilled case, so a plain struct assignment of a heap-spilled bitset is a shallow copy that aliases the heap chunks. The source comments require callers that need a deep copy to go through `clone(allocator)`. The DFS already does, but the discipline is part of what makes the explicit allocator-per-call shape work.)

#### 1.3.2 Algebraically updateable hash

The `hash` formula is

```zig
h = popcnt(bits)            // initial value
for each chunk c: h ^= c    // fold chunks in
```

This looks pedestrian, but the *chunk fold* has a property no off-the-shelf hash (FNV, xxHash, FxHash, SipHash) gives you: it is *XOR-decomposable in the chunk basis*. Setting a single bit `p` mutates exactly one chunk word, from `old_word` to `new_word = old_word | (1 << minor)`, and the chunk fold's contribution shifts by exactly `(old_word ^ new_word)`. The popcnt seed shifts independently - by `popcnt XOR (popcnt + 1)`, which is `1` when popcnt is even and a longer carry chain when odd.

`hashWithBit` returns a closed-form *lookup key* that folds in the chunk delta and the even-popcnt shape of the popcnt delta:

```zig
hashWithBit(p) = self.hash() ^ old_word ^ new_word ^ 1
```

Two technical notes on what this actually is and is not:

- **It is a self-consistent cache key, not the post-set `hash()`.** When popcnt is odd, the true popcnt delta is wider than `1` (e.g., `3 → 4` differs by `0b111`), so `hashWithBit(p)` does not in general equal `hash()` of the bitset *after* `set(p)`. What it does guarantee is *self-consistency*: any two probes that would land on the same `(linearized + bit, state)` cache entry have equal predecessor popcnts and equal post-set chunks, so they hash to the same value under this formula. Since insertion and lookup both go through `hashWithBit`, the cache's hit rate depends only on self-consistency - and self-consistency holds for every `popcnt` parity. (The function name and the unit test in `bitset.zig` both suggest the stronger property; only the weaker one is load-bearing for the algorithm.)
- **It is `O(chunks)`, not `O(1)`, and currently makes two passes over the active chunks.** `self.hash()` first calls `popcnt()`, which walks `self.data()` to count set bits, and then loops over `self.data()` again to fold the chunks. `hashWithBit` adds the three-XOR adjustment on top. The structural saving in the hot path therefore comes from `hashWithBit` not having to *allocate, clone, or mutate*, not from skipping the chunk scan. Fusing the popcount and the XOR fold into a single loop is a trivial micro-optimisation; a more substantive version that maintained `popcnt` and the running chunk-XOR incrementally on `set`/`clear` - and used the proper `popcnt XOR (popcnt + 1)` delta - would deliver true `O(1)` hashing and post-set equivalence as a free side-effect. The current code chose simpler state.

The architectural point still stands: chunk-level XOR-decomposition is what makes *any* incremental scheme possible. Swapping in FxHash, xxHash, or SipHash would force every probe to materialize the post-set bitset and rehash from scratch - which is exactly what the Go original does. The hash, the SBO, and the deferred-clone protocol form one design; you can't extract any single piece and keep the others.

The hash table the cache lives in (an `std.HashMapUnmanaged` in the checker's `CacheEntry` map) is keyed by a precomputed `u64` produced by `hashWithBit`, and the table uses the identity context `U64IdentityCtx` to consume that `u64` directly as the bucket hash. The library's default context would re-hash the `u64`, layering an unnecessary mixing pass over a value that already came out of a hash-shaped formula. Note the precise claim: the stored key is the `hashWithBit` value, not necessarily `Bitset.hash()` of the post-set bitset (on odd predecessor popcounts those differ, per the parity caveat above). The identity context is correct because the cache key is precomputed, not because `Bitset.hash()` is universally the table key.

#### 1.3.3 Deferred-clone cache probing

Equipped with `hashWithBit`, the matching `eqlWithBit` closes the loop. Equality between "self with bit `p` set" and an existing cache entry is checked by walking the chunk arrays in lockstep and OR-ing the relevant word on the fly:

```zig
const set_mask = @as(u64, 1) << ix.minor;
for (a, b, 0..) |x, y, i| {
    const adj = if (i == ix.major) x | set_mask else x;
    if (adj != y) return false;
}
```

The hot-path call site is `cacheContainsWithBit`:

```zig
const h = linearized.hashWithBit(op_id);
if (!cacheContainsWithBit(M, &cache, model, h, &linearized, op_id, &ns)) {
    // miss: now we pay for clone + set + cache insert
    var new_linearized = try linearized.clone(arena_alloc);
    new_linearized.set(op_id);
    ...
}
```

The structural consequence: on every probe that hits the cache, the algorithm never allocates a bitset, never mutates a bitset, and never copies its chunks into a temporary. It hashes the predecessor (two chunk passes today, as discussed above), adjusts with three XORs, walks one hash bucket's chain (`eqlWithBit` synthesizing the post-set word inline), and prunes. What is *deferred* is the clone and the bit-set: for inline bitsets that's a `[4]u64` struct copy plus a single OR; for heap-spilled bitsets it's an `allocator.dupe` plus a memcpy plus an OR. The clone happens only on the cache-miss branch, where the bitset is about to be stored anyway and the allocation amortises against the work of the new DFS subtree. The size of the win on a probe-heavy workload is therefore proportional to the probe-to-commit ratio, which is the property the algorithm-level numbers from the Go and Rust ports describe.

This optimisation is present in the Rust port (where it is named `hash_with_bit`) and absent in the Go original, which clones-then-hashes on every probe. The performance gap on long histories is meaningful for that exact reason.

`std.DynamicBitSet` exposes no `hash` method at all, let alone an algebraically-updateable one. Bolting it on would require deciding the hash formula, which immediately re-introduces the design choice that drives the custom type.

#### 1.3.4 Allocator on the call site

The bitset stores no allocator handle. Every allocating method takes one explicitly:

```zig
pub fn init(allocator: std.mem.Allocator, n: usize)         !Bitset
pub fn deinit(self: *Bitset, allocator: std.mem.Allocator)  void
pub fn clone(self: *const Bitset, allocator: std.mem.Allocator) !Bitset
```

This matches the DFS's lifetime model. `checkSingle` constructs an `std.heap.ArenaAllocator` per partition (per worker thread when partitions parallelise) and feeds *every* bitset, *every* state clone, *every* hash-map bucket through that arena. At the end of the partition the arena is reset in one operation; thousands of bitsets and tens of thousands of hash buckets are reclaimed together with no per-object teardown.

`std.DynamicBitSet` (the managed variant) embeds an `Allocator` pointer in the struct. Storing thousands of these in the cache would cost one extra pointer per entry for a value that is constant across all entries.  `DynamicBitSetUnmanaged` solves that, but still leaves you without SBO, without `hash`/`hashWithBit`, and without `eqlWithBit`. The unmanaged form is closer to the right shape, but at that point you would be writing the custom type around it.

#### 1.3.5 Small, inlineable indexing primitives

`isSet`, `set`, `clear`, and `index` are declared `inline fn`, and the chunk-pointer selection is a single `chunks > inline_cap` branch against a field that is constant for the lifetime of a given bitset. The `index` helper splits `pos` into `(major, minor)` with `pos / 64` and `pos % 64`; since 64 is a power of two and `pos` is a `usize`, this is the canonical shape the compiler can lower to a shift and an `AND`. In `-Doptimize=ReleaseFast`, the `debug.assert` bounds checks fall away and the inline-vs-heap branch is predictable enough to be cheap (and in many call sites, hoistable). The point is not a specific assembly sequence - that would require a disassembly listing this post does not include - but that the primitives are small and shaped to be optimised, which matters because they run once per node visited in the DFS.

### 1.4 Summary table for the hot path

| Requirement | Custom `Bitset` | `std.DynamicBitSet` |
| --- | --- | --- |
| No allocation for histories ≤ 256 ops | Yes (SBO) | No |
| `hash()` defined | Yes, XOR-decomposable | No |
| Probe a "virtually-mutated" key | Yes (`hashWithBit`/`eqlWithBit`) | Not expressible without re-implementing the type |
| Allocator passed per call (arena-friendly) | Yes | Managed: no. Unmanaged: yes (but still missing the above). |
| Wire-compatible hash with Go/Rust ports | `hash()` formula matches; `hashWithBit` is a self-consistent cache key, not always equal to the post-set `hash()` (parity caveat) | N/A |

---

## 2. The cold path: `assertPartitionIndependent`

### 2.1 What it is for

`assertPartitionIndependent` validates a debug-mode invariant: when a model declares a `partition` hook, the returned list of partitions must be a *partition* of `[0, N)` in the mathematical sense. Specifically:

1. **Disjoint** - no index appears in two partitions.
2. **Complete** - every index in `[0, N)` appears at least once.
3. **In-bounds** - no index is `≥ N`.

A bug in a user-supplied `partitionEvents` could otherwise cause the DFS to silently skip events (violating soundness) or read past the history (a panic, but not a useful error).

### 2.2 Access pattern

The function runs at most once per `checkOperations` / `checkEvents` invocation that actually receives partitions from the user-supplied `partition` / `partitionEvents` hook, and it compiles to a no-op outside Debug mode. When it does run, it performs:

- One allocation of a bit array of length `N`.
- One pass over all partitions; for each index, one bounds check, one `isSet` test, one `set`, one increment.
- One terminal equality check on the count.
- One deinit.

That is it. No clone, no hash, no equality, no nested probes, no per-step mutation across DFS frames, no allocator threading. The function exists to fail loudly on invalid partitionings, not to be fast.

### 2.3 Requirements derived from the access pattern

Reading the algorithm rather than the type:

1. **Allocate `N` bits, set each, query each, count.** Nothing else.
2. **Lifetime is one stack frame.** No need for arena routing or for the bitset to outlive the function.
3. **Performance is irrelevant.** This path is removed in release builds.
4. **Code clarity is paramount.** This is an assertion: future readers should be able to see at a glance that the three invariants are enforced.

### 2.4 Why `std.DynamicBitSet` is the *right* choice here

- **It is in the standard library.** No reader needs to chase project-local semantics to understand what `set`/`isSet`/`initEmpty` mean. The standard library type is the lingua franca; using it is a hint that nothing special is going on.
- **It exposes standard, familiar operations and no cache-specific protocol.** `std.DynamicBitSet` actually has a larger surface than this site uses (range ops, set algebra, iterators, clone), but crucially it has *no* `hashWithBit`, no virtual-mutation invariants, no "bit must be currently clear" precondition to violate, no SBO branch to reason about. Nothing about its API hints at the cache-probe protocol. The shared vocabulary is the win, not minimalism.
- **One allocation, one deinit, simple lifetime.** ~`N/8` bytes via the caller's allocator, released by a single `defer bits.deinit()`. SBO would be unused: the function holds exactly one bitset, ever, so an inline buffer would not save a heap allocation per clone - there are no clones.
- **The three invariants compose elegantly with bit semantics.** Disjoint = `assert(!bits.isSet(idx))` before `set`; in-bounds = `assert(idx < expected_total)`; complete = `count == expected_total`. The bitset is doing exactly the work a set membership test should do, with a tight, obvious mapping from invariant to line of code.

### 2.5 Why the custom `Bitset` would be *wrong* here

This is the more interesting direction. The custom `Bitset` would mechanically work - `init`/`set`/`isSet`/`deinit` are present and correct.  But choosing it would be a category error:

- **The custom type carries preconditions tuned to the cache protocol.** `hashWithBit` requires the probed bit to be currently clear; the `debug.assert`s inside `hashWithBit` and `eqlWithBit` enforce that. None of this matters here, but importing the type into a debug-only path creates the implicit suggestion that the whole protocol is in scope.  Future maintainers reading the assertion code would have to figure out why.
- **SBO is dead code on this path.** The function holds exactly one bitset of size `N`. The inline buffer either adds 32 bytes of stack that go unused (when `N > 256`) or replaces a single `~N/8`-byte allocation that nobody cares about (when `N ≤ 256`). The optimisation is invisible at the call site and pays nothing.
- **The custom hash is unused machinery.** A bitset whose `hash`/`eql` facilities are never called is a worse fit than one that does not have them; the API surface is wider than the requirement, and unused API surface attracts misuse.
- **Wire compatibility with Go/Rust does not apply.** This is a Zig-only invariant check; there is no peer port doing the same check whose hash output we need to match.

The custom type was designed for a specific, demanding access pattern.  Using it where that pattern doesn't exist would be like reaching for a B-tree to store five elements: it works, but it advertises a complexity class the data does not have.

---

## 3. When to revisit this decision

A choice this load-bearing should come with explicit triggers for re-evaluation. The custom `Bitset` is justified *given the current shape of the algorithm and workload*; if either shifts, the cost/benefit calculation shifts with it. Concrete triggers:

1. **The DFS stops doing virtual-mutation probes.** If `cacheContainsWithBit` is removed - say, the cache key changes shape, or the algorithm switches to a different pruning strategy - then `hashWithBit`/`eqlWithBit` lose their callers, and with them the constraint that pins the hash to a specific XOR-decomposable formula.  At that point `std.DynamicBitSetUnmanaged` plus a thin `hash()` helper covers the remaining requirements. The custom type is no longer load-bearing; deleting it would be a strict simplification.
2. **Typical histories outgrow the inline buffer.** SBO is calibrated for `n_ops ≤ 256`. If the project's target workloads shift to consistently larger histories - e.g., new fixtures in the few-thousand-ops range - the inline buffer becomes dead weight rather than a fast path, and the case for SBO weakens. The right response there is probably to retune `inline_cap` rather than abandon the custom type, but the question is worth asking.
3. **The cache hit rate collapses.** The deferred-clone optimisation pays off in proportion to the probe-to-commit ratio. If a future model shape produces hit rates closer to 50/50, the savings from `hashWithBit` shrink (though SBO continues to pay on the misses).  This is a tuning signal, not a redesign signal - but it's a place to measure rather than assume.
4. **Zig's standard library grows what we built.** If a future `std` ships a small-buffer-optimized bitset or a bitset with an algebraically-updateable hash, the custom type becomes a candidate for deletion. Until then, neither feature is in `std`, and external dependencies are not worth introducing for a 330-line file.
5. **The arena lifetime model goes away.** The "allocator on the call site" discipline matters because every bitset in the cache is owned by a per-partition arena. If the checker moves to a different memory strategy - long-lived caches across partitions, persistent search trees, anything - the unmanaged-allocator argument weakens, and a managed type becomes more reasonable.

Symmetrically, the std-bitset choice in `assertPartitionIndependent` is trivially robust: it would only need to change if the function were promoted out of debug-only mode, at which point the access-pattern analysis would have to be redone from scratch.

## 4. The general principle

The two call sites in this project illustrate a discipline that is easy to state and consistently hard to follow: **data structure choice should be driven by access pattern, not by category.** "Bitset" is not a specification. The questions that produce a specification look like:

- How often is the structure mutated relative to how often it is read?
- Is it cloned? At what rate? With what allocator?
- Is it hashed or compared? Is the hash on the critical path?
- Does the algorithm ask "what would the structure look like if I applied operation `X`?" without committing to `X`? (If so, the structure may need primitives for *virtual* mutation.)
- What is the lifetime - one stack frame, one DFS subtree, one partition, one process?
- What hardware-level effects matter - cache lines, branch prediction, allocator contention?

For the DFS in `checker.zig`, the answers point to a small custom type with SBO, an algebraically-updateable hash, deferred-clone probe primitives, and an arena-friendly allocator interface; given the constraints, that is the simplest and most controllable implementation in this codebase. For `assertPartitionIndependent`, the answers point to the simplest bit-of-storage primitive in the standard library.

These are not opposing aesthetics; they are the same aesthetic - *fit the tool to the work* - applied to different work. The fact that one site needs a finely-engineered custom type does not mean every other bitset use in the project should reach for it. The fact that the standard library has a perfectly adequate `DynamicBitSet` does not mean the hot path should give up SBO, an updateable hash, and virtual-mutation probing to use it - those primitives, not a generic "slowdown" headline, are the load-bearing pieces, and the right way to size their value is a benchmark table comparing the current code against (a) a custom `Bitset` without virtual-mutation probing and (b) `std.DynamicBitSetUnmanaged` plus clone-then-hash.

The two bitsets in this codebase exist because the two call sites have disjoint requirements. Picking one tool for both would either pessimise the algorithm that defines the project's performance, or import a specialised type's invariants into a debug-only assertion that has no use for them. Keeping them separate is the boring, correct answer.
