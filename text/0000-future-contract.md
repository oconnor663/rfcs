- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Clarify and document the contract requirements of the [`Future::poll`] method,
in particular that a future should be polled or dropped promptly when it
requests a wakeup. In other words, we can cancel a future at any time, but we
should never "pause" a future.

[`Future::poll`]: https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll

There are widely used patterns that violate this rule, including
[`select!`]-by-reference and [`StreamExt::next`]. This RFC identifies several,
but it avoids endorsing specific changes beyond the `Future` docs. The goal is
to agree that these contract violations are bugs and that we can and should fix
them, but deciding how exactly to fix or replace each problematic pattern is
deferred to follow-up RFCs and/or the crates ecosystem.

[`StreamExt::next`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.next

## Motivation
[motivation]: #motivation

Cancellation and pausing are distinct features of async Rust. Regular,
synchronous Rust doesn't let us kill or suspend threads, because doing either
of those things tends to cause deadlocks.[^deprecated] Async cancellation
solves the deadlock problem(!) by dropping cancelled futures, which
automatically releases any locks they might be holding. But async pausing
doesn't solve the deadlock problem, and that makes it arguably more of a bug
than a feature. For a case study in how confusing and non-local these deadlocks
can be in practice, see ["Futurelock"] (Oxide, October 2025).

[^deprecated]: Lots of languages have old APIs for killing or suspending
    threads that are now deprecated ([Java][java], [C#][c_sharp],
    [Python][python]). The fundamental problem is that killing threads doesn't
    respect language-level cleanup constructs like `try`/`finally` or
    destructors. If we kill a thread in Rust, we leak everything that thread
    owned, including lock guards. Pausing a thread doesn't leak anything per
    se, but if the paused thread is holding a lock, and the thread that's
    supposed to unpause it touches the same lock in the meantime, we get
    similar deadlocks.

[java]: https://docs.oracle.com/javase/8/docs/technotes/guides/concurrency/threadPrimitiveDeprecation.html
[c_sharp]: https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/5.0/thread-abort-obsolete
[python]: https://docs.python.org/3/c-api/threads.html#c.PyThread_exit_thread

Another problem with async pausing is that, although we almost never do it
explicitly,[^dioxus] we often do it implicitly, and it's surprisingly easy to
do it accidentally. "Futurelock" was caused by unintended pausing in a
[`select!`] loop, and async streams have been [battling pausing bugs][barbara]
for years. These mistakes are invisible unless you know exactly what you're
looking for.

Let's look at an example to get a sense of how things go wrong and where we
might be able to intervene. We'll start with something minimal and contrived,
and then we'll expand it to look more realistic. Here's the minimal version
([playground link][foo1]):

[^dioxus]: The only widely-used counterexample might be the Dioxus framework,
    which [provides a `pause` method][dioxus_docs] and sometimes [calls it
    automatically][dioxus_src].

[dioxus_docs]: https://docs.rs/dioxus/0.7.10/dioxus/prelude/struct.UseFuture.html#method.pause
[dioxus_src]: https://github.com/DioxusLabs/dioxus/blob/v0.7.10/packages/hooks/src/use_future.rs#L63-L72

[foo1]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Apoll%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+foo_future+%3D+pin%21%28foo%28%29%29%3B%0A++++_+%3D+poll%21%28foo_future%29%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
async fn foo() {
    // Acquire a global lock, sleep briefly, and release it.
    static LOCK: Mutex<()> = Mutex::const_new(());
    let _guard = LOCK.lock().await;
    sleep(Duration::from_millis(10)).await;
}

#[tokio::main]
async fn main() {
    let foo_future = pin!(foo());
    _ = poll!(foo_future);
    foo().await; // Deadlock!
}
```

The [`poll!`] macro calls `Future::poll` exactly once, driving `foo_future` to
the point where it's acquired `LOCK` and started sleeping. The second call to
`foo` tries to take the same lock, but nothing polls `foo_future` during that
`.await`, and the result is a deadlock.

[`poll!`]: https://docs.rs/futures/latest/futures/macro.poll.html

`poll!` is kind of obscure, and the usual suspect in practice is [`select!`].
That's [how "Futurelock" happened][futurelock_select]. But we don't have to use
macros (besides `pin!`) to demonstrate this. Here's a version using [`timeout`]
([playground link][foo2]):

[futurelock_select]: https://github.com/oxidecomputer/omicron/blob/58f95ded7eed49fd30659035c5c16b5bb9e63a76/nexus/src/app/background/tasks/support_bundle_collector.rs#L516-L550
[`timeout`]: https://docs.rs/tokio/latest/tokio/time/fn.timeout.html

[foo2]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+foo_future+%3D+pin%21%28foo%28%29%29%3B%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+foo_future%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
#[tokio::main]
async fn main() {
    let foo_future = pin!(foo());
    _ = timeout(Duration::from_millis(1), foo_future).await;
    foo().await; // Deadlock!
}
```

This 1 ms timeout always expires before the 10 ms sleep in `foo` finishes, so
again `foo_future` only gets polled once. But this still isn't very realistic.
It would make more sense (and fix the deadlock) to [pass `foo_future` to
`timeout` by value][foo_by_value] instead of pinning it like this. Driving
futures in a loop is usually what forces us to `pin!` things, so let's add a
loop. We'll also add a couple layers of abstraction around `foo`, for dramatic
effect ([playground link][foo3]):

[foo_by_value]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+foo_future+%3D+foo%28%29%3B%0A++++%2F%2F+Passing+%60foo_future%60+to+%60timeout%60+by+value+means+that+it+drops+when%0A++++%2F%2F+the+timeout+expires%2C+releasing+%60LOCK%60+and+fixing+the+deadlock.+Note%0A++++%2F%2F+that+we+also+don%27t+need+to+%60pin%21%60+it+in+this+case.%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+foo_future%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...and+also+here%21%22%29%3B%0A%7D>

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

This is more realistic. To complete the illusion, imagine that `foo`, `bar`,
`baz`, and `main` are all defined in different crates. The lock is private to
`foo`, but `main` doesn't depend on `foo` directly. Maybe the author of `main`
has never even heard of `foo`. Who's "at fault" for a deadlock like this?

Let's look more closely at the sequence of events. If we [add some
prints][squawk], we can see that `main` gets polled three times. The first
wakeup is at 0 ms when control first enters `main`, the second is at 5-6 ms
when the `timeout` expires, and the third is at 10-11 ms when the `sleep` in
`bar` (that is, in `foo`) completes.[^slack] Control in `main` is in the
`baz().await` expression at that point, so the `baz` future gets polled again,
even though it didn't request a wakeup. That's not in and of itself a problem,
since futures are expected to tolerate over-polling. The problem is that the
`bar` future _did_ request a wakeup, but it did _not_ get polled. In the body
of `foo`, control enters the 10 ms sleep, but it never comes back out again,
even though all the `Waker` and timer machinery is working correctly. That's
not ok.

[squawk]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+Instant%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+main_inner%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+mut+bar_future+%3D+pin%21%28bar%28%29%29%3B%0A++++let+tick+%3D+Duration%3A%3Afrom_millis%285%29%3B%0A++++while+timeout%28tick%2C+%26mut+bar_future%29.await.is_err%28%29+%7B%0A++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++baz%28%29.await%3B%0A++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++%7D%0A%7D%0A%0A%2F%2F+Squawk+a+timestamp+every+time+%60future%60+gets+polled.%0Afn+squawk%3CFut%3A+Future%3E%28future%3A+Fut%29+-%3E+impl+Future%3COutput+%3D+Fut%3A%3AOutput%3E+%7B%0A++++let+start+%3D+Instant%3A%3Anow%28%29%3B%0A++++let+mut+future+%3D+Box%3A%3Apin%28future%29%3B%0A++++std%3A%3Afuture%3A%3Apoll_fn%28move+%7Ccx%7C+%7B%0A++++++++let+elapsed+%3D+Instant%3A%3Aelapsed%28%26start%29.as_secs_f32%28%29+*+1000.0%3B%0A++++++++println%21%28%22%5B%7Belapsed%3A.3%7D+ms%5D+POLLED%21%22%29%3B%0A++++++++future.as_mut%28%29.poll%28cx%29%0A++++%7D%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++squawk%28main_inner%28%29%29.await%3B%0A%7D>

[^slack]: Tokio's timer implementation adds ~1 ms of slack to our 5 ms timeout
    and our 10 ms sleep.

TODO: finish this section

[`join_maybe` playground][join_maybe]

[join_maybe]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Afuture%3A%3AMaybeDone%3B%0Ause+std%3A%3Apin%3A%3APin%3B%0Ause+std%3A%3Atask%3A%3A%7BContext%2C+Poll%7D%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Afn+join_maybe%3CLeft%3A+Future%2C+Right%3A+Future%3E%28left%3A+Left%2C+right%3A+Right%29+-%3E+JoinMaybe%3CLeft%2C+Right%3E+%7B%0A++++JoinMaybe+%7B%0A++++++++left%2C%0A++++++++right%3A+MaybeDone%3A%3AFuture%28right%29%2C%0A++++%7D%0A%7D%0A%0Apin_project_lite%3A%3Apin_project%21+%7B%0A++++struct+JoinMaybe%3CLeft%3A+Future%2C+Right%3A+Future%3E+%7B%0A++++++++%23%5Bpin%5D%0A++++++++left%3A+Left%2C%0A++++++++%23%5Bpin%5D%0A++++++++right%3A+MaybeDone%3CRight%3E%2C%0A++++%7D%0A%7D%0A%0Aimpl%3CLeft%3A+Future%2C+Right%3A+Future%3E+Future+for+JoinMaybe%3CLeft%2C+Right%3E+%7B%0A++++type+Output+%3D+%28Left%3A%3AOutput%2C+Option%3CRight%3A%3AOutput%3E%29%3B%0A%0A++++fn+poll%28self%3A+Pin%3C%26mut+Self%3E%2C+cx%3A+%26mut+Context%29+-%3E+Poll%3CSelf%3A%3AOutput%3E+%7B%0A++++++++let+mut+this+%3D+self.project%28%29%3B%0A++++++++let+left_poll+%3D+this.left.poll%28cx%29%3B%0A++++++++_+%3D+this.right.as_mut%28%29.poll%28cx%29%3B%0A++++++++if+let+Poll%3A%3AReady%28left_output%29+%3D+left_poll+%7B%0A++++++++++++Poll%3A%3AReady%28%28left_output%2C+this.right.take_output%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Poll%3A%3APending%0A++++++++%7D%0A++++%7D%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+background_loop+%3D+async+%7B%0A++++++++loop+%7B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%285%29%29.await%3B%0A++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++baz%28%29.await%3B%0A++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++%7D%0A++++%7D%3B%0A++++join_maybe%28bar%28%29%2C+background_loop%29.await%3B%0A++++println%21%28%22...and+then+we+exit.%22%29%3B%0A%7D>

```rust
#[tokio::main]
async fn main() {
    // While `bar` is running, call `baz` every 5 ms.
    let background_loop = async {
        loop {
            sleep(Duration::from_millis(5)).await;
            baz().await;
        }
    };
    join_maybe(bar(), background_loop).await;
}
```

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Important new text in the `Future` docs

> This new text is the focus of this whole RFC. It's presented here in
> isolation, for readers who want to see the important part first and who don't
> need context. It's repeated in context in the following section.

The `poll` method also imposes two responsibilities on its caller:

1. If the last call to `poll` returned `Pending`, and the `Waker` passed to
   that call is later invoked, and the future hasn't been dropped in the
   meantime, the caller should **`poll` again promptly.**

2. After `poll` returns `Ready(_)`, the caller should not call `poll` again and
   should **drop the future promptly**. Further calls to `poll` may panic or
   otherwise misbehave (within the bounds of safe code).[^exceptions]

[^exceptions]: This is how we need to treat generic futures that we don't know
    anything about. But specific types like [`Fuse`] or [`MaybeDone`], which
    handle dropping internally and/or tolerate further calls to `poll` after
    returning `Ready`, can document their exceptions to this rule.

[`MaybeDone`]: https://docs.rs/futures/latest/futures/future/enum.MaybeDone.html
[`Fuse`]: https://docs.rs/futures/latest/futures/future/trait.FutureExt.html#method.fuse

### Expanded `Future` docs

> This section repeats the important text above, but in the context of an an
> expanded intro to `Future`, the way new learners might encounter it. This is
> both to avoid presenting the important part entirely in a vacuum, and also to
> make this RFC slightly more accessible to folks who haven't written a ton of
> async Rust.

A future represents a possibly-ongoing asynchronous computation and the value
it might eventually return. The most common way to create a future is to call
an `async fn`. Often we `.await` a future without giving it a name, like this:

```rust
async fn add_one(x: u32) -> u32 {
    x + 1
}

assert_eq!(add_one(42).await, 43);
```

If we break that last line into three lines, we can see some of the temporary
values involved:

```rust
let my_future = add_one(42);
let my_output = my_future.await;
assert_eq!(my_output, 43);
```

Intuitively, `u32` is the "return type" of `add_one`, but here we see that the
expression `add_one()` actually evaluates to a future, and we get the `u32`
when we `.await` that future. We can look at `add_one` in two different ways:
it's an `async fn` that returns `u32`, but it's also a regular function that
returns a future whose output is `u32`. That's what it means to be an `async
fn`.

Normally the compiler generates the "regular function that returns a future"
for us, and we don't need to write it ourselves. But we can write it if we
like. The following `fn add_one` is a drop-in replacement for `async fn
add_one` above:

```rust
struct AddOne(u32);

impl Future for AddOne {
    type Output = u32;

    fn poll(self: Pin<&mut Self>, _: &mut Context) -> Poll<u32> {
        Poll::Ready(self.0 + 1)
    }
}

fn add_one(x: u32) -> AddOne {
    AddOne(x)
}

assert_eq!(add_one(42).await, 43);
```

This version of `add_one` explicitly returns an `AddOne` future. It behaves
like an `async fn`, and we call it and `.await` it the same way. Implementing
the `Future` trait is what makes `AddOne` a future, and the core of the
`Future` trait is the `poll` method.

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

1. If the last call to `poll` returned `Pending`, and the `Waker` passed to
   that call is later invoked, and the future hasn't been dropped in the
   meantime, the caller should **`poll` again promptly.**

2. After `poll` returns `Ready(_)`, the caller should not call `poll` again and
   should **drop the future promptly**. Further calls to `poll` may panic or
   otherwise misbehave (within the bounds of safe code).[^exceptions]

> Everything that follows, including the "Cancellation" section below, assumes
> those new rules and elaborates on them.

Here's an example of a `Future` implementation that fails the first
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
arrives, and we don't want to poll it -- say because it's exceeded a timeout,
or because we no longer need its output -- we must drop it promptly. When we
drop a still-pending future like this, we call that "cancellation".

There's nothing particularly special about cancelling a future compared to
dropping any other Rust object. Its `Drop::drop` function runs (if any), and
then the `Drop::drop` functions of its fields run (if any), all as usual.
Importantly, this includes local variables in `async` blocks and functions,
which are fields in their compiler-generated futures. It's good that we don't
leak those, of course. But the important thing to understand about cancellation
is less _how_ we do it, and more that we're forced to _do something_ rather
than nothing. The `Poll::Pending` rule requires every future to actively
participate in what we might call the "`Waker` protocol" between its caller and
any child futures it might contain. When a future is polled, it can poll its
own children in turn (unless it knows for a fact that they did not request a
wakeup), or it can cancel them by dropping them (either directly or indirectly,
e.g. by returning `Ready` and trusting the caller to drop it), but it can't
silently ignore a child's wakeup.

This turns out to be essential for futures that can acquire locks or other
exclusive resources. If a future is supposed to hold a lock for a short time,
the programmer needs to consider that it might release the lock sooner if it's
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

### A _lot_ of existing code breaks the "`Poll::Pending` rule"

There are many patterns that currently violate the "`Poll::Pending` rule", and
we can come up with a version of the the deadlock in the "Motivation" section
for each of them. This RFC doesn't attempt to decide how each case should be
fixed, but see the "Future possibilities" section for possible approaches.

#### `select!` by reference

The `timeout` example in the "Motivation" is a little bit easier to understand
than this [`select!`] version, but the `select!` version is more common
([playground link][select_deadlock]):

[select_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+foo_future+%3D+pin%21%28foo%28%29%29%3B%0A++++loop+%7B%0A++++++++select%21+%7B%0A++++++++++++_+%3D+%26mut+foo_future+%3D%3E+%7B%7D%2C%0A++++++++++++_+%3D+sleep%28Duration%3A%3Afrom_millis%281%29%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++++++foo%28%29.await%3B%0A++++++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++++++%7D%2C%0A++++++++%7D%0A++++%7D%0A%7D>

```rust
let mut foo_future = pin!(foo());
loop {
    select! {
        _ = &mut foo_future => {},
        _ = sleep(Duration::from_millis(1)) => foo().await, // Deadlock!
    }
}
```

#### cancellation-by-reference in general

([playground link][poll_immediate_deadlock])

[poll_immediate_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+foo_future+%3D+pin%21%28foo%28%29%29%3B%0A++++futures%3A%3Afuture%3A%3Apoll_immediate%28foo_future%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
let foo_future = pin!(foo());
futures::future::poll_immediate(foo_future).await;
foo().await; // Deadlock!
```

([playground link][take_until_deadlock])

[take_until_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3A%7Bself%2C+StreamExt+as+_%7D%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+my_stream+%3D+pin%21%28stream%3A%3Aonce%28foo%28%29%29%29%3B%0A++++my_stream%0A++++++++.take_until%28sleep%28Duration%3A%3Afrom_millis%281%29%29%29%0A++++++++.for_each%28async+%7C_%7C+%7B%7D%29%0A++++++++.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
let my_stream = pin!(stream::once(foo()));
my_stream
    .take_until(sleep(Duration::from_millis(1)))
    .for_each(async |_| {})
    .await;
foo().await; // Deadlock!
```

#### `futures::future::select`

([playground link][select_fn_deadlock])

[select_fn_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afuture%3A%3Aready%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+_ret+%3D+futures%3A%3Afuture%3A%3Aselect%28Box%3A%3Apin%28foo%28%29%29%2C+ready%28%28%29%29%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
let _ret = futures::future::select(Box::pin(foo()), ready(())).await;
foo().await; // Deadlock!
```

#### `StreamExt::next`

([playground link][next_deadlock])

[next_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3A%7Bself%2C+StreamExt%7D%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+stream+%3D+pin%21%28stream%3A%3Aonce%28foo%28%29%29%29%3B%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+stream.next%28%29%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
let mut stream = pin!(stream::once(foo()));
_ = timeout(Duration::from_millis(1), stream.next()).await;
foo().await; // Deadlock!
```

#### concurrent streams

([playground link][merge_deadlock])

[merge_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3A%7Bself%2C+StreamExt+as+_%7D%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0Ause+tokio_stream%3A%3AStreamExt+as+_%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++stream%3A%3Aonce%28foo%28%29%29%0A++++++++.merge%28stream%3A%3Aonce%28foo%28%29%29%29%0A++++++++.for_each%28%7C_%7C+async+%7B%0A++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++foo%28%29.await%3B%0A++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++%7D%29%0A++++++++.await%3B%0A%7D>

```rust
stream::once(foo())
    .merge(stream::once(foo()))
    .for_each(|_| async {
        foo().await; // Deadlock!
    })
    .await;
```

#### `FutureExt::shared`

([playground link][shared_deadlock])

[shared_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Afuture%3A%3AFutureExt%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+foo_future+%3D+foo%28%29.shared%28%29%3B%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+foo_future.clone%28%29%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
let foo_future = foo().shared();
_ = timeout(Duration::from_millis(1), foo_future.clone()).await;
foo().await; // Deadlock!
```

#### `LocalSet`

([playground link][localset_deadlock])

[localset_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+local+%3D+tokio%3A%3Atask%3A%3ALocalSet%3A%3Anew%28%29%3B%0A++++local.spawn_local%28foo%28%29%29%3B%0A++++local.run_until%28sleep%28Duration%3A%3Afrom_millis%281%29%29%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++foo%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
let local = tokio::task::LocalSet::new();
local.spawn_local(foo());
local.run_until(sleep(Duration::from_millis(1))).await;
foo().await; // Deadlock!
```

### Pausing things is useful, and it would've been nice to allow it.

Some applications might want to pause low-priority work when load is high.
Others might have an actual pause button (e.g. games, media) that they'd like
to implement by pausing futures. Those might not be our recommended
architectural choices, but effectively forbidding them at the language level
seems quite opinionated.[^forbid]

[^forbid]: Of course an application can ultimately do whatever it likes,
    including pausing futures or killing threads. Part of what's at stake here
    is the question of who's "at fault" when such an application collides with
    a library ecosystem that uses locks.

Similarly, Windows has [`SuspendThread`] and [`TerminateThread`], and Unix has
[`pthread_cancel`], because many applications over the years have wanted to
non-cooperatively pause or cancel running threads.[^raymond_chen1] That's
understandable; passing a cancel flag around everywhere is inconvenient at
best, and it can be impossible when we're working with other people's code.
However, today we understand that these functions are _radioactive_. Outside of
a short list of low-level use cases, they tend to corrupt the entire
process.[^raymond_chen2] We generally ban them.

[^raymond_chen1]: "Originally, there was no `TerminateThread` function. The
    original designers felt strongly that no such function should exist because
    there was no safe way to terminate a thread, and there's no point having a
    function that cannot be called safely. But people screamed that they needed
    the `TerminateThread` function, even though it wasn't safe, so the
    operating system designers caved and added the function because people
    demanded it. Of course, those people who insisted that they needed
    `TerminateThread` now regret having been given it." - [Raymond
    Chen][raymond_chen1]

[raymond_chen1]: https://devblogs.microsoft.com/oldnewthing/20150814-00/?p=91811

[^raymond_chen2]: "These results are not specific to C#. The same logic applies
    to Win32 or any other threading model. In Win32, the process heap is a
    threadsafe object, and since it's hard to do very much in Win32 at all
    without accessing the heap, suspending a thread in Win32 has a very high
    chance of deadlocking your process." - [Raymond Chen][raymond_chen2]

[raymond_chen2]: https://devblogs.microsoft.com/oldnewthing/20031209-00/?p=41573

Unfortunately, pausing futures in async Rust has all the same problems. Taking
an async lock is far less common than e.g. calling `malloc`, so the symptoms
aren't as noticeable today, but they'll get worse as the ecosystem grows and
private locks appear in more places. In [the original "Futurelock"
incident][incident], the culprit was a semaphore buried in the
`tokio::sync::mpsc` channel implementation. The channel in question wasn't even
visible at the point where pausing happened. The non-local nature of these bugs
forces us to take a position on pausing at the language/ecosystem level.

[incident]: https://github.com/oxidecomputer/omicron/issues/9259

Given all that, it's remarkable that non-cooperative[^noncoop] cancellation
works as well as it does in async Rust. It [has its
issues][cancelling_async_rust], but it doesn't generally cause deadlocks, and
many applications use selects and timeouts routinely in production. That's
quite an achievement, and perhaps an unexpected benefit of destructor-based
cleanup and by extension the borrow checker.

[^noncoop]: "Cooperative" vs "non-cooperative" has a couple different
    interpretations here. From the perspective of an executor thread that's
    calling `Future::poll`, everything is cooperative, because we can't force
    that function to ever return. On the other hand, a `poll` function that
    doesn't return promptly is gumming up the executor, and we have [tools for
    finding those][slow_poll]. If we take it for granted that every long `poll`
    bug eventually gets fixed, then we could think of cancellation in async
    Rust as _non_-cooperative. There's nothing a correct `async fn` can do to
    prevent it or delay it for very long.

[slow_poll]: https://docs.rs/tokio-metrics/latest/tokio_metrics/struct.TaskMonitor.html#method.with_slow_poll_threshold

[cancelling_async_rust]: https://sunshowers.io/posts/cancelling-async-rust/

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Can we enforce the `Future` contract programmatically?

TODO: Yes.

### What's the point of the drop requirement after `Poll::Ready`?

The focus of this RFC is the "`Poll::Pending` rule" about polling again
promptly after a wakeup, but it also establishes a "`Poll::Ready` rule" about
dropping a future promptly after it's finished. The benefit of this rule is
that futures like [`Timeout`][`timeout`] and [`Race`] can contain their child
futures directly (like they do today), without needing `Option`, [`MaybeDone`],
or similar to represent the state where they drop a child without being dropped
themselves. Instead, they cancel their children by returning `Ready` and
trusting that their caller will drop them promptly. In other words, `Timeout`
and `Race` can rely on the `Poll::Ready` rule to guarantee that they follow the
`Poll::Pending` rule.

[`Race`]: https://docs.rs/futures-lite/latest/futures_lite/future/fn.race.html

Combinators like [`Join`] do need extra state to meet this requirement. When
one side of a `Join` finishes, it needs to drop that future immediately,
without waiting for both sides to finish. Luckily, most implementations of
`Join` already do this today, using `MaybeDone` or similar, because it saves
space. (`MaybeDone` holds either a future or its output, but not both at the
same time.) Codifying the `Poll::Ready` rule isn't expected to require many
code changes,[^code_changes] but it clarifies that callees can rely on the rule
for correctness.

[`Join`]: https://docs.rs/futures/latest/futures/future/fn.join.html

[^code_changes]: There are no known violations of the `Poll::Ready` rule in the
    current versions of `futures-rs`, Tokio, or `futures-lite`. The original
    implementation of the [`futures_lite::future::Zip`] combinator did break
    the rule, but that was [reported as a deadlock bug][zip_deadlock] and fixed
    in 2024.

[`futures_lite::future::Zip`]: https://docs.rs/futures-lite/latest/futures_lite/future/fn.zip.html
[zip_deadlock]: https://github.com/smol-rs/futures-lite/issues/105

## Prior art
[prior-art]: #prior-art

TODO

## Unresolved questions
[unresolved-questions]: #unresolved-questions

### Should we allow an indefinite delay between creation and polling?

In other words, should the following be allowed, or should it e.g. fail Clippy?

```rust
let future1 = foo();
let future2 = foo();
future1.await;
future2.await;
```

We could say that `future2` is paused here across the first await. On the other
hand, `future2` has never been polled (or even pinned), and it's not likely to
be holding any exclusive resources in its initial state. We could imagine
giving `foo` e.g. a `MutexGuard` argument, but in that case the caller could
clearly see what's going on. To create a true "nonlocal reasoning" problem,
we'd need to write `foo` in a sync-then-async style, like this:

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

This style is uncommon, and it's especially uncommon to combine it with eagerly
acquired locks, but [here's an example of Wasmer doing it][wasmer]. At the same
time, there are published `Future` extension methods (e.g. [`delay`] from
`async-std`) that are only correct if delayed initial polling is acceptable.

[`delay`]: https://docs.rs/async-std/latest/async_std/prelude/trait.FutureExt.html#method.delay
[wasmer]: https://github.com/wasmerio/wasmer/blob/dce84a907542c331661f201eff9d898ecb2fbe08/lib/virtual-net/src/rx_tx.rs#L171-L187

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

TODO: linting on the blanket impl, new macros

TODO: `AsyncIterator`

[barbara]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html
["Futurelock"]: https://rfd.shared.oxide.computer/rfd/0609
[`pthread_cancel`]: https://man7.org/linux/man-pages/man3/pthread_cancel.3.html
[`TerminateThread`]: https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread
[`SuspendThread`]: https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-suspendthread
[snooze]: https://jacko.io/snooze.html
[`select!`]: https://docs.rs/tokio/latest/tokio/macro.select.html
[`AsyncIterator`]: https://doc.rust-lang.org/core/async_iter/trait.AsyncIterator.html
