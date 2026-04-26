+++
title = "Dyn-Compatible Async Traits in Rust: Why the Manual Boxed Future Idiom is Required"
date = 2026-04-26
description = "Explains why async fn in traits breaks object safety for dynamic dispatch and how the explicit pinned boxed future return type, combined with 'static + Send bounds and pre-move cloning, restores dyn compatibility for service-oriented trait objects"
template = "page.html"

[taxonomies]
tags = ["rust", "async", "traits", "future", "pin"]
[extra]
mermaid = true
+++

# `Pin<Box<dyn Future + Send>>` in Traits - Why the Manual Desugar?

## Context

When a Rust trait needs async methods **and** must be usable as a trait object (`Box<dyn Trait>`, `&dyn Trait`, `Arc<dyn Trait>`), you cannot just write `async fn` and move on. `async fn` in traits returns an **opaque** `impl Future` whose concrete type differs per implementation - which breaks vtable dispatch. The compiler's own error-recovery hint spells it out:

> to make this trait dyn-compatible, use `-> Pin<Box<dyn Future<Output = ...> + Send>>` instead.

The manual form - write a synchronous fn returning `Pin<Box<dyn Future<Output = T> + Send>>`, implement it with `Box::pin(async move { ... })` - exists to preserve dyn-compatibility. It is what `#[async_trait]` generates under the hood. It is what `tower::Service`, `hyper::Service`, and many service-oriented abstractions use (sometimes with an associated future type, sometimes with the boxed form). It looks unfamiliar at first; it is mechanically load-bearing.

This doc explains why the pattern exists, what it gives you, and the lifetime subtlety that turns every "clone Arc fields before `async move`" line in the implementation into a consequence of the trait's signature.

---

## 1. The setup: async + polymorphism + pluggable backends

A common design:

- You define a trait that abstracts some capability with async methods - an executor, a service, a storage backend, a message bus.
- Production uses one implementation (a real HTTP executor, a real database backend).
- Tests use another (a mock that records calls and returns canned responses).
- Consumers of the trait want to swap between them at construction time without recompiling.

The most natural Rust encoding is:

```rust
pub struct Dispatcher {
    executor: Box<dyn StepExecutor>,   // production or mock, decided at construction
}
```

For this to compile, `StepExecutor` must be **dyn-compatible** (the compiler's current term for what used to be called "object-safe"). That is: you can build a vtable for it, because every method's signature is known at compile time regardless of the `Self` type that implements it.

Synchronous methods are trivially dyn-compatible. Async methods are not - and that is where the whole pattern starts.

---

## 2. What `async fn` actually is

`async fn foo(...) -> T { body }` is sugar for:

```rust
fn foo(...) -> impl Future<Output = T> { async move { body } }
```

The return type is `impl Future<Output = T>` - an **anonymous existential type** whose concrete identity is "whatever compiler-generated state machine this particular function body produced." The caller gets something that implements `Future`, but the caller does not know (and cannot name) the concrete type (that's what existential types are for).

This is excellent for monomorphized, generic code. The compiler specializes each call site against the concrete future type and inlines aggressively.

It is fatal for dyn dispatch.

### 2.1 Why `impl Future` and dyn dispatch are incompatible

A vtable stores function pointers with fixed signatures. For a sync method `fn run(&self, x: u32) -> bool`, the vtable slot is a pointer to a function taking `(&dyn Trait, u32)` and returning `bool`. Every implementation of the trait must produce a function with that exact signature.

For `fn run(&self, x: u32) -> impl Future<Output = bool>`:

- Implementation A's `impl Future` is some compiler-generated `Running<A>` state machine.
- Implementation B's `impl Future` is a different `Running<B>` state machine with a different size, different layout, and different `poll` code.
- They are **not the same type**. There is no single function signature the vtable can store.

You cannot put `Running<A>` and `Running<B>` behind the same function pointer. Polymorphism through dispatch requires a stable, type-erased return type. `impl Future` is the opposite of type-erased.

### 2.2 What the compiler does today (Rust 1.75+)

You *can* write `async fn` in traits. The compiler added that support in Rust 1.75 (late 2023). But the same method on `dyn Trait` is rejected with an error that explicitly instructs you to switch to the manual form.

Consider this minimal example - a trait with one `async fn`, then an attempt to use it behind a trait object:

```rust
type Result<T> = std::result::Result<T, Box<dyn std::error::Error>>;

pub trait StepExecutor {
    async fn execute(&self) -> Result<()>;
}

fn build() -> Box<dyn StepExecutor> { todo!() }
```

The trait declaration compiles fine on its own. The line `Box<dyn StepExecutor>` is what trips the dyn-compatibility check, producing:

```
error[E0038]: the trait `StepExecutor` is not dyn compatible
  --> src/lib.rs:10:18
   |
10 | fn build() -> Box<dyn StepExecutor> { todo!() }
   |                  ^^^^^^^^^^^^^^^^^ `StepExecutor` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
help: consider boxing the future:
   |
-    async fn execute(&self) -> Result<()>;
+    fn execute(&self) -> Pin<Box<dyn Future<Output = Result<()>> + Send + '_>>;
```

The compiler is telling you the lowered form. Adopting it is *the* way to keep dyn compatibility.

---

## 3. The manual form, line by line

Here is the paired trait declaration and impl, written the manual way:

```rust
// Trait declaration
pub trait StepExecutor: Send + Sync {
    fn execute_prepared(
        &self,
        requests: Vec<PreparedRequest>,
        max_concurrency: usize,
    ) -> Pin<Box<dyn Future<Output = Vec<Result<ExecutionResult>>> + Send>>;
}

// Implementation
impl StepExecutor for ParallelExecutor {
    fn execute_prepared(
        &self,
        requests: Vec<PreparedRequest>,
        max_concurrency: usize,
    ) -> Pin<Box<dyn Future<Output = Vec<Result<ExecutionResult>>> + Send>> {
        let executor = self.http_executor.clone();     // (1) snapshot fields
        let handler  = self.adapter_handler.clone();   //     before the async block

        Box::pin(async move {                          // (2) build the future
            let sem = Arc::new(Semaphore::new(max_concurrency.max(1)));
            let mut set = JoinSet::new();
            for req in requests {
                let exec = executor.clone();
                let hdl  = handler.clone();
                let s    = sem.clone();
                set.spawn(async move {
                    let _p = s.acquire().await.ok()?;
                    exec.dispatch(req, hdl).await
                });
            }
            set.join_all().await
        })                                             // (3) return the pinned box
    }
}
```

What each piece does:

**(1) Clone fields before `async move`.** The returned future is `Send + 'static` (no explicit lifetime bound in the trait signature means the elided lifetime must outlive the future - and since there's no lifetime parameter on the return type, it is `'static`). `async move` captures everything it references by move. If it captured `&self.http_executor`, the future would need to outlive `self`, which means its type would need a lifetime parameter (`+ '_`). The trait said no such parameter. So the body cannot touch `self` directly - it must capture owned clones of Arc-wrapped fields.

**(2) `Box::pin(async move { ... })`.** The `async move { ... }` block produces some compiler-generated future type with the moved-in captures. `Box::pin` heap-allocates it and pins it - pinning is required because futures may self-reference (an await point can borrow locals that live in the future's state machine), and moving such a future invalidates those borrows. Pinning prevents the move. Once allocated and pinned, we erase the concrete type by coercing to `Pin<Box<dyn Future<Output = T> + Send>>`.

**(3) The return type unifies every impl's future.** `ParallelExecutor::execute_prepared` returns one heap-allocated, pinned future. `MockStepExecutor::execute_prepared` returns a different heap-allocated, pinned future. To the caller - through `dyn StepExecutor` - they both look like `Pin<Box<dyn Future<Output = ...> + Send>>`. The vtable stores one function pointer; each impl's pointer builds and returns a different underlying future, but all wearing the same type-erased wrapper.

That is the whole trick. The return type is the lingua franca that makes dispatch possible.

---

## 4. Why the `.clone()` dance is load-bearing, not just style

The trait signature controls what the returned future may and may not do. Two constraints matter:

### 4.1 `+ Send`

The future will cross thread boundaries - it will be spawned on a tokio runtime, passed to a `JoinSet`, `select!`-ed against other futures, etc. `Send` is mandatory for that to be sound. Every value captured by the `async move` must itself be `Send`. If `self.http_executor` is `Arc<HttpExecutor>` and `HttpExecutor: Send + Sync`, then `Arc<HttpExecutor>: Send`, and cloning it into a local gives you a `Send` owned handle. If you tried to capture `&self.http_executor`, you'd capture a `&HttpExecutor` - which is `Send` only if `HttpExecutor: Sync`, and even then you'd tie the future's lifetime to `self`.

### 4.2 `'static` (implicit in the absence of `+ '_`)

Inside `Pin<Box<dyn Future<Output = T> + Send>>`, the trait-object bound is `'static` by default. There is no lifetime parameter on the return type. The future must not contain any borrows that outlive the enclosing function - it must own all its captures.

If you write:

```rust
fn execute_prepared(&self, ...) -> Pin<Box<dyn Future<Output = T> + Send>> {
    Box::pin(async move {
        do_work(&self.http_executor).await  // captures &self!
    })
}
```

The compiler rejects this: `self` has an anonymous lifetime bounded by the call, but the returned future needs `'static`. The error will be some variant of "`self` does not live long enough" or "captured variable cannot escape `FnMut` closure body" or "argument requires that `'1` must outlive `'static`," depending on the exact shape.

The fix is to replace the borrow with an owned clone:

```rust
fn execute_prepared(&self, ...) -> Pin<Box<dyn Future<Output = T> + Send>> {
    let executor = self.http_executor.clone();  // owned Arc, 'static
    Box::pin(async move {
        do_work(&executor).await
    })
}
```

This is the same idiom Tokio documentation shows for `tokio::spawn`, which has the same constraint (`Send + 'static`). The spawn body clones Arcs before `move` into the task. Trait-level dyn-safe async works the same way for the same reason.

### 4.3 If you want to borrow `self`, say so

You can write a dyn-safe async trait method that *does* borrow `self`, by adding the lifetime:

```rust
fn execute_prepared<'a>(
    &'a self,
    ...
) -> Pin<Box<dyn Future<Output = T> + Send + 'a>>;
```

Now the future is permitted to borrow `self` for the duration `'a`. The cost is that callers cannot store the future beyond the borrow; it cannot be `tokio::spawn`ed as-is without some form of owned-handle wrapping. Most service-oriented traits choose the `'static` flavor (no `'a`) precisely so the futures are spawnable; they accept the clone-before-move cost as the price of spawn-ability.

---

## 5. `#[async_trait]` is the same thing, macroed

The `async-trait` crate (proc macro) lets you write the sugared form:

```rust
#[async_trait]
pub trait StepExecutor: Send + Sync {
    async fn execute_prepared(
        &self,
        requests: Vec<PreparedRequest>,
        max_concurrency: usize,
    ) -> Vec<Result<ExecutionResult>>;
}
```

At expansion time, the macro rewrites both the trait and every `#[async_trait] impl` into the manual form we just showed. The return type becomes `Pin<Box<dyn Future<Output = ...> + Send + 'async_trait>>` (with a generated lifetime that matches `&self`); the body gets wrapped in `Box::pin(async move { ... })`; and a handful of scaffolding traits are generated.

That is:

- `#[async_trait]` style and manual style produce **byte-identical** compiled output in many cases.
- The macro is a purely syntactic shortcut. It buys you nothing at runtime.
- The cost of the macro is an extra dependency, some compile-time overhead, and a layer of obfuscation when reading expanded diagnostics.
- The cost of the manual form is ~5 more lines per method: one `fn` signature with `Pin<Box<…>>`, one `Box::pin(async move { … })`, one set of field clones.

Choosing between them is largely a judgment call about dependency hygiene and explicitness. For a library with a handful of dyn-safe async methods on one trait, the manual form is often preferable: no proc macro in the dep graph, the reader sees exactly what's happening, and rustc diagnostics are about your real code rather than generated code.

For a codebase with dozens of such traits, `#[async_trait]` amortizes its dep cost and reduces ceremony. Common Rust advice: prefer the manual form when it's small and contained; prefer `#[async_trait]` when the volume justifies it.

---

## 6. Alternatives and when to reach for them

The boxed form is not the only way to make async work across trait implementations. It is the most common one when you need `dyn Trait`. Here are the alternatives worth knowing.

### 6.1 Associated future type (the "tower" style)

```rust
pub trait Service {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>> + Send;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

Each implementation names its own future type. No boxing, no allocation per call. The catch: it is not dyn-compatible on its own - to use `dyn Service`, you typically wrap it with a helper like `tower::util::BoxService` that boxes the future at the object boundary. `tower` takes this approach precisely to decouple the trait's abstract shape (zero-cost associated future) from the erasure-at-the-edge concern (box only when you need dyn).

Good choice when you are writing a service that will *usually* be composed generically and *occasionally* boxed. Overkill for traits that always want dyn dispatch anyway.

### 6.2 Native `async fn` in traits (without `dyn`)

If you do not need `dyn Trait`, post-1.75 you can simply write:

```rust
pub trait StepExecutor: Send + Sync {
    async fn execute_prepared(&self, requests: Vec<PreparedRequest>) -> Vec<Result<()>>;
}
```

Callers use generics: `fn run<E: StepExecutor>(e: E)` rather than `fn run(e: Box<dyn StepExecutor>)`. The compiler monomorphizes, inlines, and produces tight code with no boxed futures.

Downside: you lose type erasure. Everywhere `StepExecutor` appears in a signature, the concrete impl leaks through as a generic parameter. If you were using `dyn` specifically to hide the impl from downstream code (test vs production, feature-gated variants), that hiding is gone.

### 6.3 Both, via a helper `Boxed<T>` wrapper

The pattern `tower::util::BoxService` uses: a thin wrapper that implements the trait by boxing the associated future at each call. You define the trait with the native/associated form, then provide a `Boxed` adapter for the erasure case.

More code to maintain, but you get both generic and dyn dispatch from the same abstraction.

### 6.4 Returning `impl Future` directly (no dyn, no macro)

```rust
pub trait StepExecutor {
    fn execute_prepared(&self, requests: Vec<PreparedRequest>)
        -> impl Future<Output = Vec<Result<()>>> + Send;
}
```

Available since 1.75. Equivalent to `async fn` in a trait. Same dyn-incompatibility. Same monomorphization behavior.

---

## 7. How to read a method written in the manual form

When you encounter code like this in a codebase:

```rust
fn do_thing(
    &self,
    arg: SomeArg,
) -> Pin<Box<dyn Future<Output = Result<Thing>> + Send>> {
    let state = self.state.clone();
    let client = self.client.clone();

    Box::pin(async move {
        let s = state.lock().unwrap();
        client.request(s.derive_key(&arg)).await
    })
}
```

Read it as:

1. **Signature**: "This is `async fn do_thing(&self, arg: SomeArg) -> Result<Thing>`, written in the dyn-compatible desugared form. The returned future is `Send + 'static` - spawnable, cross-thread-safe, owns its data."
2. **The field clones**: "These are the captures the future needs. They are cloned because the future is `'static` and cannot borrow `self`."
3. **`Box::pin(async move { ... })`**: "This is the function body of the `async fn`. It allocates one pinned future on the heap and erases its concrete type."
4. **The async block's contents**: the real logic.

Once you have done this translation a few times, the extra ceremony recedes into background pattern. The first time, it looks like ritual; the second time, it looks like the compiler speaking through the code.

---

## 8. When the pattern is right and when it is not

Use the manual `Pin<Box<dyn Future + Send>>` form when:

- You need a trait object (`Box<dyn Trait>`, `&dyn Trait`, `Arc<dyn Trait>`) with async methods.
- You want to avoid the `async-trait` proc-macro dependency for a small number of methods.
- You want the desugaring explicit in the source so readers can inspect the lifetime and `Send` bounds directly.

Use `#[async_trait]` when:

- You have many async trait methods and the repetition overhead is real.
- You want the sugared `async fn` reading experience across a team that's less practiced in manual futures.
- You don't mind one proc-macro dep.

Skip the dyn form entirely (use generics + `async fn in traits`) when:

- You never need `dyn Trait`.
- All your trait consumers are generic code that monomorphizes.
- You want peak performance - no per-call boxing, no vtable indirection.

Use associated futures (`type Future: Future<Output = …>;`) when:

- You are writing a library that wants to support both generic and boxed use without committing to one, at the cost of more scaffolding.
- You have performance-critical hot paths where the per-call `Box::pin` allocation measurably matters.

---

## 9. Quick reference

```rust
// Dyn-safe async method, manual desugar:
fn foo(&self, x: T) -> Pin<Box<dyn Future<Output = R> + Send>> {
    let state = self.state.clone();       // clone captures (no &self in body)
    Box::pin(async move {
        state.do_work(x).await
    })
}

// Dyn-safe async method, borrowing self:
fn foo<'a>(&'a self, x: T) -> Pin<Box<dyn Future<Output = R> + Send + 'a>> {
    Box::pin(async move {
        self.state.do_work(x).await         // can touch self; future tied to 'a
    })
}

// #[async_trait] equivalent - compiler expands to the dyn-safe form:
#[async_trait]
trait Foo {
    async fn foo(&self, x: T) -> R;
}

// Native async fn in trait (Rust 1.75+) - works for generics, NOT for dyn:
trait Foo {
    async fn foo(&self, x: T) -> R;
}

// Tower-style associated future - no box by default:
trait Foo {
    type Future: Future<Output = R> + Send;
    fn foo(&self, x: T) -> Self::Future;
}
```

---

## 10. Takeaway

`async fn` in traits and `Box<dyn Trait>` are two features that Rust individually supports and jointly conflict. Every implementation of an async trait method produces a different anonymous future type; vtables cannot dispatch over anonymous types; therefore dyn-safe async methods must return a concrete, type-erased representation of a future. The only such representation the language offers is `Pin<Box<dyn Future<Output = T> + Send>>` - a pinned, boxed, erased trait object of `Future`.

Everything else in the manual form is a consequence of that single decision:

- The `Box::pin(async move { ... })` wrap produces the erased value.
- The clone-before-move on `Arc`-wrapped fields is because the future must be `'static` (no lifetime parameter on the return type) and therefore cannot capture `&self`.
- The `#[async_trait]` macro expands to exactly this shape; reading it as generated code makes the behavior predictable.

Once the return type is recognized for what it is - the compiler's recommended lowering for "async in a dyn trait" - the rest of the ceremony reads as mechanics, not cleverness.

---

## References

### Crates

- [`tower`](https://docs.rs/tower) ([repo](https://github.com/tower-rs/tower)) - modular service/middleware abstraction built around the `Service` trait with an associated future type. Source of the "tower style" in §6.1.
- [`tower::util::BoxService`](https://docs.rs/tower/latest/tower/util/struct.BoxService.html) - the adapter that boxes the future at the dyn boundary, so a tower service can be used behind `dyn`.
- [`async-trait`](https://docs.rs/async-trait) ([repo](https://github.com/dtolnay/async-trait)) - the proc macro that expands sugared `async fn` in traits into the manual `Pin<Box<dyn Future + Send>>` form.
- [`hyper`](https://hyper.rs/) and [`tonic`](https://github.com/hyperium/tonic) - HTTP and gRPC libraries built on top of `tower::Service`.

### Language and standard library

- [Rust 1.75 release notes](https://blog.rust-lang.org/2023/12/28/Rust-1.75.0.html) - the release that stabilized `async fn` in traits and `-> impl Trait` in trait methods (RPITIT).
- [Announcing `async fn` and return-position `impl Trait` in traits](https://blog.rust-lang.org/inside-rust/2023/05/03/stabilizing-async-fn-in-trait.html) - the lang team post explaining the feature, its limitations, and explicitly why it does not work with `dyn Trait`.
- [E0038: the trait is not dyn compatible](https://doc.rust-lang.org/error_codes/E0038.html) - the error code emitted when a trait cannot be made into a trait object, with the full list of reasons.
- [Reference: object safety / dyn compatibility](https://doc.rust-lang.org/reference/items/traits.html#object-safety) - the language reference's definition of which traits can be used as `dyn Trait`.
- [`std::pin`](https://doc.rust-lang.org/std/pin/) - module documentation for `Pin`, including why self-referential futures need pinning.
- [`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html) - the trait being boxed.

### Background reading

- [The Async Rust Book](https://rust-lang.github.io/async-book/) - foundational material on `Future`, `async`/`await`, executors, and pinning.
- [Tokio tutorial: Spawning](https://tokio.rs/tokio/tutorial/spawning) - explains the `Send + 'static` requirement for `tokio::spawn`, which is the same constraint that drives the clone-before-`async move` idiom in §4.

