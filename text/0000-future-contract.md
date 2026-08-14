- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Clarify and document the contract requirements of the [`Future::poll`] method, in
particular that a future should be polled or dropped promptly when it requests
a wakeup. Also, document what it means to cancel a future.

[`Future::poll`]: https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll

There are widely used patterns in async Rust that violate this "poll promptly
requirement", including [`select!`]-by-reference and [`StreamExt::next`]. This
RFC identifies several, but it avoids endorsing specific changes beyond the
`Future` docs.

[`StreamExt::next`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.next

## Motivation
[motivation]: #motivation

Cancellation is a well-established if under-documented feature of async Rust.
But it has an obscure cousin that's not documented at all, and maybe not even a
feature, what we might call "pausing" (complimentary) or "snoozing"
(pejorative). Pausing is almost never explicit,[^dioxus] but it's common and
surprisingly easy to snooze a future _implicitly_. This can cause hangs and
deadlocks, most famously ["Futurelock"][futurelock]. Here's a minimal example
([playground link][foo1]):

[^dioxus]: The only widely-used counterexample might be the Dioxus framework,
    which [provides a `pause` method][dioxus_docs] and sometimes [calls it
    automatically][dioxus_src].

[dioxus_docs]: https://docs.rs/dioxus/0.7.10/dioxus/prelude/struct.UseFuture.html
[dioxus_src]: https://github.com/DioxusLabs/dioxus/blob/v0.7.10/packages/hooks/src/use_future.rs#L63-L72

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

The [`poll!`] macro calls `Future::poll` exactly once, driving `future1` to the
point where it's acquired `LOCK` and started sleeping. The second call to `foo`
tries to take the same lock, but nothing polls `future1` during that `.await`,
so the result is a deadlock.

[`poll!`]: https://docs.rs/futures/latest/futures/macro.poll.html

We don't often use `poll!` outside of tests, so let's make this example more
realistic. The most common way we snooze futures in practice is [`select!`],
and that's how ["Futurelock"][futurelock] happened. But to emphasize that this
is a broader problem, let's use [`timeout`] instead ([playground link][foo2]):

[`timeout`]: https://docs.rs/tokio/latest/tokio/time/fn.timeout.html

[foo2]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+future1+%3D+pin%21%28foo%28%29%29%3B%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+future1%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
#[tokio::main]
async fn main() {
    let future1 = pin!(foo());
    _ = timeout(Duration::from_millis(1), future1).await;
    foo().await; // Deadlock!
}
```

This still isn't very realistic; it would be simpler and more correct to [pass
`future1` to `timeout` by value][foo_by_value] instead of pinning it like this.
Driving futures in a loop is usually what forces us to `pin!` things, so let's
add a loop. Let's also add a couple layers of abstraction around `foo`, for
dramatic effect ([playground link][foo3]):

[foo_by_value]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+future1+%3D+foo%28%29%3B%0A++++%2F%2F+Passing+%60future1%60+to+%60timeout%60+by+value+means+that+it+drops+when%0A++++%2F%2F+the+timeout+expires%2C+releasing+%60LOCK%60+and+fixing+the+deadlock.%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+future1%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...and+also+here%21%22%29%3B%0A%7D>

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

Now this is starting to look like code someone might actually write. To
complete the illusion, imagine that `foo`, `bar`, `baz`, and `main` are all
defined in different crates. The lock is private to `foo`, but `main` doesn't
depend on `foo` directly. Maybe the author of `main` has never heard of `foo`.
Who's "at fault" for a deadlock like this?

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### important new text in the `Future` docs

> This new text is the focus of this whole RFC. It's presented here in
> isolation, for readers who want to see the important part first and who don't
> need context. It's repeated in context in the following section.

The `poll` method also imposes two responsibilities on its caller:

1. After `poll` returns `Ready(_)`, the caller should not call `poll` again and
   should **drop the future promptly**. Further calls to `poll` may panic or
   otherwise misbehave (within the bounds of safe code).

2. If the last call to `poll` returned `Pending`, and the `Waker` passed to
   that call is later invoked, and the future hasn't been dropped in the
   meantime, the caller should **`poll` again promptly.**

### expanded `Future` docs

> The this section repeats the important text above, but in the context of an
> an expanded intro to `Future`, the way new learners might encounter it. This
> is both to avoid presenting the important part entirely in a vacuum, and also
> to make this RFC slightly more accessible to folks who haven't written a ton
> of async Rust. The other details in this intro aren't new in this RFC, and a
> PR adding docs like this might not need to go through the RFC process.

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
expression `double()` actually evaluates to a future, and we get a `u32` when
we `.await` that future. We can look at `double` in two different ways: it's an
`async fn` that returns `u32`, but it's also a regular function that returns a
future whose output is `u32`. That's what it means to be an `async fn`.

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
   on the inner future to invoke the `Waker` when it should be polled again.
   Some low-level futures use threads or operating system APIs like [`epoll`]
   to implement wakeups, but `async fn` futures almost always delegate this
   responsibility.

[`epoll`]: https://en.wikipedia.org/wiki/Epoll

> Here's the important new part that's excerpted in the previous section.

The `poll` method also imposes two responsibilities on its caller:

1. After `poll` returns `Ready(_)`, the caller should not call `poll` again and
   should **drop the future promptly**. Further calls to `poll` may panic or
   otherwise misbehave (within the bounds of safe code).

2. If the last call to `poll` returned `Pending`, and the `Waker` passed to
   that call is later invoked, and the future hasn't been dropped in the
   meantime, the caller should **`poll` again promptly.**

> Everything that follows, including the "Cancellation" section below, assumes
> those new rules and elaborates on them.

Here's an example of a `Future` implementation that fails that second
requirement, a.k.a. the "`Poll::Pending` rule":

```rust
pub struct CoinFlip<Fut>(Pin<Box<Fut>>); // TODO: a standard way to do unboxed pin projection?

impl<Fut: Future> Future for CoinFlip<Fut> {
    type Output = Fut::Output;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<Fut::Output> {
        if rand::random() {
            self.0.as_mut().poll(cx)
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

- Return `Ready` in the `else` branch, which requires the caller to drop
  `CoinFlip` promptly. This would also mean changing the `Output` type to
  `Option<_>`, or maybe adding a `Default` bound.
- Drop the inner `Fut` in the `else` branch before returning `Pending`. We'd
  need to make `self.0` an `Option<_>` or similar.
- Panic in the `else` branch. This probably isn't what users want, but it's
  technically correct, the best kind of correct.

#### Cancellation

Unlike threads, which have a life of their own once they start running, a
future only makes progress when something polls it. A future's owner can
effectively pause its execution by _not_ polling it again. However, the
"`Poll::Pending` rule" above tightly constrains our options here: Whoever last
called `poll` is supposed to make sure that `poll` gets called again promptly
after a wakeup, unless the future is dropped in the meantime. If a wakeup
arrives, and a future's owner doesn't want to poll it -- say because it's
exceeded a timeout, or because its output is no longer needed -- they must drop
it promptly. When we drop a still-pending future like this, we call that
"cancellation".

There's nothing particularly special about cancelling a future compared to
dropping any other Rust object. Its `Drop::drop` function runs (if any), and
then the `Drop::drop` functions of its fields run (if any), all as usual.
Importantly, this does include local variables in `async` blocks and functions,
which are fields in their compiler-generated futures. It's good that we don't
leak those, of course. But perhaps the most important thing to understand about
cancellation is less what it _does_, and more that we can be _forced to do it_.
The `Poll::Pending` rule requires every future to actively participate in what
we might call the "`Waker` protocol" between its caller and any child futures
it might contain. When a future is polled, it can poll its own children in
turn, or it can cancel them by dropping them (either directly or indirectly,
e.g. by returning `Ready` and trusting the caller to drop it), but it can't
silently ignore a child's wakeup.

This turns out to be essential for futures that can acquire locks or other
exclusive resources. If a future is supposed to hold a lock for a short time,
the programmer needs to consider how it might release that lock sooner if it's
cancelled, or maybe a bit later because of timer slack or CPU load. But the
programmer doesn't need to worry about the caller's caller's caller pausing
execution and thereby (accidentally, unknowingly) holding the lock _forever_.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### What exactly does "promptly" mean?

The usual expected behavior is that an implementation of `poll` should call
`poll` on all of its child futures (if any) before returning. However, there
are other valid ways to do things. Some async runtimes may [track an execution
"budget"][budget], such that some of their futures return `Pending` earlier
than usual once that budget is used up. In that case they arrange to get polled
again as soon as other tasks have had a chance to run, and those other tasks'
`poll` functions are also subject to the "return promptly" requirement, so any
work deferred to the re-poll can still be considered prompt. Some futures might
also offload their work to other threads, and their `poll` function might
return without waiting for a completion notification from those threads.

[budget]: https://tokio.rs/blog/2020-04-preemption

What these cases have in common is that, barring program exit or the power
going out, future progress is guaranteed. There's no (legitimate) way for user
code to stop the runtime from working through its task list or freeze a private
worker thread. This is in contrast to the deadlocks in the Motivation section,
where a future is suspended across some arbitrary bit of user code that isn't
guaranteed to ever finish.

A fully formal definition of "promptly" will probably end up with somewhat
unsatisfying wording like "a finite period of time". Consider this excerpt from
[the C++ atomic memory model][finite]:

> An implementation should ensure that the last value (in modification order)
> assigned by an atomic or synchronization operation will become visible to all
> other threads **in a finite period of time**.

[finite]: https://github.com/cplusplus/draft/blob/4358c6f6856ac8b392b601f820bce7bb134cbeed/source/basic.tex#L7291-L7293

The atomic memory model is concerned with operations that complete in
_nanoseconds_, but nonetheless it's hard to give a specific number of
nanoseconds, or even a more abstract bound like "ticks" or "steps", without
getting into details of the hardware that the standard doesn't want to
constrain. Similarly, async Rust needs to accommodate all sorts of runtimes,
evented IO models, and OS environments, and that makes it hard for the formal,
general rules to say much about anything that goes on under the hood. Using
terms like "promptly" in our documentation is useful for building intuition
about how async programs work, and for helping different layers of the stack
understand each other's intent, but in the end we'll probably define them in
the negative: If something never happens at all, then clearly it didn't happen
promptly.

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

### Should we allow a delay between creation and polling?

In other words, should the following be allowed, or should it e.g. fail Clippy?

```rust
let future1 = foo();
let future2 = foo();
future1.await;
future2.await;
```

We could say that `future2` is snoozed here across the first await. On the
other hand, `future2` has never been polled (or even pinned), and it's not
likely to be holding onto any exclusive resources in its initial state. We
could imagine giving `foo` e.g. a `MutexGuard` argument, but in that case the
caller could clearly see what's going on. To create a true "nonlocal reasoning"
problem, we'd need to write `foo` in a sync-then-async style, like this:

```rust
fn foo() -> impl Future<Output = ()> {
    static LOCK: Mutex<()> = Mutex::const_new(());
    // Try to acquire `LOCK` synchronously. If we get it, the returned future takes ownership of the guard.
    let mut _guard = LOCK.try_lock().ok();
    async move {
        if _guard.is_none() {
            _guard = Some(LOCK.lock().await);
        }
        sleep(Duration::from_millis(10)).await;
    }
}
```

This style is uncommon, and there might not be any examples in the wild that
actually combine this style with an exclusive resource acquired in the sync
phase. On the other hand, there are published `Future` extension methods (e.g.
[`delay`] in `async-std`) that are only correct if delayed initial polling is
acceptable.

[`delay`]: https://docs.rs/async-std/latest/async_std/prelude/trait.FutureExt.html#method.delay

### Should we document a requirement for when `poll` panics?

In addition to the "`Poll::Ready` rule" (drop promptly) and the
"`Poll::Pending` rule" (poll again promptly after a wakeup), we could consider
a third rule regarding panicking. For example:

> If `poll` panics without terminating the whole process, the caller should not
> call `poll` again and should drop the future promptly.

On the other hand, futures that use `catch_unwind` and therefore need to worry
about this are extremely rare, and this isn't really a pressing concern for the
ecosystem. We could also consider folding this into the `Poll::Ready` rule,
since the requirement is the same.

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

[futurelock]: https://rfd.shared.oxide.computer/rfd/0609
[snooze]: https://jacko.io/snooze.html
[`select!`]: https://docs.rs/tokio/latest/tokio/macro.select.html
