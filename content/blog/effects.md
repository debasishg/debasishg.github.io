+++
title = "Why Zig’s Io Feels Like an Effect System (Without Being One)"
date = 2026-05-11
description = "Explains how Zig’s Io gives you most of the engineering wins of effect systems (no function colouring, swappable interpreters) without any type-level tracking — here’s how it stacks up against Scala 3 Caprese and Kyo across the full design spectrum of effects"
template = "page.html"

[taxonomies]
tags = ["zig", "effect-systems", "scala", "io", "zig-io"]
[extra]
+++

# Why Zig’s `Io` Feels Like an Effect System (Without Being One)

## Context: What does "effect" even mean?

The word is overloaded, and most arguments about whether something "is" an effect collapse the moment you fix a sense. At least four are in play:

1. A **semantic side effect** - reading a file, mutating memory, throwing.
2. A **type-level effect annotation** - Koka-style effect rows, where the function type itself names which effects may occur.
3. An **algebraic operation plus handler** - a `perform`/`handle` mechanism that reifies the continuation and lets a handler decide whether, when, or how often to resume it (OCaml 5, Eff, Koka).
4. A **capability value** - an explicit value that grants the authority to perform an action, threaded through call sites (object-capability style).

These can stack but they are independent. Koka has all four. OCaml 5 has (1)+(3) but not (2): its manual is explicit that unhandled effects raise `Effect.Unhandled` at runtime, with no static effect safety. Zig `Io` is squarely (1)+(4).

When this document says "effect" without qualification, it usually means sense (3) - the algebraic, handler-based shape that the term carries in PL theory.

---

## The opening question: is `Io` an effect?

Zig 0.16 (April 2026) introduced an `Io` interface that you thread through any function that performs I/O. The release notes phrase it bluntly: "all input and output functionality requires being passed an `Io` instance." The threaded implementation (`Io.Threaded`) is feature-complete today; evented, `io_uring`, `kqueue`, and Grand-Central-Dispatch backends range from work-in-progress to proof-of-concept; `std.testing.io` is the recommended fake for tests.

A typical signature looks like:

```zig
fn readConfig(io: Io, path: []const u8) ![]u8
```

The same function body is intended to work across different `Io` implementations - the threaded backend today, evented or platform-specific backends as they mature - without splitting code into `async` and `sync` variants. This is the same engineering payoff that effect systems advertise: one body, many interpretations, no function colouring.

So: is it an effect?

**Short answer.** Not in the algebraic-effect sense (3), but it is exactly an effect in the capability sense (4), and it captures most of the engineering benefits people normally reach for an effect system to get.

### Why it is not an algebraic-effect system

- **No effect row for I/O.** Unlike Koka-style effect-typed systems, Zig does not attach an I/O effect row to the return type. Two functions that do wildly different I/O have indistinguishable signatures apart from "they both take an `Io`."
- **No effect operations plus handlers.** There is no `perform`/`handle` mechanism that captures a delimited continuation and lets a handler decide whether, when, or how often to resume it. You select an interpretation by *passing a different `Io` value*, not by handling an operation.
- **Ordinary interface dispatch.** `Io` is a value-level interface, not a statically elaborated effect operation.

The absence of effect rows is not by itself what disqualifies Zig - OCaml 5 has algebraic-effect handlers without effect rows, and we still call those algebraic effects. The decisive missing piece in Zig is sense (3): operations plus handlers that capture a continuation.

### Why it nevertheless feels effect-like

- It is an explicit **capability** threaded through call sites - the same pattern as Zig's `Allocator`. To do I/O, you must be handed the right to do I/O.
- It decouples *description* (call `io.read(...)`) from *interpretation* (threaded, evented, fake). That is the same separation algebraic effects give you, achieved with an interface value instead of a handler.
- It avoids `Future[T]` / `async fn` colouring: the same function body can be driven under different I/O strategies. It does **not** eliminate all visible plumbing - idiomatic Zig still accepts an `Io` parameter or stores it in a context struct - but it does eliminate the language-level split into separate function categories.

The accurate label is **capability-passing I/O**: a *value-level* effect, in the same family as Scala's ZIO environment, OCaml's `Eio` (deliberately designed before OCaml 5 effects landed and still capability-based), or the Reader-monad-of-IO pattern. Andrew Kelley has been explicit that Zig is not adopting an effect system; `Io` is an interface that gives you most of the engineering benefits of one without the type-system machinery.

---

## Unpacking the technical claim

The previous section said `Io` "is just a parameter" with no propagation up the call graph. Two distinct claims, worth separating.

### "The function signature doesn't carry an effect row"

In an effect-typed system (Koka), the *type* of a function lists the effects it can perform:

```
fun read_config() : <io, exn> string
```

That `<io, exn>` is the **effect row** - a set of effect labels attached to the return type. If `read_config` calls something that does `<net>`, the compiler forces you to either *handle* `net` inside `read_config` or *widen* its row to `<io, exn, net>`. Effects compose by union and are visible in every signature.

Zig's `Io` has nothing like that. The signature is just:

```zig
fn readConfig(io: Io, path: []const u8) ![]u8
```

The fact that it does I/O is encoded as a *value* (`io: Io`), not as a tag on the return type.

A note in passing: Zig already *does* track one effect-shaped thing in the type - errors, via error unions (`!T`). The example above uses `![]u8`. So the claim is not "Zig tracks nothing effect-like." The narrower claim is that **I/O authority** specifically is not tracked as a row or a capture set.

### "The compiler doesn't enforce that I/O-doing code propagates a marker upward"

In an effect-typed system, effects are *contagious* through the type checker:

```
fun a() : <io> ()        // does I/O
fun b() : <> ()  = a()   // ERROR — b's row is empty but it called something with <io>
```

The fix is forced: widen `b`'s row, or install a handler. You cannot silently launder an effect away.

In Zig, nothing propagates. If you have an `Io` in scope (e.g. in a struct field), you can call I/O from a function whose signature mentions no `Io` at all:

```zig
const std = @import("std");
const Io = std.Io;

const Logger = struct {
    io: Io,

    fn log(self: *Logger, msg: []const u8) !void {
        // `self.io` is the I/O capability; the Writer is obtained from it.
        var stderr = self.io.stderr();                                                         
        try stderr.writeAll(msg); // does I/O
    }
};

fn pureLooking(l: *Logger) !void {  // signature gives no hint
    try l.log("hi");                // …yet this does I/O
}
```

`pureLooking`'s signature says nothing about I/O. The compiler is fine with this. The "I can do I/O" capability rode in *inside* the `*Logger` value, invisibly. An effect-typed system would have rejected this until you either added the effect to `pureLooking`'s row or handled it locally.

### The underlying distinction

- **Effect row in the type** → the compiler knows, statically and structurally, which effects every function may perform, and *forces* that knowledge to flow up the call graph.
- **Capability as a value** (Zig's choice) → the compiler only knows "this function received an `Io`-shaped value." Whether it uses it, hides it, or passes it to ten other functions is invisible at the type level. Discipline is by convention, not by the checker.

Zig gives you the *runtime* benefit of swappable interpretations without the *static* benefit of guaranteed effect tracking.

---

## Enter Scala 3's experimental capture checking (Caprese)

Scala 3 ships an experimental capture-checking feature, enabled with `import language.experimental.captureChecking`. The Scala documentation flags it explicitly as a research project, evolving quickly. The associated research effort is sometimes called Caprese.

Both Zig's `Io` and Scala's capture checking converge on the same philosophical move - *"the ability to do X is a value, not an ambient power"* - and both reject monadic I/O / function colouring. The split is whether the type system **tracks** where the capability flows.

### What they share

- **Capabilities-as-values.** Scala's `IO`, `FileSystem`, `Async` and Zig's `Io` are values you receive as parameters and pass on. In the intended capability discipline I/O authority is explicit rather than ambient; the checker tracks captured capabilities instead of requiring a monadic `IO[T]` wrapper.
- **Decoupled interpretation.** Calling code is identical regardless of which interpreter is plugged in.
- **No function colouring.** A function that does I/O looks like any other function - it just takes an extra parameter (Zig) or carries a capture annotation on its arrow (Scala).

### What capture checking adds: the static layer

Capture checking encodes "which capabilities did this value or closure capture?" *into the type*. The current syntax puts capture annotations on the function arrow: `->` is a pure arrow that captures nothing, `->{c, d}` may use capabilities `c` and `d`, and `=>` is treated as the impure default that may capture arbitrary capabilities. Capability values themselves carry a `^` marker on their type.

```scala
val pure: String -> Int =
  s => s.length                    // pure arrow, captures nothing

val impure: String ->{fs} Long =
  path => fs.size(path)            // captures fs

class Logger(xfs: FileSystem^):    // a class that retains a capability
  def log(s: String): Unit = xfs.append("log", s)
// inferred type carries the captured capability, e.g. Logger^{xfs}
```

That gives Scala things Zig structurally cannot give you:

1. **Capability-disciplined purity.** A function value with an empty capture set is guaranteed not to capture any tracked capabilities. In a capability-disciplined subset where all I/O flows through tracked capabilities, this gives you a strong purity guarantee - the docs put it conditionally: *if* capabilities are the only means to induce side effects, then capability polymorphism corresponds to effect polymorphism. On the JVM at large, ambient effects still exist; the guarantee is relative to the discipline.

2. **Effect / capability polymorphism.** You can abstract over the capabilities a function may use, with explicit capture variables. A higher-order function can be parametric over *which* capabilities its callback needs, and the result type tells the caller exactly what was captured. Zig has no analogue; an `Io`-using callback just propagates the `Io` parameter manually.

3. **Scope-bounded capabilities.** The type system can guarantee a capability doesn't escape the scope that granted it:

   ```scala
   Resource.use { fs =>
     // fs : FileSystem^ is local; cannot be returned, stored in
     // a longer-lived ref, or captured by an escaping closure
   }
   ```

   This makes structured resource handling *enforced*, not advisory. In Zig, once you hold an `Io`, you can stash it in any struct of any lifetime and the compiler is silent.

4. **Capability retention is visible in the type.** When a class holds a capability in a field, instances acquire a captured type (e.g.  `Logger^{xfs}`). Generic containers introduce nuance - capture information can *tunnel* through a container's projections rather than always surfacing as a simple outer capture set - but the principle is that retention is tracked. The hidden-`Io` example from the earlier section is structurally invisible in Zig; its Scala analogue is not.

### Two-way comparison

| | Zig `Io` | Scala capture checking |
|---|---|---|
| Capability is a value | yes | yes |
| Avoids function colouring | yes (with plumbing) | yes |
| Runtime-swappable interpreter | yes (interface dispatch) | yes (different instance) |
| Pure functions distinguishable in type | **no** | **yes** (under the discipline) |
| Capability hidden in struct is tracked | **no** | **yes** (with capture-tunneling nuance) |
| Polymorphism over effects | **no** | **yes** |
| Compiler prevents capability escape | **no** | **yes** |
| Maturity | new in 0.16 | experimental research |
| Learning curve | trivial | significant |

### How to think about it

Capture checking is **what Zig's `Io` would become if you bolted an effect-tracking type system on top of it.** The runtime model is essentially the same - capabilities are plain values, dispatch is dynamic, there is no monad. Capture checking adds a *bookkeeping layer in the type checker* that records which capabilities each value transitively holds, and refuses programs where capabilities flow somewhere they should not.

The trade-off is honest:

- **Zig** gets you most of the engineering benefit (swappable I/O, no colouring, testability) for ~0% of the language complexity. You lose static guarantees: a function's signature does not tell you what it can do.
- **Scala** gets you the static guarantees at the cost of a non-trivial type-system feature that interacts with everything else (variance, inference, given/using, etc.).

`Io` is the *runtime substrate* of a capability system without the *type-level discipline*. Capture checking is what you get when you keep the substrate and add the discipline back - without going all the way to algebraic effects with handlers.

---

## Enter Kyo

Kyo sits in a third corner of the design space - it is the only one of the three whose public model is directly algebraic-effect-like: effects are pending in the return type, compose as a type-level set, and are discharged by handlers.

### What Kyo actually is

Kyo encodes effects in the *return type* using a pending-effects operator `<`. Its current core effects include `Sync` (side effects), `Async`, `Scope` (resource lifecycle), and `Abort[E]` (typed failure). The README describes the design as based on algebraic effects and modular handlers, with pending effects represented as unordered type-level sets via intersection.

```scala
def readConfig(path: String): String < (Sync & Abort[ConfigError]) = ...
def parse(s: String): Config < Abort[ConfigError]                  = ...

val program: Config < (Sync & Abort[ConfigError]) =
  readConfig("conf.json").map(parse)

// Handlers discharge effects, peeling them off the pending set:
val safe: Result[ConfigError, Config] < Sync =
  Abort.run(program)

// The remaining Sync is normally discharged at the application boundary
// (e.g. by extending KyoApp), not by ad-hoc unsafe runs.
```

Three things to notice:

1. **Effects appear in the type, not in the parameter list.** For ordinary effects such as `Sync` or `Abort`, `readConfig` does not take an `io: IO` parameter; the requirement is encoded in the pending-effects type. (Kyo can still model environments and services, but those are themselves represented as effects rather than as manually threaded interface values.)
2. **Effects compose by intersection** in the type (`Sync & Abort[E]`), not by passing more parameters.
3. **Handlers discharge effects.** `Abort.run`, `Async.run`, and the `KyoApp` boundary that runs `Sync`, peel an effect off the pending set, transforming the type. This is the algebraic-effect handler shape - code is *interpretation-free* until a handler runs it.

Under the hood, Kyo is a single CPS-trampoline encoding (not transformer stacks), so it gets direct-style-ish performance with a uniform composition story. It is a *Scala library* encoding of algebraic effects, not a language feature like OCaml 5 or Koka - worth keeping in mind when comparing it to those.

### Three-way table

| | Zig `Io` | Scala capture checking | Scala Kyo |
|---|---|---|---|
| Where the effect lives | parameter value | capture set on the type | pending set on the return type |
| Effect appears in signature | no | yes (`->{io,fs}`) | yes (`< (Sync & Fs)`) |
| Pure code distinguishable | no | yes (under the discipline) | yes (`A` vs `A < S`) |
| Effect / capability polymorphism | no | yes | yes (poly over `S`) |
| Handler-based discharge | **no** | **no** | **yes** (`Abort.run`, `Async.run`, `KyoApp`, …) |
| User-defined non-determinism, backtracking, custom scheduling | no | no | **yes** (handler controls the continuation) |
| Composition mechanism | n/a | union of capture sets | intersection of effect types |
| Resource scoping enforced by checker | no | yes (escape-checked) | partially, via `Scope` and disciplined handlers/runtime boundaries |
| Runtime model | interface dispatch | plain values | CPS trampoline |
| Avoids function colouring | yes (with plumbing) | yes | yes-ish (the colour moves into the *type*) |
| Learning curve | trivial | significant | significant (different shape from capture checking) |

### The three philosophies in one line each

- **Zig `Io`** - *"the ability is a value; the type system stays out of it."*
- **Capture checking** - *"the ability is a value; the type system tracks where the value flows."*
- **Kyo** - *"the ability is a type; handlers interpret it at the boundary."*

### Where Kyo gives you something the other two structurally cannot

Both Zig and capture checking let you **swap implementations** - pass a different `Io`, pass a different `FileSystem`. That is enough for "blocking vs. async vs. fake for tests." But neither lets the *handler* control the *control flow* of the effectful code. Kyo does, because handlers receive a continuation:

- **Non-determinism**: a `Choice` handler that runs the rest of the computation once per branch and collects results. You cannot express this by swapping a vtable - it requires capturing what comes after the effect.
- **Backtracking / search**: handler peeks at intermediate results and decides whether to resume, restart, or abandon.
- **Custom schedulers**: an `Async` handler that decides ordering, fairness, cancellation policy entirely in user code.
- **Effect translation**: a handler that converts `Abort[E1]` into `Abort[E2]`, or interprets `Sync` as `Trace` (logging every operation without performing it) for testing.

Capture checking cannot do this directly because capabilities are just *values* - they do not reify the continuation of the caller. Zig cannot do this because there is no notion of a handler at all; you only pick the implementation, not the control structure.

### Where capture checking gives you something Kyo doesn't

Capture checking still wins on one axis: **enforced lexical scoping of capabilities.** It can prove a `FileSystem` doesn't escape `Resource.use { fs => ... }` because the type system refuses to let `fs` flow into a longer-lived type. Kyo can model this with `Scope`, but the guarantee comes from the handler/runtime discipline, not from a structural impossibility - a poorly written handler could leak a resource. (In practice Kyo's standard handlers are correct, but the *type system* is not what is stopping you.)

Conversely, capture checking can't express "run this block 17 times under a different interpretation each time," which Kyo gets for free.

---

## Synthesis: a spectrum, not a hierarchy

Mapping back to the original question — *is Zig's `Io` an effect?* — the honest answer is that "effect" is a continuum, and these three systems sit at distinct points along it:

```
Zig Io  ──►  capture checking  ──►  Kyo  ──►  Koka / Eff   and   OCaml 5
(none)        (capture sets)       (lib AE)   (language AE; Koka/Eff are
                                               effect-typed, OCaml 5 has
                                               handlers without static
                                               effect safety and only
                                               one-shot continuations)
   │              │                    │            │
   │              │                    │            └─ Effects + handlers as
   │              │                    │               a first-class language
   │              │                    │               feature.
   │              │                    │
   │              │                    └─ Effects in types, handlers in user
   │              │                       code, implemented as a library on a
   │              │                       CPS runtime.
   │              │
   │              └─ Capabilities as values, with type-system tracking of
   │                 where capabilities flow. No handlers — interpretation
   │                 is chosen by which capability value you pass.
   │
   └─ Capabilities as values. No type tracking, no handlers.
      Engineering benefits of effects without the formal machinery.
```

Each step rightward buys more guarantees and more expressive control flow at the cost of more language or library complexity, more annotations, and (often) more runtime indirection.

### A final philosophical reading

There is a useful way to see the progression:

1. **Zig `Io`** treats effects as a *runtime engineering problem*: "we need the ability to swap implementations and avoid colouring functions." It solves exactly that and refuses to pay for anything more.

2. **Capture checking** treats effects as a *flow-of-authority problem*: "if a piece of code holds the ability to do X, the type system should know, and should be able to prevent that ability from leaking past where it was granted." It keeps the value-level model and adds a static accounting layer on top.

3. **Kyo** (and Koka, OCaml 5) treats effects as a *control-flow problem*: "operations are syntactic markers whose meaning is supplied by the surrounding handler, including how - or whether - to resume." Effects stop being about *who has permission* and become about *who decides what happens next*.

Zig's `Io` is, by this lens, the minimal viable point in the design space: enough effect-shape to fix function colouring and enable testing, no more. Whether that is "really" an effect depends on which sense of "effect" from §0 you have in mind - and on whether you are asking a language theorist or a working systems programmer. The theorist will say no in sense (3); the systems programmer will note that they got most of the benefit they cared about, with none of the syntactic or runtime cost - and that, for a language whose target audience is largely the second group, is probably the correct trade.

---

## Appendix: status as of writing

- **Zig `std.Io`** is new in 0.16.0 (April 2026). The threaded backend is feature-complete; evented, `io_uring`, `kqueue`, and Grand-Central-Dispatch backends range from work-in-progress to proof-of-concept. `std.testing.io` is the recommended fake for tests.
- **Scala 3 capture checking** is enabled via `import language.experimental.captureChecking` and is documented as a research project that is evolving quickly. Function-arrow syntax (`->`, `->{c}`, `=>`) and capture markers (`T^`, `T^{c}`) reflect the current shape; details may shift.
- **Kyo's** core effects today are `Sync`, `Async`, `Scope`, and `Abort` - not the older `IO` / `Resource` terminology. The README treats `KyoApp` as the standard application boundary that discharges `Sync`.