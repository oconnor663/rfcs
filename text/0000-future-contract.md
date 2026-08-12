- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Clarify and document the contract requirements of the `Future::poll` method, in
particular that a future should be polled or dropped promptly when it requests
a wakeup. More generally, document what it means to cancel a future.

There are widely used patterns in async Rust that violate this "poll promptly
requirement", including `select!`-by-reference and `StreamExt::next`. This RFC
identifies several, but it avoids endorsing specific changes beyond the
`Future` docs. Some changes will need follow-up RFCs of their own and/or time
to experiment with alternatives in the ecosystem.

## Motivation
[motivation]: #motivation

Consider this contrived deadlock example ([playground link][foo1]):

[foo1]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Apoll%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+future1+%3D+pin%21%28foo%28%29%29%3B%0A++++_+%3D+poll%21%28%26mut+future1%29%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
async fn foo() {
    // Acquire a global lock, sleep briefly, and release it.
    static LOCK: Mutex<()> = Mutex::const_new(());
    let _guard = LOCK.lock().await;
    sleep(Duration::from_millis(10)).await;
}

#[tokio::main]
async fn main() {
    let mut future1 = pin!(foo());
    _ = poll!(&mut future1);
    foo().await; // Deadlock!
}
```

`poll!` puts `future1` in the state where it's acquired `LOCK` and started
sleeping, and then the second call to `foo` starts waiting for `LOCK`. We don't
poll `future1` again or drop it while we're waiting, so this is a deadlock.

The `poll!` macro isn't common in practice, and a more realistic version of
this example would use `select!`. However, to emphasize that this problem is
broader than `select!`, let's use `timeout` instead ([playground link][foo2]):

[foo2]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+future1+%3D+pin%21%28foo%28%29%29%3B%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+%26mut+future1%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D%0A>

```rust
#[tokio::main]
async fn main() {
    let mut future1 = pin!(foo());
    _ = timeout(Duration::from_millis(1), &mut future1).await;
    foo().await; // Deadlock!
}
```

This still isn't very realistic, because nothing forces us to use `pin!` here.
(Passing `future1` to `timeout` by value would fix the deadlock, because we'd
drop it when the timeout expires.) What usually forces us to use `pin!` is when
we're driving a future in a loop. Let's put the loop in, and while we're at it,
let's add a couple layers of abstraction around `foo` ([playground
link][foo3]):

[foo3]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+mut+bar_future+%3D+pin%21%28bar%28%29%29%3B%0A++++let+tick+%3D+Duration%3A%3Afrom_millis%285%29%3B%0A++++while+timeout%28tick%2C+%26mut+bar_future%29.await.is_err%28%29+%7B%0A++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++baz%28%29.await%3B%0A++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++%7D%0A%7D>

```rust
async fn bar() {
    foo().await;
}

async fn baz() {
    foo().await;
}

#[tokio::main]
async fn main() {
    // While `bar` is running, call `baz` every 5 ms.
    let mut bar_future = pin!(bar());
    let tick = Duration::from_millis(5);
    while timeout(tick, &mut bar_future).await.is_err() {
        baz().await; // Deadlock!
    }
}
```

Now we can see how these things happen in practice. To complete the illusion,
imagine that `foo`, `bar`, `baz`, and `main` are all defined in different
crates. The lock is private to `foo`, but the `main` crate doesn't depend on
`foo` directly, and the author of `main` has never heard of `foo`. Who's "at
fault" for a deadlock like this?

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `Future` docs

> The focus of this RFC is the new/clarified polling responsibility in the
> subsection "The `poll` method" below. The following is an expanded general
> intro to lead into that section, both to avoid presenting it in a vacuum, and
> to make the RFC slightly more accessible to folks who haven't written a ton
> of async Rust.

A future represents an asynchronous computation and the value it might
eventually return. The most common way to create a future is to call an `async
fn`. Often we `.await` a future without giving it a name, like this:

```rust
async fn double(x: u32) -> u32 {
    2 * x
}

assert_eq!(double(42).await, 84);
```

If we break that last line into three lines, we can see some of the temporary
values involved:

```rust
let my_future = double(42);
let my_output = my_future.await;
assert_eq!(my_output, 84);
```

Intuitively, `u32` is the "return type" of `double`, but here we see that the
expression `double()` actually evaluates to a future, and we get the `u32` when
we `.await` that future. We can look at `double` in two different ways: it's an
`async fn` that returns `u32`, but it's also a regular function that returns a
future whose output is a `u32`. That's what it means to be an `async fn`.

Normally the compiler generates the "regular function that returns a future"
for us, and we don't need to write it ourselves. But we can write it if we
like. The following `fn double` is a drop-in replacement for `async fn double`
above:

```rust
struct Foo(u32);

impl Future for Foo {
    type Output = u32;

    fn poll(self: Pin<&mut Self>, _: &mut Context) -> Poll<u32> {
        Poll::Ready(self.0 * 2)
    }
}

fn foo(x: u32) -> Foo {
    Foo(x)
}

assert_eq!(double(42).await, 84);
```

This version of `foo` explicitly returns a `Foo` future. It behaves like an
`async fn`, and we call it and `.await` it the same way. Implementing the
`Future` trait is what makes `Foo` a future, and the core of the `Future` trait
is the `poll` method.

#### The `poll` method

The `Future::poll` method is called with a `Context` and (inside that) a
`Waker`, and it returns either `Poll::Ready(_)` or `Poll::Pending`. `poll` has
three responsibilities:

1. Perform as much of the future's remaining work as possible, while returning
   promptly and not blocking the caller.

2. If the future's work is finished, return `Ready(_)` containing its
   output.

3. Otherwise, arrange to invoke the caller's `Waker` when the future should be
   polled again, and return `Pending`.

In the common case of a compiler-generated future that represents an `async`
block or function, these responsibilities mean that `poll` will:

1. Begin or resume executing the body. If control reaches an `.await` of an
   inner future, and polling that inner future returns `Ready(_)`, continue
   execution without returning.

2. If control reaches an exit (end-of-scope, `return`, or a short-circuiting
   `?`), return `Ready(_)` wrapping the body's return value.

3. If control reaches an `.await` of an inner future, and polling that inner
   future returns `Pending`, stop executing the body and return `Pending`. Rely
   on the inner future to trigger a wakeup when it should be polled again.
   (Some low-level futures use operating system APIs like `epoll` to implement
   wakeups, but `async fn` futures almost always delegate this.)

The `poll` method also imposes three responsibilities on its caller:

1. After `poll` returns `Ready(_)`, the caller should not call `poll` again and
   should drop the future promptly. Further calls to `poll` may panic or
   otherwise misbehave (within the bounds of safe code).

2. If the last call to `poll` returned `Pending`, and the `Waker` passed to
   that call is later invoked, and the future hasn't been dropped in the
   meantime, the caller should `poll` again promptly.

3. If `poll` panics without terminating the whole process, the caller should
   not call `poll` again and should drop the future promptly.

Here's an example of a `Future` implementation that fails that second
requirement, a.k.a. "the `Poll::Pending` rule":

```rust
pub struct CoinFlip<Fut>(#[pin] Fut); // TODO: a standard way to do pin projection

impl<Fut: Future> Future for CoinFlip<Fut> {
    type Output = Fut::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Fut::Output> {
        if rand::random() {
            self.project().0.poll(cx) // TODO: a standard way to do pin projection
        } else {
            Poll::Pending
        }
    }
}
```

The problem here is that `random` might be true the first time, polling the
inner `Fut` and letting it register wakeups, but then it might be false the
second time when those wakeups trigger, failing to poll `Fut` promptly. This
mistake tends to cause hangs and deadlocks, and `CoinFlip` would be "at fault"
for those bugs. There are three ways we can fix it:

1. Return `Ready` in the `else` branch, which requires the caller to drop
   `CoinFlip` promptly. This would also mean changing the `Output` type to
   `Option<_>`, or maybe adding a `Default` bound.
2. Drop the inner `Fut` in the `else` branch before returning `Pending`. We'd
   need to make `self.0` an `Option<_>` or similar.
3. Panic in the `else` branch. This probably isn't what users want, but it's
   technically correct, the best kind of correct.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

## Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
