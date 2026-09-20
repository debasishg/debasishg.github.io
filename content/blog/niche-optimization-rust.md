+++
title = "Niche Optimization in Rust: How `Option` can Get to Be Free"
date = 2026-09-20
description = "Most types cannot use every bit pattern their bytes can hold, and the Rust compiler spends the leftovers on enum discriminants. Where niches
come from, and where the trick stops working."
template = "page.html"

[taxonomies]
tags = ["rust", "memory-layout", "niche-optimization", "enums", "newtype",
"option", "performance"]
+++

# Niche Optimization in Rust: How `Option` can Get to Be Free

When we think of a type as a bound on the values it can represent, it automatically imposes a validity constraint on it. A `bool` takes a whole byte, but the valid values can only be 0 or 1. A reference is never null, an enum with 3 variants leaves every value past the third discriminant unused. 

Rust exploits these unusable patterns as an optimization technique for layout of the data structure - it calls this technique a *niche*. The compiler uses them, e.g. for appropriate cases when it lays out an enum it looks for a niche in one variant's payload and stores the discriminant there instead of allocating space for a separate tag.

In this post, I discuss some of the cases where Rust can do niche optimization (and some other cases where it cannot).

## Niche-optimized newtypes 
When designing data structures or applications, you will often come across values that have known invalid bit patterns e.g. in the standard library, for `NonZeroU32`, zero is never a valid value - hence all zeros is an invalid bit pattern that can never appear as a valid domain value. So, when you write `Option<NonZeroU32>`, the compiler uses that invalid pattern (all zeros) to represent a `None` instead of adding a tag for the discriminant, as it usually does for an `Option`. So the standard library guarantees that `Option<NonZeroU32>` is the same size as `u32`, using the `0` niche for `None`. This is an example of **niche optimization** in Rust.

Here's an example for such an optimization:

```rust
use std::num::NonZeroU32;

#[repr(transparent)]
#[derive(Copy, Clone, Eq, PartialEq, Hash)]
pub struct SlotId(NonZeroU32);

pub struct FreeListNode {
    next: Option<SlotId>,   // 4 bytes total, not 8 - no discriminant word
}
```

The wins you get are:
- expressivity of domain vocabulary without sacrificing any efficiency in representation
- strong typing - `SlotId` cannot be mixed with an `u32` id
- `#[repr(transparent)]` keeps the layout identical to the underlying integer

## Not only newtypes
An important point to note is that *niche optimization* is not only about newtypes. A *niche* is any bit pattern a type cannot hold and the enum layout doesn't need to allocate space for a discriminant. The Rust compiler can actually get quite creative in figuring out how to use invalid values for the *niche*.

Here's another example with a simple struct ..

```rust
struct Header  { len: u32, valid: bool }   // 0 or 1 only: 254 spare
struct NoNiche { len: u32, count: u32 }    // every bit pattern is legal

size_of::<Header>()           // 8
size_of::<Option<Header>>()   // 8   None = byte value 2, in `valid`
size_of::<NoNiche>()          // 8
size_of::<Option<NoNiche>>()  // 12  nothing spare, so a tag word is appended
```

Note in the `Header` case, `bool` represents 2 valid values and the rest of the 254 patterns (out of 256) allows the compiler to implement the *niche*. That's where `Header` and `NoNiche` differs ..

## Null pointer optimization
However, the most familiar *niche* that Rust implements is the null pointer. References, `Box` and `NonNull` can never be null and so `Option<&u32>` and `Option<Box<u32>>` cost the same 8 bytes as the bare pointer. `None` is stored as all zeros. This is known as the null pointer optimization. Along with `NonZero` this is something that the standard library guarantees.

## What about enums
Another source of *niche* implementation are the enums - a fieldless enum's valid range is the span of its discriminants, so every value outside the discriminant range is spare and can be used for the *niche*:

```rust
use std::mem::size_of;
pub enum ConnState { Handshaking, Open, Closing }   // valid: 0..=2

size_of::<ConnState>()          // 1
size_of::<Option<ConnState>>()  // 1   None takes the byte value 3
```

## Limitations
### Does it work for `Result`? 
So you can use `Option<ConnState>` in any of your domain types and pay nothing for the `Option`. An obvious question is whether the *niche* optimization is also applicable for `Result`. It's a qualified "yes" -  it doesn't hold for `Result` generally. But what if the `Err` variant can store its value directly without the tag and also there's an invalid bit pattern associated with it ? Use that invalid bit pattern (the spare value) to model the other variant, `Ok`.

Here's an example where `Box<u32>` (the `Err` variant) can never be null, so null is spare. `Err` keeps its layout untagged (i.e. stores its value directly). `Ok(())`, which needs no bytes of its own, is stored in that spare value (all zeros). Hence no tag.

```rust
Box<u32>                  // 8
Result<(), Box<u32>>      // 8 Ok needs no bytes: null encodes it
```

On the contrary, `NonZeroU32` can never be zero, so zero is spare. But that spare value is the whole 4-byte field, and `Ok`'s `u32` needs those same bytes, so there is nowhere to store it. Hence a tag: 4 for the tag, 4 for the payload.

```rust
NonZeroU32                // 4
Result<u32, NonZeroU32>   // 8  payload 4 + tag 4: niche unusable
```

The following is another interesting example - the `Err` variant in the `Result` grows in size from 0 to 8, but the size stays at 16.
```rust
struct Big { x: u64, n: NonZeroU8 }   // 16 - niche byte at offset 8

Result<Big, ()>    // 16
Result<Big, u8>    // 16
Result<Big, u64>   // 16 - Err carries a full u64, still niche-filled
```

`Result<Big, u64>` is 16 bytes. The `Err` variant carries eight bytes of payload and the optimization still fires, because `Big`'s niche byte sits at offset 8 and the u64 goes in bytes 0..8. Nothing overlaps.

### What about fixed layouts? 
If you ask for a fixed layout, you lose the optimization, since `#[repr(C)]` and `#[repr(u8)]` pin the tag beside the payload.

### Many variants 
In order to have niche optimization, you need to have enough spare values to fit in all the niches. When you have a payload with a single spare pattern (e.g. `NonZeroU32`), you can encode exactly one other variant. In the following example, `Two` fits the case as you can use the single value spare from `W`. But for `Three`, you need the tag since you have 2 variants and 1 spare value.

You can use the same counting logic to explain why the trick nests (e.g. below). `ConnState` leaves 253 spare values, so each `Option` layer spends just one and there is plenty of room left.

```rust
Option<Option<ConnState>>            // 1
struct W { a: u32, b: NonZeroU32 }   // 8 : exactly 1 spare pattern
enum Two   { A(W), B }               // 8 fits
enum Three { A(W), B, C }            // 12 : 2 needed, 1 available: tag
```