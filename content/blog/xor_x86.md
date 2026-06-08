+++
title = "Why x86 Zeroes a Register With `xor eax, eax`"
date = 2026-06-08
description = "A practical reference on why x86 zeroes a register with xor reg, reg - breaking down the size and dependency advantages over mov, CPU zeroing idiom recognition, and the subtle flags tradeoff, with real code examples and tradeoffs"
template = "page.html"

[taxonomies]
tags = ["cpu", "x86-assembly", "out-of-order-execution", "optimization"]
[extra]
+++

# Why x86 Zeroes a Register With `xor eax, eax`

If you have ever disassembled a program - even a trivial one - you have almost certainly seen this line:

```asm
xor eax, eax
```

It looks like a riddle. The instruction says "XOR the `eax` register with itself," but what it actually *means* is "set `eax` to zero." In optimized code, compilers and assembly programmers commonly reach for it instead of the more obvious:

```asm
mov eax, 0
```

Both leave `eax` holding zero. So why does the industry so often prefer the cryptic one? The short answer: it is smaller, it avoids a false dependency, and many CPUs have been taught to recognize it as a special "I want a zero" signal. The longer answer touches instruction encoding, out-of-order execution, register renaming, and one subtle correctness trap involving CPU flags.

This post works through all of it - starting from the intuition and ending at the details an assembly author actually needs to get right.

---

## The Essence (for the impatient)

- **`xor eax, eax`** and **`mov eax, 0`** both **set `eax` to 0.** Their effect on the *register value* is identical.
- `xor` wins on **size** (2 bytes vs. 5 for `eax`) and **dependency behavior** (modern CPUs recognize it as a "zeroing idiom" that does not need the old register value).
- They are **not** fully interchangeable: `xor` *modifies the CPU flags*; `mov` leaves them untouched. In flag-sensitive code, this difference matters.

If you only remember one thing: prefer `xor reg, reg` to zero a general-purpose register, *unless* you need the surrounding flags preserved.

---

## Why XOR-with-itself Equals Zero

XOR (exclusive OR) is a bitwise operation that compares two bits and returns `1` only when the bits differ:

| a | b | a XOR b |
|---|---|---------|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

Now apply that to a value XOR'd with *itself*. Every bit is being compared to a copy of itself, so every pair of bits is identical - and identical bits always produce `0`. Whatever was in `eax`, `eax XOR eax` is guaranteed to be all-zero bits. That is the trick: you do not need to know the old value to wipe it out.

---

## Reason 1: Smaller Code

Machine instructions are bytes, and bytes have costs. Here is how the two encode for the `eax` case:

```
xor eax, eax   →  31 C0              (2 bytes)
mov eax, 0     →  B8 00 00 00 00     (5 bytes)
```

The usual shortest encoding of `mov eax, 0` has to carry the literal value `0` as a full 32-bit immediate - four bytes of zeros riding along just to say "nothing." `xor eax, eax` encodes the operation alone; the operands are registers, so there is no immediate to store.

Small x86-64 note: writing a 32-bit register such as `eax` also clears the upper half of the corresponding 64-bit register, `rax`. So `xor eax, eax` is also a common way to zero all of `rax`; writing `xor rax, rax` works too, but needs an extra REX prefix byte.

Three extra bytes per zeroing sounds trivial. But zeroing registers is one of the most common things code does (loop counters, return values, clearing accumulators). Multiply three bytes across a large binary and you affect **code density** - how much real work fits in the instruction cache. Tighter code means fewer cache misses, which on a modern CPU is a real performance lever, not a micro-optimization fetish.

---

## Reason 2: The CPU Knows This Trick

Here is where it gets interesting. On modern x86 processors, `xor eax, eax` is not treated as a generic XOR that happens to produce zero. The hardware specifically recognizes the pattern `xor reg, reg` as a **zeroing idiom** and gives it special treatment.

To understand why that helps, you need two ideas from how modern CPUs actually run.

### Out-of-order execution and dependencies

Modern CPUs do not execute instructions strictly one after another. They run many in flight at once, reordering them to keep the execution units busy. The constraint is **data dependencies**: if instruction B needs the result of instruction A, B has to wait for A.

A normal XOR reads both its operands. So in principle `xor eax, eax` *reads* the old `eax` before overwriting it - which would force it to wait for whatever last wrote `eax`. That is a false dependency: the result (zero) does not actually depend on the old value at all.

CPUs are smart enough to know this. When they see `xor reg, reg`, they recognize the result is unconditionally zero and **break the dependency chain** - the instruction does not wait on the previous `eax` writer. This frees the out-of-order engine to schedule it immediately, which is exactly what you want in a tight, performance-critical loop.

### Register renaming - zeroing for "free"

There is a second, deeper trick. Internally, the named registers you write (`eax`, `ebx`, …) are mapped onto a much larger pool of physical registers - a mechanism called **register renaming**. It is how the CPU runs many instructions concurrently without them stepping on each other's register names.

On many microarchitectures, a recognized zeroing idiom can be handled entirely in or near this renaming stage: the CPU allocates a fresh physical register whose value is known to be zero. No integer ALU has to run, no arithmetic happens, and in the best case the instruction has **zero execution latency**. It is not literally free - it still takes bytes in the instruction stream and uses front-end/retirement resources - but it can disappear from the execution units. A `mov eax, 0` is still cheap, but it is not the canonical dependency-breaking zero idiom and generally has to create an immediate value through the normal instruction path.

So `xor eax, eax` is not just smaller - the processor can sometimes make it *vanish* from the execution part of the pipeline.

---

## Reason 3: Convention and History

The idiom is old. Early x86 processors already benefited from the smaller encoding, so assembly programmers and compiler authors adopted `xor reg, reg` as the standard way to zero a register when flags did not need to be preserved. Hardware designers, in turn, optimized for the pattern they saw everywhere - a feedback loop that cemented it as *the* idiom. Today GCC, Clang, MSVC, and hand-written assembly all commonly use it in that role.

---

## The Catch: Flags Are Not the Same

Now the nuance that trips people up - and the reason "functionally equivalent" is an overstatement.

x86 has a set of status **flags** (EFLAGS in 32-bit mode, RFLAGS in 64-bit mode) that many instructions update as a side effect. They record things about a result: was it zero, was it negative, did it carry, and so on. Conditional jumps like `jz` (jump if zero) read these flags.

The crucial difference:

- **`mov eax, 0` touches no flags.** A `mov` only moves data. Every flag keeps whatever value it had before.
- **`xor eax, eax` rewrites the flags** based on its result (zero):

| Flag | Meaning | After `xor eax, eax` |
|------|---------|----------------------|
| ZF (Zero) | result was zero | **1** (set) |
| SF (Sign) | result was negative | 0 |
| PF (Parity) | low byte has even parity | 1 (zero has even parity) |
| CF (Carry) | unsigned carry/borrow | 0 |
| OF (Overflow) | signed overflow | 0 |
| AF (Adjust) | auxiliary carry | undefined |

So the two instructions leave the *register* identical but the *flags* in completely different states.

### When this bites

Consider code that sets up a flag, then zeroes a register, then branches on that flag:

```asm
; ... something earlier sets ZF based on a comparison ...
mov eax, 0    ; eax = 0, ZF is preserved from the comparison
jz  label     ; branches on the EARLIER comparison's result
```

Swap in `xor` and the meaning changes:

```asm
; ... something earlier sets ZF based on a comparison ...
xor eax, eax  ; eax = 0, but ZF is now forced to 1
jz  label     ; ALWAYS branches - the comparison's result was clobbered
```

The `xor` version destroyed the flag the `jz` depended on. This is a genuine correctness bug, not a style preference.

Conversely, sometimes the flag side effect is *useful* - if you wanted ZF set anyway, `xor` gives you the zero and the flag in one 2-byte instruction.

---

## So Which Should You Use?

**Reach for `xor reg, reg`** in the overwhelmingly common case: you want a zero, and you do not need any flags preserved across the zeroing. You get the smallest encoding and the dependency-breaking form compilers generally prefer.

**Reach for `mov reg, 0`** when surrounding code depends on the flags as they were *before* the zeroing - you are deliberately interleaving a register clear into a sequence that will branch on an earlier comparison. Here you pay extra bytes and a possible execution cost to keep the flags pristine.

A practical heuristic: in straight-line code where the zeroing is followed by a fresh comparison anyway, the flag difference is invisible, so `xor` is free and correct. The flags only matter when a *later* instruction reads a flag set by an *earlier* one, with the zeroing wedged in between.

---

## A More Accurate One-Liner

The tempting summary - "`xor eax, eax` and `mov eax, 0` are equivalent" - is wrong in one important way. A precise version:

> `mov eax, 0` and `xor eax, eax` both set `eax` to zero, but they are not fully equivalent: `xor` sets ZF, clears CF and OF, sets SF and PF from the result, and leaves AF undefined, while `mov` leaves all flags unchanged. In nearly all cases `xor eax, eax` is preferred for its compactness (2 bytes vs. 5) and dependency-breaking zero-idiom recognition - unless preserving the existing flags is required.

---

## Takeaways

- **Value:** identical - both produce zero.
- **Size:** `xor` is 2 bytes, `mov` is 5. Better code density, better instruction-cache behavior.
- **Speed:** the CPU recognizes `xor reg, reg` as a zeroing idiom - it breaks the false dependency and can resolve during register renaming on many cores, sometimes with zero execution latency.
- **Flags:** the real asymmetry. `xor` rewrites most relevant status flags and leaves AF undefined; `mov` preserves them. This is architectural, not a microarchitecture-specific optimization, and it is the one place the choice can change program behavior.
- **Rule of thumb:** zero with `xor` by default; switch to `mov` only when flags must survive the zeroing.

---

*Inspired by [x86 Internals for Fun & Profit • Matt Godbolt • GOTO 2014](https://youtu.be/hgcNM-6wr34?si=j2gKGnJjOOa2NC0f).*
