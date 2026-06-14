+++
title = "Scalar Replacement of Aggregates: How \"Copy to Locals\" Unlocks the Compiler"
date = 2026-06-15
description = "How copying a small aggregate into local variables exposes Scalar Replacement of Aggregates to LLVM, letting hot loops keep state in registers instead of repeatedly loading and storing through a pointer—illustrated with Zig/Rust reproductions and TigerBeetle's AEGIS-128L speedup."
template = "page.html"

[taxonomies]
tags = ["sroa", "llvm", "rust", "optimization", "zig", "compilers", "register-allocation"]
[extra]
+++

# Scalar Replacement of Aggregates: How "Copy to Locals" Unlocks the Compiler

Some performance fixes look almost suspiciously small in the diff. You change a couple of lines, the code still says the same thing to a human reader, and a hot loop suddenly has far less memory traffic.

TigerBeetle's [PR #3201](https://github.com/tigerbeetle/tigerbeetle/pull/3201) is one of those fixes. It changed the AEGIS-128L state update in `src/stdx/aegis.zig` from "operate through a pointer to `state.blocks`" to "copy the eight lanes into a local array, update the local array, then write it back at the boundary." The PR reported about a 2x improvement in the AEGIS microbenchmark and a 3-10% end-to-end throughput improvement in the measured TigerBeetle workloads.

The compiler idea behind this is **SROA**, or **Scalar Replacement of Aggregates**. More precisely, this is a manual SROA-friendly code shape: the source code removes an aliasing and visibility obstacle so LLVM can keep the hot state in registers instead of repeatedly loading and storing it.

This post explains what SROA is, why pointers and opaque memory effects get in the way, and how the "copy to locals, work, write back" shape helps. Then we will build small Zig and Rust reproductions and connect the result back to TigerBeetle's AEGIS code.

---

## 1. What SROA Actually Means

An **aggregate** is a value made from smaller values: a struct, array, tuple, or record. **Scalar replacement of aggregates** is the compiler transformation that breaks an aggregate into independently optimized pieces.

LLVM documents the `sroa` pass as a transformation that breaks aggregate `alloca`s, such as local structs or arrays, into member-level `alloca`s and then, when possible, promotes those pieces to scalar SSA values. That last part is the money: once the fields are SSA values, the register allocator can keep them in registers, and later optimization passes can reason about each piece independently.

That phrasing matters. SROA is not magic fairy dust for any struct-shaped value anywhere in the program. It is strongest when the aggregate is local, non-escaping, and boring. If the compiler sees "this local array is private to this function," it has a much easier proof than "this pointer points to caller-owned memory and maybe something opaque can observe it."

### A Tiny Before/After

Consider a function that updates a pair in place:

```c
struct Pair { long a, b; };

long sum_in_place(struct Pair *p) {
    p->a += 1;
    p->b += 2;
    return p->a + p->b;
}
```

The function must store the new `a` and `b` back through `p`, because mutating the caller's object is part of its behavior. A compiler may still optimize the loads and arithmetic, but it cannot simply pretend the pointed-to object is a private local.

Now compare a by-value shape:

```c
long sum_local(struct Pair in) {
    long a = in.a + 1;
    long b = in.b + 2;
    return a + b;
}
```

These functions are not semantically identical: the first updates the caller's pair, the second does not. That difference is the point. In the second version the fields are just private values. On x86-64 SysV, a two-`long` struct is commonly passed in registers, and the body can collapse to something like:

```asm
lea rax, [rdi + rsi + 3]
ret
```

No aggregate has to survive as memory in the optimized code. That is SROA's happy path: aggregate-shaped source, scalar-shaped machine code.

---

## 2. What Blocks It: Aliasing and Visibility

The hard question for the optimizer is not "would registers be faster?" The answer is nearly always yes. The hard question is "is it legal to keep this value in registers?"

When code repeatedly mutates state through a pointer, the compiler has to preserve the observable behavior of that memory. This becomes especially restrictive around operations that may read or write memory in ways the optimizer cannot see through:

* calls that are not inlined,
* volatile or atomic operations,
* inline assembly with a `"memory"` clobber,
* compiler or hardware fences,
* raw pointers whose aliasing contract is weaker than a language-level unique reference.

Compilers can recover in many cases, especially after inlining or when they have strong `noalias` information. But the robust, easy-to-prove pattern is:

```text
copy aggregate from memory -> mutate private locals -> write aggregate back in one boundary phase
```

Once the address of the local copy does not escape, the optimizer can scalarize the local copy freely. The pointer-owned memory is touched at the boundary. The hot loop works on private values.

That is the whole trick. You are not hand-allocating registers. You are making the compiler's proof small enough that the normal optimization pipeline can do its job.

---

## 3. Minimal Reproduction in Zig and Rust

The examples below intentionally include a compiler-only memory barrier inside the loop. The barrier is not part of the TigerBeetle change; it is just a demonstration tool.

Why add it? Without a barrier, a sufficiently smart optimizer may hoist, sink, or combine memory operations in the small example, making the difference harder to see. A compiler barrier says: memory operations may not freely move across this point. It does not stop pure register arithmetic. That makes the contrast crisp:

* pointer version: every round touches memory, so the barrier pins the loads and stores in the loop;
* local-copy version: the loop body touches private scalar values, so there is no per-round memory traffic to pin.

In optimized output, especially for Rust's `compiler_fence`, you may not see a barrier marker inside every local-copy iteration. Once the loop is register-only, the compiler-only barrier has no loop memory operations to constrain, so it may be moved relative to the register arithmetic. That is not a failure of the demo; the important thing to inspect is whether loads and stores remain in the loop body.

### Zig

```zig
const State = extern struct {
    blocks: [8]u128,
};

// Zig 0.15+ typed clobber syntax. This is a compiler barrier, not a CPU fence.
inline fn compilerBarrier() void {
    asm volatile ("" ::: .{ .memory = true });
}

fn roundBad(state: *State) void {
    // BAD: mutate the aggregate through a pointer every iteration.
    var i: usize = 0;
    while (i < 1000) : (i += 1) {
        state.blocks[0] +%= state.blocks[1];
        state.blocks[1] ^= state.blocks[2];
        compilerBarrier();
    }
}

fn roundGood(state: *State) void {
    // GOOD: copy by value into a private local array.
    var blocks: [8]u128 = state.blocks;

    var i: usize = 0;
    while (i < 1000) : (i += 1) {
        blocks[0] +%= blocks[1];
        blocks[1] ^= blocks[2];
        compilerBarrier();
    }

    state.blocks = blocks;
}

export fn entry_bad(s: *State) void {
    roundBad(s);
}

export fn entry_good(s: *State) void {
    roundGood(s);
}
```

The important line is:

```zig
var blocks: [8]u128 = state.blocks;
```

Read it literally:

* `var` declares a mutable local.
* `[8]u128` is a fixed-size array value, not a slice and not a pointer.
* `= state.blocks` copies the array value out of the state.

Mutating `blocks` does not mutate `state.blocks` until the explicit assignment at the end. These shapes do not make a private copy:

```zig
const slice: []u128 = state.blocks[0..state.blocks.len];
const p: *[8]u128 = &state.blocks;
```

Both are views of the original storage. They reintroduce the pointer-shaped memory access that we are trying to avoid.

Build and inspect:

```bash
zig build-obj sroa.zig -O ReleaseFast -fllvm -femit-asm=sroa.s
# Compare entry_bad and entry_good in sroa.s.
# entry_bad should load/store state in the loop.
# entry_good should load before the loop and write back after the loop.
```

For LLVM IR:

```bash
zig build-obj sroa.zig -O ReleaseFast -fllvm -femit-llvm-ir=sroa.ll
```

### Rust

```rust
use core::sync::atomic::{compiler_fence, Ordering};

#[repr(C)]
pub struct State {
    blocks: [u128; 8],
}

/// # Safety
///
/// `state` must be valid, properly aligned, and exclusively writable for the
/// duration of this call.
// SAFETY: the exported symbol name must be unique in the final linked program.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn entry_bad(state: *mut State) {
    // BAD: mutate through the raw pointer every iteration.
    for _ in 0..1000 {
        unsafe {
            (*state).blocks[0] = (*state).blocks[0].wrapping_add((*state).blocks[1]);
            (*state).blocks[1] ^= (*state).blocks[2];
        }
        compiler_fence(Ordering::SeqCst);
    }
}

/// # Safety
///
/// `state` must be valid, properly aligned, and exclusively writable for the
/// duration of this call.
// SAFETY: the exported symbol name must be unique in the final linked program.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn entry_good(state: *mut State) {
    // GOOD: copy the array, work on the local copy, write back at the boundary.
    let mut blocks = unsafe { (*state).blocks };

    for _ in 0..1000 {
        blocks[0] = blocks[0].wrapping_add(blocks[1]);
        blocks[1] ^= blocks[2];
        compiler_fence(Ordering::SeqCst);
    }

    unsafe {
        (*state).blocks = blocks;
    }
}
```

A few Rust details are deliberate:

* `#[unsafe(no_mangle)]` is the Rust 2024-compatible spelling of `#[no_mangle]`; the safety obligation is that the exported symbol name does not collide in the final linked program.
* The functions are `unsafe extern "C"` because a raw pointer from outside Rust needs a safety contract.
* The unsafe blocks are small, which keeps the actual raw-pointer operations visible.
* `[u128; 8]` is `Copy`, so `let mut blocks = (*state).blocks` is a by-value copy.
* `compiler_fence` is compiler-only. It restricts memory reordering by the compiler but emits no hardware fence instruction of its own.

Build and inspect:

```bash
rustc sroa.rs \
  --crate-type=lib \
  --edition=2024 \
  -O \
  -C target-cpu=native \
  --emit=asm \
  -o sroa.s
```

On a current optimized build, the pattern to look for is not "zero memory instructions anywhere." The good version still has to load the initial state and store the final state. The key property is narrower and more important: the **loop body** should be register arithmetic, while the bad version should retain loads and stores in the loop.

For LLVM IR:

```bash
rustc sroa.rs \
  --crate-type=lib \
  --edition=2024 \
  -O \
  -C target-cpu=native \
  --emit=llvm-ir \
  -o sroa.ll
```

In the good function, you may still see setup and write-back memory operations. Inspect the loop itself. The updated lanes should appear as SSA values carried through the loop, not as repeated loads from and stores to `state`.

---

## 4. What a Barrier Proves in the Demo

A **round** is one repeatable step of an algorithm. In this toy example, a round is just:

```text
blocks[0] += blocks[1]
blocks[1] ^= blocks[2]
```

In AEGIS-128L, the state update is built from AES round-function calls plus XORs with message blocks. The exact operation is different, but the code-generation problem is the same: several 128-bit lanes are updated repeatedly, and the fast path wants those lanes in registers.

The toy round only touches three lanes so the assembly stays easy to read. The TigerBeetle update is denser: it rotates through all eight lanes and then mixes in the two message blocks. That makes the register-residency problem more important, not less.

The demo barrier is a compiler barrier for memory operations:

* Rust's `compiler_fence` emits no machine instruction, but it restricts compiler memory reordering.
* Zig's empty `asm volatile` with a `.memory` clobber tells the optimizer that the assembly may touch arbitrary undeclared memory.

Neither one is a general inter-thread synchronization recipe. If you need synchronization, use the language's atomic operations and fences correctly. Here the barrier is only there to make the optimizer's memory behavior easy to see.

The asymmetry is the useful part:

```text
Through pointer: load -> math -> store -> barrier, repeated each round
Local copy:      load once -> register math loop -> boundary write-back phase
```

That is why the local-copy version can be much faster even though it appears to do an extra copy at the start. It pays a small boundary cost to avoid a large per-round cost. At the source level the boundary is one assignment, although the backend may lower that assignment to several scalar or vector stores.

---

## 5. Back to TigerBeetle: AEGIS-128L

The actual TigerBeetle PR is tiny. In `State128L.update`, the old code took a pointer to the state lanes:

```zig
const blocks = &state.blocks;
```

The new code copies the lanes into a local array:

```zig
comptime assert(state.blocks.len == 8);

var blocks: [8]AesBlock = state.blocks;
const tmp = blocks[7];

inline for ([_]usize{ 7, 6, 5, 4, 3, 2, 1 }) |i| {
    blocks[i] = blocks[i - 1].encrypt(blocks[i]);
}

blocks[0] = tmp.encrypt(blocks[0]);
blocks[0] = blocks[0].xorBlocks(d1);
blocks[4] = blocks[4].xorBlocks(d2);

state.blocks = blocks;
```

The `inline while` to `inline for` cleanup is nice, but the important performance change is pointer-to-array -> array value. That gives LLVM a private aggregate to scalarize. The final assignment is one source-level write-back phase after the update; the backend may still emit several stores to materialize the array.

Why does AEGIS respond so strongly? AEGIS-128L has a 1024-bit state made of eight 128-bit AES blocks, and the family is constructed from the AES encryption round function. On CPUs with AES instructions, the round operation wants to live in vector registers:

* x86 has `AESENC` and `AESENCLAST`, plus vector forms such as `VAESENC`.
* ARM has AES instructions such as `AESE` and `AESMC`.

Those instructions consume register operands and produce register results. If the surrounding code bounces every lane through memory around the AES instruction, throughput suffers. If the lanes stay in registers, the CPU can keep more AES operations in flight and avoid repeated load/store traffic.

The PR body shows exactly that contrast. The old assembly repeatedly loaded two lanes, ran `vaesenc`, and stored the result. The new assembly had several `vaesenc` instructions running on registers before the final stores. The reported microbenchmark improved from `aegis-old` wall time `0.10` to `aegis-new` wall time `0.04`, with lower instruction count and higher IPC. The reported end-to-end benchmark moved the `i8g` throughput metric from `359,253` to `387,293` and `i4i` from `263,106` to `271,259`.

That is a real systems-performance lesson: a local code-shape change in a crypto inner loop can move whole-application throughput when that loop is on the critical path.

---

## 6. How Far This Generalizes

The pattern is broadly useful, but it is not a law of nature. Use it when the hot state is small enough and plain enough that copying it locally lets the optimizer remove repeated memory traffic.

Good candidates:

* fixed-size arrays of integers or SIMD/vector blocks,
* small POD-style structs,
* cipher or hash states,
* parser states,
* tight loops where the same fields are updated repeatedly.

Important caveats:

* **Register pressure still exists.** If the local state is larger than the available register file, the allocator will spill some pieces. That can still be better than pointer traffic every round, but it is not free.
* **Copy cost can dominate.** If you only touch one field once, copying the whole aggregate is probably the wrong move.
* **Observable memory is different.** Do not apply this blindly to volatile memory, memory-mapped IO, atomics, or data that must be observed between steps.
* **Rust ownership matters.** `Copy` arrays are ideal. Moving non-`Copy` values into locals may be possible, but types with `Drop`, borrowing invariants, or interior aliasing often make the optimization less clean.
* **Measure the loop.** The goal is not to see no memory instructions anywhere. The goal is to remove unnecessary memory traffic from the hot loop.

If the state is slightly too large, tile it: copy the hot subset into locals, work on that subset, write it back, then move to the next subset. Shorter live ranges often help the register allocator more than one giant local copy.

---

## 7. Takeaways

* SROA is the compiler's ability to split local aggregate storage into scalar SSA values.
* Pointer-shaped code can hide whether memory is private, especially around opaque memory effects.
* Copying a small aggregate into a local value gives the optimizer a private object it can scalarize.
* In hot loops, the win is not the copy itself. The win is avoiding repeated `load -> operate -> store` traffic.
* In Rust, prefer safe `&mut` APIs in ordinary code, and use raw pointers only at real FFI or low-level boundaries with explicit safety contracts.
* In Zig, remember that `[N]T` assignment copies the array value; slices and pointers keep referring to the original storage.
* Always verify with optimized assembly or LLVM IR.

The deeper habit is simple: write the code so the optimizer can prove the thing you already know. "Copy to locals, work privately, write back at the boundary" is one of the most useful ways to make that proof obvious.

---

## References

* [TigerBeetle PR #3201: Improve AEGIS codegen](https://github.com/tigerbeetle/tigerbeetle/pull/3201)
* [TigerBeetle PR #3201 discussion: matklad on SROA](https://github.com/tigerbeetle/tigerbeetle/pull/3201#issuecomment-3219968770)
* [LLVM `sroa`: Scalar Replacement of Aggregates](https://llvm.org/docs/Passes.html#sroa-scalar-replacement-of-aggregates)
* [Rust `compiler_fence`](https://doc.rust-lang.org/std/sync/atomic/fn.compiler_fence.html)
* [Rust 2024 unsafe attributes](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html)
* [Zig inline assembly and clobbers](https://ziglang.org/documentation/master/#Assembly)
* [The AEGIS Family of Authenticated Encryption Algorithms, IETF draft](https://datatracker.ietf.org/doc/draft-irtf-cfrg-aegis-aead/)
* [NIST FIPS 197: Advanced Encryption Standard](https://csrc.nist.gov/pubs/fips/197/final)
* [Intel AES-NI overview](https://www.intel.com/content/www/us/en/developer/articles/technical/advanced-encryption-standard-instructions-aes-ni.html)
* [Compiler Explorer](https://godbolt.org)
