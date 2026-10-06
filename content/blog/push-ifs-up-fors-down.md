+++
title = "Push Ifs Up and Fors Down: The Idiom, Its Algebra, and Its Limits"
date = 2026-10-04
description = "Tiger Style and matklad recommend pushing ifs up and fors down. This post looks at what the idiom actually says, where it shows up in query optimization and in the algebra of functional programs, and where the analogy stops paying for itself."
template = "page.html"

[taxonomies]
tags = [
  "software-design",
  "tiger-style",
  "tigerbeetle",
  "functional-programming",
  "category-theory",
  "haskell",
  "rust",
  "databases",
  "performance"
]
[extra]
+++

# Push Ifs Up and Fors Down: The Idiom, Its Algebra, and Its Limits
## Introduction

One of the recommendations from the [Tiger Style document](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md) of TigerBeetle ..

> Centralize control flow. When splitting a large function, try to keep all switch/if statements in the "parent" function, and move non-branchy logic fragments to helper functions. Divide responsibility. All control flow should be handled by one function, the rest shouldn't care about control flow at all. In other words, "push ifs up and fors down".

The programming heuristic push-ifs-up-fors-down suggests that conditional logic (if statements) should be moved upward (towards the caller or earlier in a pipeline), while iterative loops (for operations) should be pushed downward (toward batch processing or later in a tight branch-free loop). This improves clarity and performance by centralizing branching and leveraging bulk operations.

matklad has also [blogged](https://matklad.github.io/2023/11/15/push-ifs-up-and-fors-down.html) about this principle and discussed the many virtues of adhering to this programming idiom.

In the post, matklad describes optimizing code by:

**Pushing conditionals ("ifs") up:** If a function branches on its input, consider moving that branch to the caller. Consider the example from the post - instead of `frobnicate(walrus: Option<Walrus>)` unpacking the option internally, the caller handles the `None` case and the function takes a plain `Walrus`. The function's type now states its precondition, the input state space becomes narrower and acts as a form of filtering. But the point is where the decision lives, not how much data flows downstream.

**Pushing loops ("fors") down:** Deferring loops until after filtering or reducing the dataset minimizes unnecessary computations. Rather than calling `frobnicate(walrus)` in a loop, provide `frobnicate_batch(walruses)` and let the loop live inside it. So the hot loop runs without a branch and is a candidate for vectorization.

The two moves compose. Given a collection of `Option<Walrus>` values, the caller discards the `None`s and unwraps the rest into a `Vec<Walrus>`, then hands that to `frobnicate_batch`, which never has to consider the `None` case at all.

```rust
let maybe_walruses: Vec<Option<Walrus>> = ...;
let walruses: Vec<Walrus> = maybe_walruses.into_iter().filter_map(|w| w).collect();
frobnicate_batch(&walruses); // never sees a None
```

If you think a bit, the push-ifs-up-fors-down principle has far broader applications. This post explores some of the broader perspectives of this principle with respect to relational database query optimizations and functional programming and category theory.

## Analogy in Database Queries: Projections Early, Joins Late 

The same principle appears in database query optimization. In SQL query planning, it’s well-known that one should perform projections and selections as early as possible and defer joins or expansive operations until later. But the vocabulary runs upside-down. A query plan is a tree whose leaves are table scans and whose root produces the result. Data flows up from the leaves, so "down the tree" means "earlier in execution". When an optimizer talks about *pushing a predicate down*, it means evaluating it as early as possible, which is the database counterpart of what the rest of this post calls "up" or "early".

**Early projections and selections (Push down):** In database terms, a projection (e.g., SELECT with specific columns) reduces the dataset's width by selecting only the necessary columns early in the query execution. And selections (WHERE clause) act as the filter. Both are pushed down as part of the plan tree so that they get to execute early and filter out irrelevant data, reducing the amount of data passed to later operations.

**Deferring joins (Push joins up):** Joins, which combine data from multiple tables, are computationally expensive. The optimizer moves selections and projections below the joins, so joins run on smaller inputs. The effect is that the expensive combining operators run over the smallest inputs the query semantics allow.

**Vectorized execution (the "for"):** Another database analogy of pushing fors down is the change in execution semantics. Volcano style execution processes row at a time, where each operator is called once per tuple through a virtual `next()`. The alternative is the vectorized or batch execution, where each operator is called once per batch of a thousand or so tuples and runs a tight loop inside. That is `frobnicate` versus `frobnicate_batch`, at the level of a query engine: the per-call overhead and the per-call decisions are paid once per batch, and the inner loop is branch-light and cache-friendly.

## Analogy in Functional Programming and Category Theory

Let's take a look at the same principles through the lens of functional programming and category theory.

### Pushing-Ifs-Up as a Restriction to a Subobject

In category theory, when we talk about the category of sets (or, loosely, of types in programming), we can have a predicate `p : A -> Bool`. Elements can either satisfy the predicate or not. Now consider the subset of elements that satisfies the predicate, given by `{a ∈ A | p a}`. And add to it the morphism `{a | p a} ↪ A` that defines the inclusion. The subset along with the morphism defines a *subobject* in category theory.

> Note the hooked arrow in the morphism. It's intentional and it indicates that it represents a specific type of morphism - *monomorphism* (mathematicians love to define arrow types :-)). In Set, a monomorphism is an injective function: the inclusion sends each element to itself, so distinct inputs give distinct outputs.

Now we have the link back to matklad:
- Before: the callee takes any `A` and runs `if p(a)` inside itself.
- After: the caller runs the test, and the callee's input type is the subset.

The callee no longer needs the `if`, because every input it can receive has already passed. In code you represent the subset with a type: `Walrus` instead of `Option<Walrus>`.

Categorically speaking, `Option<Walrus>` is the coproduct `1 + Walrus`, which is either nothing or a walrus. A function that takes an `Option<Walrus>` and branches inside is really a function out of a coproduct, and by the universal property of coproducts such a function is exactly a pair of functions, one for each summand. Pushing the `if` up factors that pair apart: the caller deals with the `1` summand, and the core function is just the `Walrus` component.

### Filter, Map and the Law That Relates Them

Same principle in a different manifestation - the algebra of combinators. You commonly hear the advice of "filter before you map". But when exactly is this a piece of legitimate rewrite ? Because the two following expressions are not equivalent:

```haskell
filter p (map f xs)   -- p inspects the *output* of f
map f (filter p xs)   -- p inspects the *input* of f
```

In the first line `p` has type `B -> Bool`; in the second it has type `A -> Bool`. The law that actually relates them is:

```haskell
filter p . map f  ==  map f . filter (p . f)
```

This follows from parametricity, and it is easiest to see by factoring `filter` through `Maybe`:

```haskell
keep :: (a -> Bool) -> a -> Maybe a
keep p x = if p x then Just x else Nothing

filter p = catMaybes . map (keep p)
```

Here `filter p` itself is not a natural transformation - it cannot be, since `p` fixes the element type and you cannot draw the naturality square (try it!) - but `catMaybes :: [Maybe a] -> [a]` is one, and that is where the naturality lives:

```haskell
map g . catMaybes  ==  catMaybes . map (fmap g)
```

With that, the law is a short calculation. Since `keep p . f == fmap f . keep (p . f)`:

```haskell
filter p . map f
  == catMaybes . map (keep p) . map f
  == catMaybes . map (keep p . f)
  == catMaybes . map (fmap f . keep (p . f))
  == catMaybes . map (fmap f) . map (keep (p . f))
  == map f . catMaybes . map (keep (p . f))       -- naturality of catMaybes
  == map f . filter (p . f)
```

Notice that the right-hand side is not automatically cheaper: `filter (p . f)` still computes `f` for every element in order to test it. The rewrite pays off when `p . f` simplifies to a cheap predicate `q` on the input - typically because `p` inspects a part of the value that `f` leaves alone. Then `filter p . map f == map f . filter q`, and `f` runs only on the survivors. Both sides are still a single O(n) pass; what you save is the calls to `f` on elements that were going to be discarded.

And push-if-up-fors-down pays off!


## Summary

The main takeaway from the above discussion is that we can have general principles that guide us towards a better algebra of the code base or even the whole system. But all of these work subject to some constraints that must hold good across the components. 

- Taking an `if` out of a loop is valid when the condition is loop-invariant. A per-element condition cannot leave the loop. It can only move to the boundary and be recorded in a type, as `Walrus` instead of `Option<Walrus>`.
- Pushing a selection below a join is valid when the predicate references columns from only one side of the join.
- Filtering before mapping is valid as `filter p . map f == map f . filter (p . f)`, and it saves work only when `p . f` reduces to a cheap predicate on the input, typically because `p` inspects a part of the value that `f` leaves alone.

So, it's the algebra that tells you which rewrites are legal. In the filter/map case above, it's the naturality of `catMaybes` that drives the legality and helps you reason about the overall structure of the code.

For the "fors-down" part, it's more about the cost than the equivalence. You change the shape of the arrow from `A -> B` to `[A] -> [B]` so that you pay the setup cost once per batch.
