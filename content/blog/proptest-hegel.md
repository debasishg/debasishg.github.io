+++
title = "Evaluating PBT Frameworks: How Proptest and Hegel Differ in Algebraic Expressivity"
date = 2026-05-04
description = "Explains why it’s useful to visualize how these two libraries actually think about the data they generate. Proptest views the world as a static graph of possibilities, while Hegel views it as a live conversation"
template = "page.html"

[taxonomies]
tags = ["rust", "property-based-testing", "algebraic", "proptest", "hegel"]
[extra]
+++

# Evaluating PBT Frameworks: How Proptest and Hegel Differ in Algebraic Expressivity

## Context

I have recently been working on [`porcupine-rust`](https://github.com/debasishg/porcupine-rust) is a Rust port of [Porcupine](https://github.com/anishathalye/porcupine), a linearizability checker for concurrent and distributed systems, with APIs over timestamped `Operation` histories and raw `Event` histories, optional timeout-bounded checking, P-compositional partitioning, and support for nondeterministic step semantics through `NondeterministicModel` and `PowerSetModel`. In the same codebase, the runtime invariant layer explicitly ties implementation checks back to `INV-*` identifiers in `docs/spec.md`, including well-formed history, minimal-call frontier, partition independence, and cache soundness. That mix of algebraic laws, history-shape constraints, and incremental trace reasoning is exactly the kind of workload that exposes the real design trade-offs between testing libraries.

The project’s testing setup from the inside reveals two property-based testing implementations - `tests/property_tests.rs` (using [proptest](https://docs.rs/proptest/latest/proptest/)) and `tests/hegel_properties.rs` (using [hegel](https://docs.rs/hegeltest/latest/hegel/)). They are deliberately near-mirror suites over the same invariants, with shared builders under `tests/common/`, while also documenting the handful of places where the Hegel suite genuinely goes beyond the proptest one. That makes this repository a particularly good benchmark for “same domain, same models, same assertions, different property-testing architecture”.

## Why porcupine-rust makes this comparison unusually revealing

The repository’s core problem is [linearizability](https://arxiv.org/abs/1504.00204): given a partially ordered history of calls and returns, the checker has to decide whether those operations can be linearized into some sequential history consistent with the model. The README makes clear that the implementation is, in essence, a search engine over histories, with DFS/backtracking, bitset-based state tracking, partition-aware checking, and optional nondeterministic model support. Those characteristics matter for testing because failures are rarely “one bad scalar”, they are usually tangled histories with subtle timing relationships, partition boundaries, or branching semantics.

That is also why the invariant layer matters so much. In `src/invariants.rs`, the code documents that every macro or function corresponds to an `INV-*` identifier in `docs/spec.md`, and the partition-related checks nail down strong contracts: partitions must be disjoint, complete, in-bounds, and for event histories, keep each `(Call, Return)` pair together. Those are higher-order obligations over histories, not merely per-field sanity checks. A property-testing library used here therefore has to be good at constrained structured generation, shrinking of closely-related values, and, in the stateful case, construction of histories that remain meaningful as they are simplified.

There is a subtle but crucial caveat though: neither test suite is executing live concurrent Rust code with threads or async runtimes. Both suites are checking linearizability properties over pre-built histories. So the argument is not “which library is better at race testing by itself”; it is “which library better expresses the generation and shrinking of concurrent-looking histories and prefixes”. That framing is important, because it is exactly where Hegel’s state-machine API starts to matter and where proptest’s batch-generated histories remain perfectly adequate for a large class of checks.

## Proptest and Hegel in one paragraph each

Proptest is the established Rust-native member of the QuickCheck family. Its central abstraction is the `Strategy`, which defines both how values are generated and how they are shrunk. The Proptest book describes generation and shrinking as the two defining duties of a strategy, and the project describes the crate as feature-complete enough to be in long-term passive maintenance. In practical Rust terms, Proptest’s worldview is: build a typed generator graph, compose it with strategy combinators such as `prop_map`, `prop_flat_map`, `prop_filter`, and `prop_compose!`, and then feed the resulting values into a test body.

Hegel, by contrast, is a Rust property-testing library built on Hypothesis through the [Hegel protocol](https://hegel.dev). Its centre of gravity is the live `TestCase`: you draw values imperatively with `tc.draw(...)`, reject inputs with `tc.assume(...)`, attach debugging information with `tc.note(...)`, and build dependent generators with `#[hegel::composite]`. The official documentation also makes stateful testing a first-class module, with `#[hegel::state_machine]`, `#[rule]`, `#[invariant]`, `stateful::run`, and `Variables<T>` for pools of previously generated values.

A concise way to describe the difference is this: Proptest localises randomness in strategy values, while Hegel keeps randomness live inside the test execution itself. That is not a matter of syntax alone. It changes whether generation is a prelude to the test or part of the test algorithm. In a repository like `porcupine-rust`, that architectural distinction ends up being more important than slogans like “stateless versus stateful”.

## Where both libraries are equally strong

The strongest point in favour of keeping both libraries is that a large part of the repository does not actually force a choice. The project's strategy for enforcing invariants across spec and implementation suggests that for every `INV-*` invariant in `docs/spec.md`, the two suites contain near-mirror tests using the same shared builders, and it names several such pairs explicitly: `prop_time_shift_invariance` versus `hegel_time_shift_invariance`, `prop_slice_order_invariance` versus `hegel_slice_order_invariance`, `prop_concurrent_write_overlap_read_matches_membership` versus its Hegel counterpart, `prop_powerset_eq_hashed_powerset` versus `hegel_powerset_eq_hashed_powerset`, and cache-soundness checks on both sides. In other words, for a large class of finished-history properties, the repository itself already demonstrates that the two frameworks are expressively interchangeable.

The best concrete example is **dependent generation**. Let us walk through `arb_overlap_write_read` in the proptest suite and its Hegel mirror in `tests/hegel_properties.rs`. In proptest, the pattern is the familiar multi-stage `prop_compose!` form: first draw `write_dur`, then use `Just(write_dur)` to make that value available while drawing `read_call` from `0..write_dur`. That two-stage form is essentially sugar for `prop_flat_map`: the first stage produces a value, and the second stage builds a new strategy parameterized by it. Generation is still a function of strategies, evaluated before the test body runs. In Hegel, the same constraint is written directly with sequential draws: first `write_dur`, then `read_call` with its upper bound computed from that earlier draw. Both are principled encodings of a dependent generator - neither is resorting to rejection sampling.

---
**Rejection Sampling:**

A naive way to enforce `read_call < write_dur` would be to draw both freely and then throw away cases that violate the bound (rejection sampling), which wastes work and confuses shrinking. Both libraries avoid that by making the dependency structural. The difference is purely ergonomic: proptest expresses it through strategy combinators, Hegel through ordinary control flow.
---

A distilled version of that contrast looks like this:

```rust
// proptest style: dependency staged through the strategy.
prop_compose! {
    fn overlap_history()
        (write_dur in 5u64..30, write_value in -50i64..50)
        (write_dur in Just(write_dur),
         write_value in Just(write_value),
         read_call in 0u64..write_dur) -> History {
        // build overlapping write/read history
    }
}
```

```rust
// Hegel style: dependency expressed inline at the draw site.
#[hegel::composite]
fn overlap_history(tc: TestCase) -> History {
    let write_dur = tc.draw(gs::integers::<u64>().min_value(5).max_value(30));
    let write_value = tc.draw(gs::integers::<i64>().min_value(-50).max_value(50));
    let read_call = tc.draw(gs::integers::<u64>().min_value(0).max_value(write_dur - 1));
    // build overlapping write/read history
}
```

Those snippets differ in ergonomics, not in logical power for this category of problem. both cleanly encode `read_call ∈ [0, write_dur)` without rejection sampling - but proptest forces the dependency into the shape of the strategy (two parameter lists, Just re-binding), while Hegel lets it look like ordinary imperative code.

The proptest book explicitly documents `prop_compose!` and `prop_flat_map` as the mechanism for value-dependent strategies, and the Hegel docs explicitly document feeding the result of one `draw` into later `draw` calls in `#[hegel::composite]` generators.

This equivalence matters because it prevents an overstatement that would otherwise be tempting: Hegel is not “more expressive” in the repository’s dominant workload of stateless or value-level history generation. For overlap windows, shaped reads, algebraic invariants, powerset/hash equivalences, and deterministic cache checks, proptest’s strategy algebra and Hegel’s imperative draws are simply two different surfaces over the same basic generator logic. It would be right to say that, in this category, nothing in the repository is inherently beyond one framework or the other.

## Where Hegel is materially stronger in this repository

The expressive gap opens only when the history is not generated all at once, but grown step by step while the property is being checked. The test suite identifies three Hegel-only tests in `porcupine-rust`: `IncrementalKv`, `ConcurrentWritesChain`, and `IncrementalRegister`. In `IncrementalKv`, random writer and reader rules are selected repeatedly and the checker is asserted to return `Ok` on every prefix of the accumulated history. In `ConcurrentWritesChain`, the next operation’s legal call-time range is computed from state accumulated by previous rule firings. In `IncrementalRegister`, a newly generated read derives its expected output from a backward scan over prior history before the operation is appended. Those three tests are not just “stateful”; they interleave random generation, model-state inspection, mutation, and assertion in one loop.

Hegel’s stateful API is built exactly for that shape. The official stateful docs say that `#[rule]` methods mutate `&mut self`, `#[invariant]` methods are checked after each successful rule application, and `stateful::run` repeatedly applies random rules. The `StateMachine` trait itself is defined in terms of `rules()` and `invariants()`, and `Variables<T>` adds a symbolic pool of previously generated values when a state machine needs to carry handles or references forward across rule applications. In other words, the official abstraction and the repository’s most sophisticated tests line up almost perfectly.

A representative Hegel pattern for this repository looks like the following:

```rust
#[hegel::state_machine]
impl IncrementalKv {
    #[rule]
    fn write_random_key(&mut self, tc: TestCase) {
        // draw key/value/timestamps, append operation
    }

    #[rule]
    fn read_random_key(&mut self, tc: TestCase) {
        // draw key/timestamps, compute expected read from current history, append
    }

    #[invariant]
    fn prefixes_are_linearizable(&self, _: TestCase) {
        assert_eq!(check_operations(&KvModel, &self.history, None), CheckResult::Ok);
    }
}
```

This is not just pleasantly ergonomic. It preserves the semantic unity between “how the next move is drawn”, “how state evolves”, and “what must hold now”. That unity is exactly what makes Hegel support *stateful, rule-scheduled, prefix-asserting traces with mid-test adaptive draws*. 

Proptest can approximate parts of that story, but it does so by moving complexity into the strategy layer. The proptest state-machine chapter requires a `ReferenceStateMachine` with `init_state`, `transitions(state)`, and `apply`, plus a `StateMachineTest` with separate `init_test`, `apply`, and `check_invariants`. The same chapter explains that the sequential strategy ultimately generates a `Vec<Transition>` that is then interpreted against the SUT. That is a perfectly respectable design, and it has real strengths, but it is a different shape from the Hegel rule body that can keep calling `tc.draw(...)` after inspecting current machine state.

The subtle gain Hegel gets here is not merely “stateful testing exists”. Proptest also has state-machine testing. The gain is that Hegel allows generation to remain imperative and state-sensitive *inside* the rule. The `ConcurrentWritesChain` example in the tests is the clearest illustration: the next call time is drawn from bounds computed from `self.last_call` and `self.last_return`, and that choice sits in the same body as the subsequent assertion over the whole history. In proptest-state-machine, the same logic would have to be split across `transitions(state)` for generation, `apply` for mutation and local postconditions, and `check_invariants` for global assertions. That is doable; it is just materially more fragmented.

There is also a shrinker story behind this. The attached comparison note argues that these incremental, prefix-sensitive failures tend to benefit from Hypothesis-style shrinking because the counterexample is simplified at the same granularity as rule firings rather than only as a final batch-produced move list. I would phrase that carefully: not “Hegel always shrinks better”, but “for the failure shape produced by incrementally constructed histories, Hegel’s model fits the bug more closely”. In a linearizability checker, that can be the difference between a five-step witness history and a forty-operation hairball.

## Where Proptest is still the better tool

The first proptest advantage is operational, not theoretical: speed. The shorter attached note reports a project-specific measurement at 100 cases per test of roughly `0.05 s` for `cargo test --test property_tests` versus roughly `2.2 s` for `cargo test --test hegel_properties`. That gap is entirely plausible given the official Hegel installation model: `hegeltest` uses `uv`, may install or download its server component, and Hegel’s own maintainers explicitly say that the current Python dependency is the main performance limiter. If you care about fast inner-loop feedback while editing the checker, proptest is the obvious default.

The second advantage is ecosystem maturity. Proptest describes itself as close to feature-complete and largely in passive maintenance, which is usually exactly what infrastructure code wants from a test dependency: low surprise, stable idioms, and broad community familiarity. Hegel’s crate documentation, by contrast, carries an explicit beta disclaimer and reserves the right to make breaking changes if that improves the library. That is not a criticism of Hegel’s design; it is a concrete engineering cost. For CI-critical test suites in production Rust repositories, this stability differential matters.

The third advantage is that proptest’s “generator first” model is often the right abstraction when the generated object is the thing you want to persist, replay, or compose. Its failure-persistence mechanism stores failing cases in `proptest-regressions` so later runs replay them before searching for fresh failures, and its state-machine API is explicit about the split between the abstract reference state and the SUT. The attached note is right to call that split a genuine strength when there is a real effectful system under test—say a database, a service process, or an external component—because it forces the model/implementation distinction into the type shape of the harness.

The final Proptest advantage is that its combinator algebra rewards careful modelling. The official docs show that `prop_map` preserves the relationship to the original strategy so shrinking still happens in terms of the underlying source values, and `prop_flat_map` allows a later strategy to depend on an earlier drawn value while keeping invariants such as “the index stays within the vector”. In other words, Proptest is not weak at dependency management; it just insists that you make those dependencies explicit in the strategy graph. For teams that value typed, reusable, independently testable generators, that discipline is often a feature rather than a burden.

## Which design patterns belong to which library

* Choose Hegel when the trace is the test. If your property grows a history incrementally, checks every prefix, and must let later draws depend on already-mutated model state, Hegel is the more natural fit in Rust today. `porcupine-rust`’s `IncrementalKv`, `ConcurrentWritesChain`, and `IncrementalRegister` are exemplary here: the next operation is not merely sampleable from a static strategy; it is computed in dialogue with the current history. That design pattern maps directly onto `#[hegel::state_machine]`, `#[rule]`, `#[invariant]`, `tc.draw`, and `tc.assume`.

* Choose Hegel as well when you want imperative generator code that still shrinks coherently. The official `#[hegel::composite]` docs explicitly bless feeding one draw into subsequent draws, and the attached notes repeatedly emphasise that this is exactly how the Hegel versions of the porcupine history generators are expressed. In Rust terms, Hegel feels most at home when your generator wants to read like straight-line business logic with branches, helper calls, local calculations, and late assumptions.

* Choose proptest when the input object is a value, not an interaction. That includes algebraic laws over finished histories, finished operation vectors, transformations over timestamps, partitioning rules, and cross-checks over pure model functions. It also includes cases where you want generators to be exported as reusable functions, composed through `prop_map`, `prop_flat_map`, `prop_oneof!`, and `prop_compose!`, and reused across many tests. That pattern is heavily represented in `porcupine-rust`’s mirrored value-level properties, and the repository’s own comparison note says the two frameworks are effectively interchangeable there.

* Choose proptest when execution economics or harness architecture matter more than the last ounce of stateful elegance. If you want the fastest edit-run loop, a pure-Rust dependency with no sidecar, stored regression seeds, and a state-machine API that explicitly separates model from SUT, Proptest is the better day-to-day engineering choice. That is why it's recommended for inner-loop CI work even while crediting Hegel for stronger stateful coverage.

The most useful general rule I can extract from `porcupine-rust` is this: **proptest is strongest when generation is a specification artefact; Hegel is strongest when generation is part of the execution semantics of the property itself.** That is why both libraries look equally capable on overlap-window generators, yet Hegel pulls ahead once the history is allowed to evolve under a live scheduler with per-prefix assertions. It is also why proptest remains the saner choice for fast, stable, repository-wide coverage over law-like properties.

The right conclusion, then, is not that one library wins. The right conclusion is that `porcupine-rust` exposes two different testing geometries. For finished-history algebra, typed generator combinators, and cheap CI repetition, proptest is superb. For incremental history construction, stateful linearizability prefixes, and rule bodies whose random choices must remain entangled with current model state, Hegel is the more expressive instrument. In a repository that checks linearizability, partition independence, and nondeterministic model behaviour, appreciating the power of both is not compromise; it is architectural honesty.