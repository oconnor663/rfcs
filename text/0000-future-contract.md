- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Clarify and document the contract requirements of the `Future::poll` method, in
particular that a future should be polled or dropped promptly when it requests
a wakeup. More generally, document what it means to cancel a future.

There are widely used patterns in async Rust that violate that "poll promptly
requirement", including `select!`-by-reference and `StreamExt::next`. This RFC
identifies several of them, but it avoids endorsing specific changes beyond the
`Future` docs. Some will need follow-up RFCs of their own and/or time to
experiment with alternatives in the ecosystem.

## Motivation
[motivation]: #motivation

Consider this contrived deadlock example ([playground link][foo1]):

[foo1]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Apoll%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+future1+%3D+pin%21%28foo%28%29%29%3B%0A++++_+%3D+poll%21%28future1%29%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
async fn foo() {
    // Acquire a global lock, sleep briefly, and release it.
    static LOCK: Mutex<()> = Mutex::const_new(());
    let _guard = LOCK.lock().await;
    sleep(Duration::from_millis(10)).await;
}

#[tokio::main]
async fn main() {
    let future1 = pin!(foo());
    _ = poll!(future1);
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

This still isn't very realistic, because nothing is making us use `pin!` here.
(And passing `future1` to `timeout` by value would fix the deadlock, because
then we'd drop it when the timeout expires.) Usually we only need `pin!` when
we're driving a future in a loop. Let's put the loop in, and while we're add it
we'll add a couple layers of abstraction around `foo` ([playground
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

Now we're starting to see how these things happen in practice. To complete the
illusion, imagine that `foo`, `bar`, `baz`, and `main` are all defined in
different crates. The lock is private to `foo`, but the `main` crate doesn't
depend on `foo`. The author of `main` might never even have heard of `foo`, to
say nothing of its private implementation details. So, who's "at fault" for a
deadlock like this?

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `Future` docs

> The focus of this RFC is the docs changes in the subsection "The `poll`
> method" below. The following is a general intro that might lead into that
> section, both to avoid presenting it in a vacuum, and to make this whole RFC
> slightly more accessible to folks who haven't written a ton of async Rust.

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

Intuitively `u32` is the "return type" of `double`, but what we're seeing here
is that the expression `double()` actually evaluates to a future, and we get a
`u32` when we `.await` that future. We can look at `double` in two different
ways: it's an `async fn` that returns `u32`, but it's also a regular function
that returns a future _whose output_ is a `u32`. That's what it means to be an
`async fn`.

Normally the compiler generates the "regular function that returns a future"
part for us, and we don't need to write it out ourselves. But we can write it
if we like. The following `fn double` is equivalent to `async fn double` above:

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

Here the `foo` function returns a `Foo` future, so it behaves like an `async
fn`, and we can `.await` it the same way. What makes `Foo` a future is that it
implements the `Future` trait. And the core of the `Future` trait is the `poll`
method.

#### The `poll` method

> As mentioned above, here's where the important changes in this RFC begin.

...

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
