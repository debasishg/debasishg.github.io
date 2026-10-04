+++
title = "Push Ifs Up and Fors Down: The Idiom, Its Algebra, and Its Limits"
date = 2026-09-27
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

## Introduction

One of the recommendations in TigerBeetle's [Tiger Style document](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md) reads:

> Centralize control flow. When splitting a large function, try to keep all switch/if statements in the "parent" function, and move non-branchy logic fragments to helper functions. Divide responsibility. All control flow should be handled by one function, the rest shouldn't care about control flow at all. In other words, "push ifs up and fors down".

The heuristic says that decisions (`if`s) belong with the caller, where the context to make them lives, while iteration (`for`s) belongs inside the callee, which should operate on batches rather than on one item at a time. Done together, this centralizes branching in one place and lets the work below it run as tight, branch-free loops over many items.

matklad has [blogged](https://matklad.github.io/2023/11/15/push-ifs-up-and-fors-down.html) about this principle and discussed its many virtues. In the post, he argues for two complementary moves:

**Pushing conditionals ("ifs") up:** If a function branches on its input, consider moving that branch to the caller. Instead of `frobnicate(walrus: Option<Walrus>)` unpacking the option internally, the caller handles the `None` case and the function takes a plain `Walrus`. The function's type now states its precondition, the caller often already knows the answer and can drop the check entirely, and with the branching centralized in one place, redundant or dead conditions become easy to spot. Narrowing the input this way is a form of filtering - but the point is where the decision lives, not how much data flows downstream.

**Pushing loops ("fors") down:** Make operations on batches the base case, with the scalar version as a special case. Rather than calling `frobnicate(walrus)` in a loop, provide `frobnicate_batch(walruses)` and let the loop live inside it. A batch function can pay its setup cost once per batch instead of once per item, and it can hoist any condition that doesn't change across the batch out of the loop, so the hot loop runs without a branch and is a candidate for vectorization.

The two moves compose. Given a collection of `Option<Walrus>` values, the caller discards the `None`s and unwraps the rest into a `Vec<Walrus>`, then hands that to `frobnicate_batch`, which never has to consider the `None` case at all.

```rust
let maybe_walruses: Vec<Option<Walrus>> = ...;
let walruses: Vec<Walrus> = maybe_walruses.into_iter().filter_map(|w| w).collect();
frobnicate_batch(&walruses); // never sees a None
```

The principle has applications well beyond function signatures. This post explores two of them - relational query optimization, and the algebra of functional programs - and is careful about where each analogy holds and where it stops.

## Analogy in Database Queries: Selections Early, Batches Below

Query optimizers apply a close cousin of this idea, though the vocabulary points the other way, so it is worth fixing the orientation first. A query plan is a tree whose leaves are table scans and whose root produces the result. Data flows up from the leaves, so "down the tree" means "earlier in execution". When an optimizer talks about *pushing a predicate down*, it means evaluating it as early as possible, which is the database counterpart of what the rest of this post calls "up" or "early".

**Selection pushdown (the "if"):** A selection (a `WHERE` predicate) is the filter. Optimizers push selections below joins and as close to the scans as possible, so that a join sees only the rows that can survive the predicate. The decision about which rows matter is made once, near the source, instead of being re-litigated on every intermediate result.

**Projection pushdown:** A projection removes columns rather than rows. It is not an `if` in any sense, but pushing it down shrinks the width of every tuple flowing through the plan, which cuts memory traffic and I/O. It rides along with selection pushdown rather than being an instance of it.

**Joins:** Optimizers do not defer joins as such; they choose a join order and place selections and projections below them. The effect is that the expensive combining operators run over the smallest inputs the query semantics allow.

**Vectorized execution (the "for"):** The closest database analogue of pushing fors down is the move from row-at-a-time (Volcano-style) execution, where each operator is called once per tuple through a virtual `next()`, to vectorized or batch execution, where each operator is called once per batch of a thousand or so tuples and runs a tight loop inside. That is `frobnicate` versus `frobnicate_batch`, at the level of a query engine: the per-call overhead and the per-call decisions are paid once per batch, and the inner loop is branch-light and cache-friendly.

## Analogy in Functional Programming and Category Theory

Let's look at the same principle through the lens of functional programming and a little category theory.

### Pushing Ifs Up as Restriction to a Subobject

In the category of sets (or, loosely, of types), a predicate `p : A → Bool` carves out a subobject `{a ∈ A | p a}` together with its inclusion `{a | p a} ↪ A`. That inclusion is a monomorphism. Pushing an if up means the caller does the case analysis and hands the callee an element of the subobject, so the callee's domain is the restricted one. Note that it is the inclusion that is mono, not the filtering function: `filter p : [A] → [A]` is certainly not injective, since it forgets every rejected element. And the restriction happens within the same category - a subset is a subobject, not a subcategory.

**Sum types and partial operations.** matklad's walrus example is the cleanest case. `Option<Walrus>` is the coproduct `1 + Walrus`: either nothing, or a walrus. A function that takes `Option<Walrus>` and branches inside is really a function out of a coproduct, and by the universal property of coproducts such a function is exactly a pair of functions, one for each summand. Pushing the if up factors that pair apart: the caller deals with the `1` summand, and the core function is just the `Walrus` component.

In Scala the case analysis on the coproduct is `fold`:

```scala
val maybeWalrus: Option[Walrus] = ...
val report: Report = maybeWalrus.fold(Report.empty)(process)
```

The branch lives in `fold`, at the call site, and `process: Walrus => Report` never sees the empty case. (`maybeWalrus.map(process).getOrElse(Report.empty)` means the same thing; `map` is the functor action on `Option`, and `getOrElse` supplies the other half of the case analysis.)

### Filter, Map and the Law That Relates Them

A common piece of advice is "filter before you map". It is worth being precise about when that is a legitimate rewrite, because the two obvious expressions are not equivalent:

```haskell
filter p (map f xs)   -- p inspects the *output* of f
map f (filter p xs)   -- p inspects the *input* of f
```

In the first line `p` has type `B -> Bool`; in the second it has type `A -> Bool`. The law that actually relates them is:

```haskell
filter p . map f  ==  map f . filter (p . f)
```

This is not a consequence of functoriality. It follows from parametricity, and it is easiest to see by factoring `filter` through `Maybe`:

```haskell
keep :: (a -> Bool) -> a -> Maybe a
keep p x = if p x then Just x else Nothing

filter p = catMaybes . map (keep p)
```

Here `filter p` itself is not a natural transformation - it cannot be, since `p` fixes the element type - but `catMaybes :: [Maybe a] -> [a]` is one, and that is where the naturality lives:

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

## Influence on Code Design

Thinking about the idiom algebraically also sharpens how you design code.

**Ifs up via higher-order functions.** Instead of passing a flag and checking it inside a function (`if flag then doX else doY`), the caller can pass in the behaviour itself. The branch moves to the caller, which decides which function to supply, and the callee has no control flow left to speak of. In categorical terms this is what exponential objects buy you: in a cartesian closed category the morphisms `A → B` form an object `B^A` of their own, so behaviour can be passed around like any other value.

**Fors down via batch morphisms.** On the loop side, the familiar example is batch APIs versus iterative ones - one set-oriented SQL statement versus a loop of row-by-row updates. Both live in the same category; the difference is in the shape of the arrow. A batch operation is a morphism `[A] → [B]`, and nothing forces it to be `map f` for some per-item `f`. That freedom is the whole point: a batch morphism can factor as "do the setup once, then do the per-item work", paying for one lock, one fsync or one network round trip per batch instead of one per item.

One practical application of this idea was pointed out by Joran Dirk Greef, the CEO of TigerBeetle, in a LinkedIn comment on an earlier post of mine on a similar topic:

> This idea (push ifs up, fors down) is also at the heart of TigerBeetle's performance, in how TigerBeetle ultimately processes on the order of 8K events per operation type (the CPU switches on operation type, the "if", and then the "for" loop processes the same hot instructions for all 8K events—amortizing locks across the network, and everything else).

The [TigerBeetle performance docs](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/concepts/performance.md) put the same idea this way:

> TigerBeetle works like a high-speed train — its interface always deals with batches of transfers, up to 8,190 transfers per query. Although TigerBeetle is a replicated database using a consensus algorithm, the cost of replication is paid only once per batch, which means that TigerBeetle runs almost as fast as an in-memory hash map, all the while providing extreme durability and availability.

matklad's post makes the performance argument with a specific shape of code. Given a condition that does not change inside the loop, he compares

```rust
// BAD
for walrus in walruses {
    if condition { walrus.frobnicate() } else { walrus.transmogrify() }
}

// GOOD
if condition {
    for walrus in walruses { walrus.frobnicate() }
} else {
    for walrus in walruses { walrus.transmogrify() }
}
```

and explains that the good version is good

> because it avoids repeatedly re-evaluating `condition`, removes a branch from the hot loop, and potentially unlocks vectorization. This pattern works on a micro level and on a macro level — the good version is the architecture of TigerBeetle, where in the data plane we operate on batches of objects at the same time, to amortize the cost of decision making in the control plane.

The qualifier matters: the performance win comes from hoisting a condition that is *invariant* across the loop. A condition that depends on each element cannot be hoisted out of a loop over those elements. The worked example below makes the difference concrete.

### A Worked Example: Orders

```haskell
import Data.List (foldl')
import Data.Maybe (mapMaybe)

data Order = Order
  { item      :: String
  , qty       :: Int
  , unitPrice :: Double
  }
```

Here is a first implementation of the order total, with the validation check buried in the loop body:

```haskell
total :: [Order] -> Double
total = foldl' step 0
  where
    step acc o
      | qty o > 0 && item o /= "test" = acc + fromIntegral (qty o) * unitPrice o
      | otherwise                     = acc
```

The check here depends on each order, so there is no way to lift it out of the loop - some loop somewhere has to look at every order and decide. What we *can* do is push the decision up to the boundary and record its outcome in a type, so that everything downstream is branch-free by construction:

```haskell
newtype ValidOrder = ValidOrder Order

validate :: Order -> Maybe ValidOrder
validate o
  | qty o > 0 && item o /= "test" = Just (ValidOrder o)
  | otherwise                     = Nothing

amount :: ValidOrder -> Double
amount (ValidOrder o) = fromIntegral (qty o) * unitPrice o

totalValid :: [ValidOrder] -> Double
totalValid = foldl' (+) 0 . map amount

totalClean :: [Order] -> Double
totalClean = totalValid . mapMaybe validate
```

This is the walrus refactoring again, with a batch on the other side:

- `validate` is the if, pushed up to the boundary. It still runs once per order, but it is the only place the check exists, and its result is recorded in the type.
- `totalValid` is the batch function. It cannot be handed an unvalidated order, and it contains no control flow.
- Any other consumer of `[ValidOrder]` - a tax calculation, a report, an export - inherits the guarantee without re-checking. This is matklad's point that once the caller knows the answer, the check can disappear from everywhere below it.

Both `total` and `totalClean` are also *monoid homomorphisms* from lists under `++` to numbers under `+`:

```haskell
totalClean (xs ++ ys) == totalClean xs + totalClean ys
```

(up to floating-point rounding, since `+` on `Double` is not strictly associative). This is what connects the algebra back to "fors down": a homomorphism lets you split the input into chunks, process each chunk as a batch - independently, and in parallel if you like - and combine the partial results with `+`. The batch size becomes a tuning knob rather than a change to the logic, which is exactly the freedom TigerBeetle exploits with its fixed-size batches.

Now for a condition that *is* loop-invariant. Suppose the total depends on a pricing mode that is fixed for the whole batch:

```haskell
data Pricing = Retail | Wholesale Double  -- discount fraction

-- if inside the loop: the pricing mode is re-examined for every order
totalPriced :: Pricing -> [ValidOrder] -> Double
totalPriced pricing = foldl' (\acc o -> acc + price o) 0
  where
    price o = case pricing of
      Retail      -> amount o
      Wholesale d -> amount o * (1 - d)

-- if pushed up: decide once, then run a single branch-free loop
totalPriced' :: Pricing -> [ValidOrder] -> Double
totalPriced' Retail        = totalValid
totalPriced' (Wholesale d) = (* (1 - d)) . totalValid
```

This is matklad's BAD/GOOD pair in Haskell, and the algebra helps twice. Hoisting the `case` removes the branch from the loop, and because multiplying by a constant distributes over `+`, the discount does not even need to be applied per order - it can be pulled out of the sum entirely, so both modes share the same loop (again, up to rounding). An optimizing compiler will sometimes unswitch a loop like the first version for you, but writing it the second way makes the structure explicit instead of hoping for it.

This is also the accurate way to read TigerBeetle's design. It does not filter events and then batch the survivors. It switches on the operation type once per batch - the loop-invariant if, pushed up - and then runs the same hot loop over up to 8,190 events. The per-event checks stay inside that loop, and every event gets its own result code, because those checks depend on the event and cannot be hoisted.

### A Note on Intermediate Structures

Written as a pipeline of `mapMaybe`, `map` and `foldl'`, `totalClean` looks as if it allocates intermediate lists. With optimization turned on, GHC's foldr/build fusion typically eliminates them, and the pipeline compiles down to a single loop that looks much like this:

```haskell
foldl' (\acc o -> if valid o then acc + amountOf o else acc) 0
```

which is essentially the original `total`. That is worth dwelling on. The per-element branch is still there after fusion, because it was never removable - a per-element decision has to be made per element. The refactoring bought clarity, a type that records the decision, and a batch function with no control flow in it. The branch that fusion cannot put back into the loop is the pricing one, because we hoisted it out by construction. That is the part of push-ifs-up-fors-down that actually pays for itself in performance, and it is why the idiom is phrased in terms of control flow rather than in terms of filtering data.
