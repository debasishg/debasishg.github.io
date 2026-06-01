+++
title = "Set-Associative Caches: Trading Global Optimality for Predictable Speed"
date = 2026-06-01
description = "How set-associative caches trade global eviction freedom for bounded, cache-line-friendly lookups, and how TigerBeetle's CacheMap and a SIMD search optimization (PR #3200) put that trade-off to work."
template = "page.html"

[taxonomies]
tags = ["cache", "set-associative-cache", "data structure", "optimization"]
[extra]
+++

# Set-Associative Caches: Trading Global Optimality for Predictable Speed

Caches are everywhere in modern systems: CPU L1/L2 caches, database buffer caches, storage-engine object caches, and application-level LRU maps. They do not all make the same trade-offs. A set-associative cache sits at a particular point in the design space: it gives up global replacement freedom in exchange for bounded probing, compact layout, and predictable cache-line-friendly access.

This post walks through what that means, why TigerBeetle uses a set-associative tier inside its `CacheMap`, and how a focused optimization to its SIMD search path improved instruction-level parallelism.

---

## 1. What Is a Set-Associative Cache?

A **set-associative cache** is a caching strategy borrowed from CPU architecture. It sits between two extremes:

* A **fully associative cache** lets an item live anywhere in the cache.
* A **direct-mapped cache** maps each item to exactly one slot.
* A **set-associative cache** divides the cache into **sets**, each containing a small fixed number of **ways** or **slots**. That fixed number is the cache's **associativity**.

The hash of a key determines which set the item belongs to. Within that set, the item may occupy any way. When the set is full, a local replacement policy chooses a way to evict.

For example, a 4-way set-associative cache with 256 sets has 1024 total slots.  Every key hashes to one of the 256 sets and can occupy any of that set's 4 slots. When the set overflows, one of those 4 entries is evicted.

### Benefits

* **Predictable lookup cost.** A lookup checks a small fixed number of slots.  There are no long collision chains or unbounded probe tails.
* **Fewer conflict misses than direct mapping.** Multiple keys that hash to the same set can still coexist.
* **Compact memory layout.** Sets and their metadata can be packed tightly, which matters in low-level hot paths.

### Trade-Off

Unlike a fully associative cache, an item cannot live anywhere. If one set is hot and overflows, the cache evicts aggressively from that set even if colder entries exist elsewhere.

---

## 2. How It Differs from an Ordinary Software Cache

A typical general-purpose software cache, such as a `HashMap` plus a global LRU list, behaves differently:

* Items can be placed anywhere in the structure.
* Lookups are usually average-case `O(1)` through the hash table.
* Eviction is global: the victim is chosen from the whole cache, not just one small set.

A set-associative cache, by contrast:

* Restricts a key's possible locations to one set.
* Evicts locally from that set.
* Usually has a fixed capacity and a small fixed associativity.

This gives up some hit-ratio flexibility. You may evict a warm item from an overfull set while colder items sit untouched elsewhere. The payoff is a simple, bounded, CPU-friendly memory access pattern.

---

## 3. Memory Layout: Hash Map vs. Set-Associative Cache

### A Pointer-Heavy Chained Hash Map

One classic hash-table layout uses separate chaining:

```text
[Bucket 0] -> pointer -> [Node(key, value, next)]
[Bucket 1] -> pointer -> [Node(key, value, next)]
[Bucket 2] -> pointer -> [Node(key, value, next)]
...
```

In that layout:

* Nodes are scattered across the heap.
* Traversal requires repeated pointer dereferences.
* Spatial locality is poor, because adjacent buckets do not imply adjacent keys or values.

An illustrative lookup looks like this:

```text
index = hash(key) % bucket_count
node = buckets[index]

while node != null:
    if node.key == key:
        return node.value
    node = node.next

return null
```

This is deliberately pseudocode. Labeling it as Rust would be misleading:
idiomatic Rust code would call `map.get(key)`, and Rust's [`std::collections::HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) is an open-addressed table rather than a chained node table. The sketch is here only to show the memory-access pattern of a pointer-heavy hash table.

Modern hash maps are often much more cache-conscious than this. SwissTable-like metadata groups, Robin Hood probing, and contiguous slot arrays all improve locality. So the comparison is not "set-associative cache versus every hash map." It is "bounded set scan versus a layout that may chase pointers or probe an unbounded number of locations."

### Set-Associative Cache

A 4-way set-associative cache with 256 sets can be laid out like this:

```text
Set 0:   [Slot0][Slot1][Slot2][Slot3]
Set 1:   [Slot0][Slot1][Slot2][Slot3]
Set 2:   [Slot0][Slot1][Slot2][Slot3]
...
Set 255: [Slot0][Slot1][Slot2][Slot3]
```

The lookup shape is:

```text
set_index = hash(key) % set_count
set = sets[set_index]

for slot in set:
    if slot.is_present and slot.key == key:
        return slot.value

return null
```

Again, this is pseudocode. The important property is not the surface syntax but the access pattern: the loop touches one small contiguous region, usually one cache line or a small handful of cache lines.

---

## 4. Why Set-Associative Caches Are CPU-Cache Friendly

Four properties combine to make this layout fast:

1. **Contiguous memory layout.** Slots or metadata for a set live in a compact array. When the CPU loads a 64-byte cache line, it can bring in several pieces of relevant metadata at once.

2. **Bounded probing.** Probe length is a fixed constant: 4, 8, 16, or another small associativity. The pipeline does not have to chase an unpredictable chain.

3. **Fewer pointer indirections.** Sequential scans over compact arrays are friendlier to caches and prefetchers than scattered linked nodes.

4. **Predictable branches.** A short fixed loop with simple comparisons tends to be easier for the branch predictor than data-dependent pointer chasing.

### Side by Side

| Aspect                | Pointer-heavy chained hash map | Set-associative cache              |
| --------------------- | ------------------------------ | ---------------------------------- |
| Memory layout         | Scattered, pointer-heavy       | Contiguous, array-based            |
| Lookup locality       | Often poor                     | Excellent within one set           |
| Cache-line efficiency | Low for chain traversal        | Dense metadata and bounded scan    |
| Probe length          | Variable                       | Fixed by associativity             |
| Eviction              | External/global if used as a cache | Local to one set                |

### Cache-Line Behavior

```text
Chained hash map:        Set-associative cache:
Load bucket pointer      Load compact set metadata
Jump to node             Scan a bounded set of ways
Maybe jump again         Compare candidate slots
```

The set-associative layout trades global freedom for a probing pattern the CPU can handle well.

---

## 5. If Set-Associative Caches Are So Good, Why Use Ordinary Caches?

Set-associative caches win on raw lookup predictability and hot-path locality.  General-purpose caches win on flexibility, richer policies, and often better hit ratios for messy application workloads.

### When Set-Associative Caches Shine

* Hot-path, low-latency systems: CPU caches, databases, storage engines, and similar systems code.
* Workloads where bounded worst-case lookup work matters.
* Fixed memory budgets.
* Compact metadata and predictable memory access.

This is why CPU designers use set associativity for L1/L2 caches: hardware cannot tolerate pointer chasing or unbounded probe lengths in the common path.

### Their Weaknesses

1. **Local eviction can be suboptimal.** A hot set may evict useful entries while cold sets remain underused. A global policy may achieve a better hit ratio at the same capacity.

2. **Fixed associativity can thrash.** If more than `N` hot keys map to an `N`-way set, the set can churn even when the total cache has spare-looking capacity elsewhere.

3. **Replacement policies are usually simple.** LRU-within-set, CLOCK-like policies, and random replacement are common. Feature-rich application caches may offer TTLs, weights, admission policies, async refresh, and other workload-specific behavior.

4. **Capacity is usually fixed.** General-purpose caches often grow, shrink, or resize under memory pressure. Set-associative caches are normally sized up front.

### Why Ordinary Caches Still Dominate Application Code

Application caches often care more about policy than nanoseconds. They need dynamic sizing, TTLs, weighted entries, observability, workload-specific admission rules, and global eviction behavior. A set-associative cache is a sharp tool for a hot path, not a universal replacement for a full cache library.

### Analogy

* A CPU L1 cache is fast, compact, and set-associative, but not globally fair.
* An OS page cache uses broader replacement machinery because it is optimizing a larger and messier working set.

Application code usually wants the page-cache model. Systems code on a hot path often wants the L1-cache model.

---

## 6. Case Study: TigerBeetle

TigerBeetle is a distributed financial accounting database where correctness, latency, and predictability matter. Its `CacheMap` is a useful real-world example because it combines a fast set-associative tier with a bounded `std.HashMapUnmanaged` stash, and because the set-associative tier has received careful low-level optimization.

### 6.1 `CacheMap`: A Two-Tier In-Memory Cache

`CacheMap` lives in TigerBeetle's LSM lookup path. The [`cache_map.zig`](https://github.com/tigerbeetle/tigerbeetle/blob/main/src/lsm/cache_map.zig) source describes it as a hybrid between `SetAssociativeCache` and a `HashMap` stash:

1. **`SetAssociativeCache` fast tier.** A compact structure optimized for predictable lookup and insertion. Bounded associativity gives constant-time probing and local replacement.

2. **`HashMap` stash.** A preallocated `std.HashMapUnmanaged` used as an auxiliary tier. It catches values evicted from the set-associative tier and helps guarantee that prefetched values remain in memory during their respective commit.

This is reminiscent of the "stash" idea used with cuckoo hashing: a small auxiliary structure catches cases the primary structure cannot cheaply retain.  The setting is different here, but the shape is similar. See the [cuckoo-stash paper](https://www.eecs.harvard.edu/~michaelm/postscripts/esa2008full.pdf) for the underlying idea.

Crucially, the stash is not a complete in-memory copy of the database. The lookup hierarchy in TigerBeetle's `cache_map.zig` is:

```text
cache -> stash -> immutable table -> LSM
```

A miss in `CacheMap` is therefore not a correctness failure. `CacheMap.get()` checks the set-associative cache and then the stash; the broader LSM lookup path can continue below that. Correctness belongs to the whole hierarchy, not to the stash alone.

The stash's job is narrower: it gives hard in-memory availability for specific short-lived cases, such as prefetched values during commit, and catches values evicted from the fast tier.

### 6.2 Bounded by Capacity, Invalidated by Compaction

A natural objection is: does the stash retain entries indefinitely? No. Two separate mechanisms matter:

* **Bounded by construction.** `CacheMap.init()` calls `ensureTotalCapacity()` with `stash_value_count_max`, and later stash updates use assume-capacity operations. The code path is designed around fixed allocation rather than unbounded runtime growth.

* **Cleared by compaction.** `CacheMap.compact()` clears the stash while retaining its allocated capacity. The source comment is explicit that stash invalidation is handled by `compact`.

* **Driven by the LSM's compaction cadence.** TigerBeetle's LSM compaction runs incrementally in "beats" after commits. In Groove, the object cache is compacted on the last beat of the compaction bar, aligning cache invalidation with a deterministic storage-engine event.

The correctness invariant is intentionally narrow: call `compact()` only when the stash entries no longer need to provide their in-memory guarantee. If the cache is undersized, the operational cost is more misses and IO, not unbounded stash growth.

### 6.3 TigerBeetle's `SetAssociativeCache` Layout

The current default layout in
[`src/lsm/set_associative_cache.zig`](https://github.com/tigerbeetle/tigerbeetle/blob/main/src/lsm/set_associative_cache.zig) is more specific than the generic examples above:

* **16 ways** per set by default. The implementation currently allows 2, 4, or 16 ways for an efficient CLOCK-hand representation.
* **8-bit tags** by default, stored compactly for fast tag filtering. The code also supports 16-bit tags.
* **2-bit per-way counts** by default, used by a CLOCK Nth-Chance replacement policy.
* **Packed metadata.** Tags, counts, and clock hands are laid out with cache-line-sized packing constraints. Values are stored separately and may be larger than a cache line.

A lookup proceeds as follows:

1. Hash the key into a 64-bit entropy value.
2. Derive a short tag from that entropy.
3. Map the entropy to a set index.
4. SIMD-compare the set's tags against the probed tag.
5. Do full key comparisons only for ways whose tag matched and whose count is nonzero.

The tag compare is only a filter. An 8-bit tag can collide, so the final `key_from_value()` comparison is mandatory.

Two clarifications are worth keeping in mind:

* When the earlier sections talked about a "single cache line," that is most precise for compact set metadata. Full values may be too large to fit in one line.
* TigerBeetle does not use plain LRU or random replacement here. It uses a [CLOCK Nth-Chance](https://www.cse.iitd.ac.in/~sbansal/os/lec/l30.html) policy driven by packed per-way counts and a per-set clock hand.

### 6.4 Performance Deep Dive: PR #3200

[PR #3200](https://github.com/tigerbeetle/tigerbeetle/pull/3200) restructured the SIMD search function to improve instruction-level parallelism and out-of-order execution. The benchmark deltas reported in the PR discussion are roughly +3.5%, +4.2%, +5.9%, and +5.2%, with improvements on both x86 and ARM machines and higher measured IPC.

#### Change 1: SIMD Tag Compare to Integer Mask via `@bitCast`

The set's tags are compared against the probed tag using a Zig vector comparison:

```zig
const x: @Vector(layout.ways, Tag) = tags.*;
const y: @Vector(layout.ways, Tag) = @splat(tag);

const result: @Vector(layout.ways, bool) = x == y;
return @bitCast(result);
```

The resulting boolean vector is bit-cast into an integer mask whose set bits represent candidate ways. For the default 16-way layout, that mask is a `u16`. This is clean Zig and matches the current implementation.

#### Change 2: Bit Iterator to Fixed-Trip-Count Masked Scan

The old shape consumed the mask by repeatedly finding the next set bit with `ctz`, clearing that bit, and looping. The new shape scans all ways and tests each bit:

```zig
const ways: u16 = search_tags(set.tags, set.tag);
if (ways == 0) return null;

for (0..layout.ways) |way| {
    if ((ways >> @as(u4, @intCast(way)) & 1) == 1 and
        self.counts.get(set.offset + way) > 0)
    {
        if (key_from_value(&set.values[way]) == key) {
            return @intCast(way);
        }
    }
}
return null;
```

The explicit `@as(u4, @intCast(way))` is not decorative. In Zig, the right-hand side of a shift has a specific integer type. Since TigerBeetle's current layout caps the default search at 16 ways, `u4` is enough to represent shift counts 0 through 15.

The code comment says the fixed scan is intended to help **out-of-order execution**. The reason is dependency shape. The bit-iterator loop had a **loop-carried dependency**: each iteration needed the mutated mask before it could know the next candidate way. The fixed scan leaves each bit test independent, which gives the compiler and CPU more freedom to overlap count loads, value loads, and comparisons.

#### Change 3: Modulo to `fastrange`, and Simpler Tag Derivation

The index and tag derivation also changed:

```zig
const entropy = hash(key);

const tag: Tag = @truncate(entropy);
const index = fastrange(entropy, self.sets);
const offset = index * layout.ways;
```

The important performance point is removing integer division from the hot path.  TigerBeetle's [`stdx.fastrange()`](https://github.com/tigerbeetle/tigerbeetle/blob/main/src/stdx/stdx.zig) implements [Daniel Lemire's multiply-high range reduction](https://lemire.me/blog/2016/06/27/a-fast-alternative-to-the-modulo-reduction/): multiply a `u64` by the range as `u128`, then take the high 64 bits. That is not the same function as `%`, but it is appropriate for well-distributed hash-derived input when its bias properties are acceptable.

Do not blindly replace `%` with `fastrange` for structured or biased integers.  The TigerBeetle source includes a test named "fastrange not modulo" for exactly that reason.

#### Why These Changes Help ILP and OoO

**A) Fixed trip count exposes independent work.** The fixed loop tests `(ways >> way) & 1` independently for each way. With no mask mutation between iterations, the compiler can unroll more easily, and the out-of-order engine can overlap more address generation, count loads, and candidate value checks.

**B) Removing `ctz` plus bit-clearing shortens the serial chain.** `ctz` is fast, but the old loop used it as part of a stateful iterator. Each step depended on the previous mask update. The fixed scan turns that into many small, regular tests.

**C) Fixed control flow is friendlier to the front end.** A fixed `0..ways` loop has a simple shape. Even when it does more tiny tests, the regularity can be better for branch prediction, scheduling, and unrolling.

**D) `fastrange` removes a divider bubble.** Integer division and modulo are high-latency operations on many CPUs. Replacing `% self.sets` with multiply-high range reduction shortens the hash-to-index-to-load chain.

**E) More candidate checks can be in flight.** The new code may perform more scalar bit tests than the old bit iterator, but modern superscalar CPUs often prefer more independent work over fewer serialized operations. That is the core trade-off the PR exploits.

#### Why "More Work" Can Be Faster

The fixed scan can test all 16 ways even if only one tag matched. On paper, that looks wasteful. On a modern CPU, however, sixteen independent tiny tests can be cheaper than a smaller number of dependent operations. The PR's benchmark results support that trade-off: throughput improved while IPC also improved.

### 6.5 Before vs. After: A Schematic Pipeline Timeline

The timings below are illustrative. The exact cycle counts depend on the microarchitecture. The important shape is that a long serial operation disappears, and the candidate-scan loop loses a loop-carried dependency.

#### Before

```text
Time ->   0    4    8   12   16   20   24   28   32   36   40
------------------------------------------------------------------------
Index     |------ integer DIV: entropy % sets ------| idx
Tag                         shift/truncate ----------> tag
Set addr                              AGU ------------> base
Tags load                                  L1/L2 -----> tags
SIMD eq                                             ---> mask

Bit-iterator loop:
Iter0: ctz(mask) -> way0; clearbit(mask) -> mask'
       load count[way0]; maybe load value[way0]; compare

Iter1 waits for mask':
       ctz(mask') -> way1; clearbit(mask') -> mask''
       load count[way1]; maybe load value[way1]; compare

Iter2 waits for mask'' ...
```

The divider is on the critical path before the set load. The bit iterator then limits overlap because each iteration depends on the previous mask update.

#### After

```text
Time ->   0    4    8   12   16   20   24   28   32   36   40
------------------------------------------------------------------------
Index     fastrange multiply/shift -> idx
Tag       truncate(entropy) -------> tag
Set addr             AGU ----------> base
Tags load                  L1/L2 --> tags
SIMD eq                           -> mask via @bitCast

Fixed loop, independent tests:
Way0:  (mask >> 0) & 1   load count[0]   maybe load value[0]
Way1:  (mask >> 1) & 1   load count[1]   maybe load value[1]
Way2:  (mask >> 2) & 1   load count[2]   maybe load value[2]
...
Way15: (mask >> 15) & 1  load count[15]  maybe load value[15]
```

The index is available sooner, and the candidate checks have less serial state. That gives the out-of-order engine more independent work to overlap.

#### Critical-Path Summary

| Stage                    | Before                      | After                                | Why it matters                                  |
| ------------------------ | --------------------------- | ------------------------------------ | ----------------------------------------------- |
| Index computation        | `%` integer division        | `fastrange` multiply-high reduction  | Removes a long-latency divider from the path    |
| Candidate iteration      | `ctz` plus mask mutation    | Fixed loop with independent bit test | Removes loop-carried dependency                 |
| SIMD to scalar handoff   | Pointer-cast-style handoff  | `@bitCast(result)`                   | Clean conversion from vector result to mask     |
| Memory-level parallelism | Fewer candidates in flight  | More candidates can overlap          | Better latency hiding on cache misses           |

---

## 7. Takeaways

1. **Fixed-count masked scans can beat variable-length bit iteration.** If the old loop carries state from one iteration to the next, a fixed scan may expose more instruction-level parallelism even when it performs more tests.

2. **Replacing `%` on a hot path can matter.** Multiply-based range reduction, such as `fastrange`, can remove a high-latency divider when the input distribution makes it valid.

3. **SIMD filters work well with scalar verification.** Compare compact tags in parallel, turn the result into a mask, then run full key comparisons only for candidate ways.

4. **A fast bounded tier can compose with a correctness-owning storage hierarchy.** TigerBeetle's set-associative tier is a hot cache. The stash catches evictions and prefetched values. The LSM below remains the source of truth.

5. **Invalidation is easier when tied to a deterministic system event.** Clearing the stash during compaction keeps the memory story simple and bounded.

Set-associative caches are not a universal hammer. They trade global replacement freedom for bounded, predictable access. In a hot path like TigerBeetle's `CacheMap`, that trade-off is exactly the point.
