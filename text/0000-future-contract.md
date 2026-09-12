- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Clarify and document the contract requirements of the [`Future::poll`] method,
in particular that a future should be polled or dropped promptly when it
requests a wakeup. In other words, we can cancel a future at any time, but we
should never "pause" or ["snooze"][snooze] a future.

[`Future::poll`]: https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll

There are widely used functions and patterns that violate this rule, including
[`select!`]-by-reference and [`StreamExt::next`]. This RFC identifies several,
but it avoids endorsing any specific changes beyond the `Future` docs. The goal
is to agree that existing contract violations are bugs, and that we can and
should fix them, but deciding how exactly to fix or replace each problematic
case is left to follow-up RFCs and the crates ecosystem.

[`StreamExt::next`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.next

## Motivation
[motivation]: #motivation

Cancellation and pausing are distinct features of async Rust. Regular,
synchronous Rust doesn't let us kill or suspend threads, because doing either
of those things tends to cause deadlocks.[^deprecated] Async cancellation
solves the deadlock problem(!) by dropping cancelled futures, which
automatically releases any locks they might be holding. But async pausing does
not solve the deadlock problem, which makes it more of a bug than a feature.
For a case study in how difficult and non-local these deadlocks can be, see
["Futurelock"] (Oxide, October 2025).

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
do it accidentally. Futurelock was caused by a snoozing bug [in a `select!`
loop][futurelock_select], and async streams have been [battling hangs and
deadlocks][barbara] for years. These mistakes are invisible unless you know
exactly what you're looking for.

[^dioxus]: The only widely-used counterexample might be the Dioxus framework,
    which [provides a `pause` method][dioxus_docs] and sometimes [calls it
    automatically][dioxus_src].

[dioxus_docs]: https://docs.rs/dioxus/0.7.10/dioxus/prelude/struct.UseFuture.html#method.pause
[dioxus_src]: https://github.com/DioxusLabs/dioxus/blob/v0.7.10/packages/hooks/src/use_future.rs#L63-L72

[futurelock_select]: https://github.com/oxidecomputer/omicron/pull/9268/changes?diff=split

Here's an example deadlock using [`timeout`].[^usual_suspect] Imagine that
`foo`, `bar`, `baz`, and `main` are all defined in different crates, and that
the author of `main` has never heard of `foo` ([playground
link][timeout_deadlock]):

[timeout_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%2F%2F+A+couple+trivial+wrapper+functions%2C+to+make+the+deadlock+below+less+%22obvious%22.%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+mut+bar_future+%3D+pin%21%28bar%28%29%29%3B%0A++++while+timeout%28Duration%3A%3Afrom_millis%285%29%2C+%26mut+bar_future%29.await.is_err%28%29+%7B%0A++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++baz%28%29.await%3B%0A++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++%7D%0A%7D>

[^usual_suspect]: `select!` is the usual suspect in these issues, but using
    `timeout` instead makes it easier to discuss its function signature. As
    we'll see below, any form of cancellation has the same problem when
    combined with [the blanket `Future` impl on `Pin<&mut _>`
    references][blanket].

[blanket]: https://doc.rust-lang.org/std/future/trait.Future.html#impl-Future-for-Pin%3CP%3E

```rust
async fn foo() {
    // Acquire a global lock, sleep briefly, and release it.
    static LOCK: Mutex<()> = Mutex::const_new(());
    let _guard = LOCK.lock().await;
    sleep(Duration::from_millis(10)).await;
}

// A couple trivial wrapper functions, to make the deadlock below less "obvious".
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
    while timeout(Duration::from_millis(5), &mut bar_future).await.is_err() {
        baz().await; // Deadlock!
    }
}
```

While control is waiting on `baz()`, nothing is polling `bar_future`. But
`bar_future` is already holding the lock that `baz` wants to acquire, and the
result is a deadlock. Let's look closely at who gets polled when. If we [add
some prints][squawk], we can see that `main` gets polled three times:[^slack]

[squawk]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+Instant%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%2F%2F+A+couple+trivial+wrapper+functions%2C+to+make+the+deadlock+below+less+%22obvious%22.%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+main_inner%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+mut+bar_future+%3D+pin%21%28bar%28%29%29%3B%0A++++while+timeout%28Duration%3A%3Afrom_millis%285%29%2C+%26mut+bar_future%29.await.is_err%28%29+%7B%0A++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++baz%28%29.await%3B%0A++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++%7D%0A%7D%0A%0A%2F%2F+Squawk+a+timestamp+every+time+%60future%60+gets+polled.%0Afn+squawk%3CFut%3A+Future%3E%28future%3A+Fut%29+-%3E+impl+Future%3COutput+%3D+Fut%3A%3AOutput%3E+%7B%0A++++let+start+%3D+Instant%3A%3Anow%28%29%3B%0A++++let+mut+future+%3D+Box%3A%3Apin%28future%29%3B%0A++++std%3A%3Afuture%3A%3Apoll_fn%28move+%7Ccx%7C+%7B%0A++++++++let+elapsed+%3D+Instant%3A%3Aelapsed%28%26start%29.as_secs_f32%28%29+*+1000.0%3B%0A++++++++println%21%28%22%5B%7Belapsed%3A.3%7D+ms%5D+POLLED%21%22%29%3B%0A++++++++future.as_mut%28%29.poll%28cx%29%0A++++%7D%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++squawk%28main_inner%28%29%29.await%3B%0A%7D>

[^slack]: Tokio's timer implementation adds ~1 ms of slack to our 5 ms timeout
    and our 10 ms sleep.

- 0 ms: Control first enters `main` and `bar`.
- 5-6 ms: The first (and only) `timeout` expires and invokes its `Waker`,
  `main` gets polled again, and control enters `baz`.
- 10-11 ms: The `sleep` in `bar` completes and invokes its `Waker`,[^sleep]
  `main` gets polled a third time, and it polls `baz` again.

[^sleep]: This can be confusing: How does `sleep` invoke anything if the
    `Sleep` future isn't getting polled? It's true that control never reaches
    the `Sleep` future again after the first poll, but low-level IO is driven
    by external events, and we arrange for those events to invoke wakers. For
    sleeps and other timers, Tokio runs a ["hashed timer wheel"][tokio_timer]
    in the background, and `sleep` registers its `Waker` there.

[tokio_timer]: https://tokio.rs/blog/2018-03-timers

We re-poll `baz` at 10 ms, even though it didn't request a wakeup. That's not
in and of itself a problem; futures are expected to tolerate extra polling. But
`bar` did request a wakeup, and `main` doesn't poll it. That's a problem.[^hang]

[^hang]: Even if there was no lock in this example, and thus no deadlock, it's
    still surprising that `bar` is paused every time we call `baz`. Most users
    in this situation expect them to run concurrently, except in [rare cases
    involving complicated mutation][mini_redis]. Snoozing futures in a real
    application can cause performance issues, network timeout errors, or UI
    stuttering. This RFC focuses on deadlocks for clarity, but these other bugs
    are also [common in practice][barbara].

Consider the problem from the perspective of the programmer writing `foo`.
You're acquiring `LOCK`, and it's your responsibility not to hold it for too
long. If you do any blocking IO while you hold it, you're trusting the runtime
to trigger your wakeup correctly. That's fine; you naturally rely on the
runtime for correctness, just like you rely on the standard library and the
compiler. But then, you're also trusting your callers to deliver that wakeup.
Is that fine? What if you're writing library code, and you don't know anything
about your callers?[^spawn_task]

[^spawn_task]: Futures spawned as tasks get their wakeups directly from the
    runtime, so spawning a task is one way to guarantee our wakeups will arrive
    without trusting unknown callers. But spawning isn't a general-purpose
    solution for these issues. It requires heap allocation, it isn't supported
    in all environments, and it isn't compatible with local borrowing.

For async locks to be usable -- or any type that contains one, like a
[`OnceCell`] or a [bounded `mpsc` channel][mpsc] -- we need a guarantee that
callers will either deliver our wakeups or drop us promptly. The whole
ecosystem needs to agree on this "strict" `Future` contract. In the example
above, `main` is at fault for the deadlock, and the `Future` docs need to make
that clear.

[`OnceCell`]: https://docs.rs/tokio/latest/tokio/sync/struct.OnceCell.html
[mpsc]: https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html

Unfortunately, `main` doesn't _look_ broken. If we want the `Future` contract
to have strict rules, we also need warnings and errors that let us know when we
break those rules. Can we deprecate some problematic type or function that
`main` is using? The natural suspect here is `timeout`.[^pin] Let's look at its
[function signature][`timeout`]:

[at its signature]: https://docs.rs/tokio/1.53.1/src/tokio/time/timeout.rs.html#86-98

[^pin]: `pin!` is arguably a red flag, but pinning per se doesn't have anything
    to do with control flow or wakeups. We often manage heterogenous futures as
    `Pin<Box<dyn Future>>`, but we can also [abuse one of those][boxed] to
    replace `pin!` in this example.

[boxed]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3AFutureExt%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%2F%2F+A+couple+trivial+wrapper+functions%2C+to+make+the+deadlock+below+less+%22obvious%22.%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+mut+bar_future+%3D+bar%28%29.boxed%28%29%3B%0A++++while+timeout%28Duration%3A%3Afrom_millis%285%29%2C+%26mut+bar_future%29.await.is_err%28%29+%7B%0A++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++baz%28%29.await%3B%0A++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++%7D%0A%7D>

```rust
pub fn timeout<F: IntoFuture>(duration: Duration, future: F) -> Timeout<F::IntoFuture>
```

That signature shows us something important: `timeout` takes `future` _by
value_. It winds up in some field of the [`Timeout`] struct, so when that
struct drops, `future` drops too. In other words, if `duration` expires before
`future` is finished, `timeout` cancels `future` by dropping it. That's exactly
what the strict contract says it should do.

[`Timeout`]: https://docs.rs/tokio/1.53.1/tokio/time/struct.Timeout.html

But then, why didn't that prevent our deadlock above? We didn't actually pass
the `bar` future to `timeout` by value.[^compiler_error] Instead, we used a
`&mut Pin<&mut _>` reference. The question is, why did that compile? There are
several blanket impls involved, but the most important one is [the `Future`
impl for `Pin<&mut _>` references][blanket]. In effect, a `Pin<&mut _>`
reference to a `Future` is itself a `Future`, except that dropping it has no
effect. If the strict `Future` contract requires us to drop cancelled futures
promptly, then that blanket impl is broken.

[^compiler_error]: Also, if we passed `bar` to `timeout` by value, our loop
    wouldn't compile. The compiler would force us to create a new `bar` future
    in each loop iteration. That's deadlock-free, but it's not the behavior we
    want here. See [the rationales
    section](#what-does-a-corrected-version-of-the-broken-main-function-above-look-like)
    for examples of implementing the behavior we want correctly.

However, this RFC doesn't propose deprecating it today. For one thing, Rust
doesn't currently have a way to deprecate a trait impl. Also, the same impl
covers `Pin<Box<_>>`, which absolutely should implement `Future`. But most
importantly, tons of existing async code uses `Pin<&mut _>` references as
futures today, and it will take months to years to roll out helper functions
and macros that handle the same use cases with ownership instead. Also, while
this is the most common way to violate the strict `Future` contract today, it's
not the only way. `AsyncIterator` and `Stream` have similar deadlock bugs, and
we'll need at least one follow-up RFC to address those. See the drawbacks
section below for a list of related problems.

[^box]: That impl covers all `Pin<P> where P: DerefMut<Target: Future>`, which
    includes both `Pin<&mut _>` and `Pin<Box<_>>`. The former is broken, but
    the latter is fine, and it's critical for working with trait objects. Even
    ignoring backwards compatibility concerns, we don't want to deprecate the
    whole impl.

Instead, this RFC proposes the smallest possible change: Document the strict
`Future` contract. Once we agree on exactly where these bugs are coming from,
we can start the gradual process of fixing them.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `Future::poll` docs

The `poll` method imposes two responsibilities on its caller:

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

#### Example

Here's an example of a `Future` implementation that fails the first requirement
above, a.k.a. the "`Poll::Pending` rule":

```rust
pub struct CoinFlip<Fut>(Pin<Box<Fut>>);

impl<Fut: Future> Future for CoinFlip<Fut> {
    type Output = Fut::Output;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<Fut::Output> {
        if rand::random() {
            self.0.as_mut().poll(cx)
        } else {
            // XXX: `self.0` might have requested a wakeup. Returning without polling it here
            // violates the `Future` contract.
            Poll::Pending
        }
    }
}
```

The problem is that `random()` might be true the first time, polling the inner
`Fut` and letting it register wakeups, but then it might be false the second
time when those wakeups trigger, failing to poll `Fut` promptly.[^first_time]
This tends to cause hangs and deadlocks, not only for the future that didn't
get polled, but also in distant and unrelated futures that happen to use the
same shared resources ([playground link][coin_flip]). `CoinFlip` would be at
fault for those bugs. There are three different ways we could fix it:

[coin_flip]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3APin%3B%0Ause+std%3A%3Atask%3A%3A%7BContext%2C+Poll%7D%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%2F%2F+A+couple+trivial+wrapper+functions%2C+to+make+the+deadlock+below+less+%22obvious%22.%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Apub+struct+CoinFlip%3CFut%3E%28Pin%3CBox%3CFut%3E%3E%29%3B%0A%0Aimpl%3CFut%3A+Future%3E+Future+for+CoinFlip%3CFut%3E+%7B%0A++++type+Output+%3D+Fut%3A%3AOutput%3B%0A%0A++++fn+poll%28mut+self%3A+Pin%3C%26mut+Self%3E%2C+cx%3A+%26mut+Context%29+-%3E+Poll%3CFut%3A%3AOutput%3E+%7B%0A++++++++if+rand%3A%3Arandom%28%29+%7B%0A++++++++++++self.0.as_mut%28%29.poll%28cx%29%0A++++++++%7D+else+%7B%0A++++++++++++%2F%2F+XXX%3A+%60self.0%60+might+have+requested+a+wakeup.+Returning+without+polling+it+here%0A++++++++++++%2F%2F+violates+the+%60Future%60+contract.%0A++++++++++++Poll%3A%3APending%0A++++++++%7D%0A++++%7D%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+iteration+%3D+0%3B%0A++++loop+%7B%0A++++++++iteration+%2B%3D+1%3B%0A++++++++dbg%21%28iteration%29%3B%0A++++++++%2F%2F+This+deadlocks+25%25+of+the+time%2C+so+we+run+it+in+a+loop.+We+need+three+things+to+happen%3A%0A++++++++%2F%2F+++1.+%60select%21%60+polls+%60coin_flip%60+first.+The+%60biased%60+keyword+guarantees+this.%0A++++++++%2F%2F+++2.+The+first+poll+of+%60coin_flip%60+flips+%60true%60%2C+so+%60bar%60+acquires+%60LOCK%60.%0A++++++++%2F%2F+++3.+The+second+poll+of+%60coin_flip%60+flips+%60false%60%2C+so+%60bar%60+never+releases+%60LOCK%60.%0A++++++++let+coin_flip+%3D+CoinFlip%28Box%3A%3Apin%28bar%28%29%29%29%3B%0A++++++++select%21+%7B%0A++++++++++++biased%3B%0A++++++++++++_+%3D+coin_flip+%3D%3E+%7B%7D%0A++++++++++++_+%3D+baz%28%29+%3D%3E+%7B%7D+%2F%2F+Maybe+deadlock%21%0A++++++++%7D%0A++++%7D%0A%7D>

- Return `Ready` in the `else` branch, which requires the caller to drop
  `CoinFlip` promptly. We'd probably need to change the `Output` type to
  `Option` or `Result`.
- Drop `self.0` in the `else` branch before returning `Pending`. In this case
  `self.0` would need to be `Option<Fut>` or similar.
- Panic in the `else` branch. This probably isn't what anyone wants, but it's
  technically correct.[^futurama]

[^first_time]: On the other hand, if `random` is false the first time, we might
    never poll `Fut`. Whether that's acceptable according to the `Future`
    contract is an open question. See [the unresolved questions
    section](#should-we-allow-an-indefinite-delay-between-creation-and-polling).

[^futurama]: [The best kind of correct.][futurama]

[futurama]: https://www.youtube.com/watch?v=aIzMuPMicGc&t=21s

#### Cancellation

Unlike threads, which have a life of their own once they start running, a
future only makes progress when something polls it. We can effectively pause
the execution of a future by not polling it again. However, the
"`Poll::Pending` rule" above limits our options here. If a wakeup arrives, but
we don't want to poll the future that triggered it[^unknown] -- for example
because a deadline has passed, or because we no longer need its output -- we
must drop that future promptly. When we drop a still-pending future like this,
we call that "cancellation".

[^unknown]: It's possible to know which child (or children) triggered a given
    wakeup by giving each child a unique `Waker`. [`FuturesUnordered`] does
    this, for example. But most combinators forward their own `Waker` directly
    to their children, so they don't know which child triggered a wakeup, and
    they need to poll all their children every time. Futures tolerate extra
    polling, so both approaches are valid.

[`FuturesUnordered`]: https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html

This rule is essential for futures that acquire locks or other exclusive
resources. When an async function holds a lock across an await point, the
programmer needs to consider that it might release that lock sooner than
expected if it's cancelled, or a bit later due to timer slack or CPU load. But
the programmer doesn't need to worry about the caller pausing execution and
thereby (accidentally, unknowingly) holding the lock _forever_.

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

What these cases have in common, though, is that barring program exit or the
power going out, future progress is guaranteed. There's no legitimate way for
user code to stop the runtime from working through its task list or freeze a
private worker thread.[^illegitimate] The deadlocks above are different: a
future is suspended across some arbitrary bit of user code that isn't
guaranteed to ever finish.

[^illegitimate]: _Illegitimate_ ways to interfere with the runtime include
    synchronous blocking in `poll`, calling unsafe functions like
    [`pthread_cancel`], or just corrupting memory.

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

#### unwinding from `block_on`

([playground link][unwinding_deadlock])

[unwinding_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apanic%3A%3Acatch_unwind%3B%0Ause+std%3A%3Atime%3A%3ADuration%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0A%0Astatic+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A%0Aasync+fn+async_foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++tokio%3A%3Atime%3A%3Asleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Afn+sync_foo%28%29+%7B%0A++++%2F%2F+As+above%2C+but+sync+rather+than+async.%0A++++let+_guard+%3D+LOCK.blocking_lock%28%29%3B%0A++++std%3A%3Athread%3A%3Asleep%28Duration%3A%3Afrom_millis%2810%29%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++%2F%2F+Build+a+single-threaded+runtime.+If+we+used+%60new_multi_thread%60+instead%2C+then+%60async_foo%60%0A++++%2F%2F+would+start+running+on+a+worker+thread+as+soon+as+we+spawned+it%2C+and+it+would+keep+running%0A++++%2F%2F+even+after+the+panic+below.+We+wouldn%27t+get+a+deadlock+in+that+case.+See%3A%0A++++%2F%2F+https%3A%2F%2Fdocs.rs%2Ftokio%2F1.53.1%2Ftokio%2Fruntime%2F%23driving-the-runtime%0A++++let+runtime+%3D+tokio%3A%3Aruntime%3A%3ABuilder%3A%3Anew_current_thread%28%29%0A++++++++.enable_time%28%29%0A++++++++.build%28%29%0A++++++++.unwrap%28%29%3B%0A%0A++++%2F%2F+Run+%60async_foo%60+in+the+background.+Execution+doesn%27t+actually+begin+until+%60block_on%60+below.%0A++++runtime.spawn%28async_foo%28%29%29%3B%0A%0A++++%2F%2F+Start+driving+the+runtime+with+%60block_on%60+and+a+second+future.+This+async+block+panics+after%0A++++%2F%2F+5+ms%2C+which+unwinds+out+of+%60block_on%60%2C+but+we+catch+the+panic+here+in+%60main%60.%0A++++_+%3D+catch_unwind%28%7C%7C+%7B%0A++++++++runtime.block_on%28async+%7B%0A++++++++++++tokio%3A%3Atime%3A%3Asleep%28Duration%3A%3Afrom_millis%285%29%29.await%3B%0A++++++++++++panic%21%28%22panic+while+%60async_foo%60+holds+%60LOCK%60%22%29%3B%0A++++++++%7D%29%3B%0A++++%7D%29%3B%0A%0A++++%2F%2F+At+this+point+the+%60async_foo%60+future+is+still+holding+%60LOCK%60%2C+but+we%27re+no+longer+driving%0A++++%2F%2F+the+runtime+that+owns+it.+If+we+try+to+take+%60LOCK%60+any+other+way+before+we+either+resume%0A++++%2F%2F+driving+%60runtime%60+or+drop+it%2C+we+get+a+deadlock.%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++sync_foo%28%29%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rust
let runtime = tokio::runtime::Builder::new_current_thread()
    .enable_time()
    .build()
    .unwrap();
runtime.spawn(async_foo());
_ = catch_unwind(|| {
    runtime.block_on(async {
        tokio::time::sleep(Duration::from_millis(5)).await;
        panic!("panic while `async_foo` holds `LOCK`");
    });
});
sync_foo(); // Deadlock!
```

### Pausing things is useful, and it would've been nice to allow it.

Some applications might want to pause low-priority work when load is high.
Others might have an actual pause button (e.g. games, media) that they'd like
to implement by pausing futures. Those might not be our recommended
architectural choices, but effectively forbidding them at the language level
seems quite opinionated.[^forbid]

[^forbid]: Of course applications can ultimately do whatever they like,
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

### What does a corrected version of the broken `main` function above look like?

As a reminder, the broken `main` function from the motivation section looked
like this ([playground link][timeout_deadlock]):

```rust
#[tokio::main]
async fn main() {
    // While `bar` is running, call `baz` every 5 ms.
    let mut bar_future = pin!(bar());
    while timeout(Duration::from_millis(5), &mut bar_future).await.is_err() {
        baz().await; // Deadlock!
    }
}
```

We'd like to factor out the `baz` loop into its own `async` block and run it
concurrently, but we can't [`join`] that block with `baz`, because it never
returns. There are a few different ways we could approach this, but if we
wanted to stick with existing, widely-used helpers, one option would be to use
`select!` ([playgroud link][select_baz_loop]):[^cancellation_token]

[`join`]: https://docs.rs/futures/latest/futures/future/fn.join.html

[^cancellation_token]: Another option is to wrap the `baz` loop with a
    [`CancellationToken`], though that's easier to get wrong, and it also uses
    heap allocation internally.

[`CancellationToken`]: https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html

[select_baz_loop]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+baz_loop+%3D+async+%7B%0A++++++++loop+%7B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%285%29%29.await%3B%0A++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++baz%28%29.await%3B%0A++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++%7D%0A++++%7D%3B%0A++++select%21+%7B%0A++++++++_+%3D+bar%28%29+%3D%3E+%7B%7D%2C%0A++++++++_+%3D+baz_loop+%3D%3E+%7B%7D%2C%0A++++%7D%0A++++println%21%28%22...and+then+we+exit.%22%29%3B%0A%7D>

```rust
#[tokio::main]
async fn main() {
    // While `bar` is running, call `baz` every 5 ms.
    let baz_loop = async {
        loop {
            sleep(Duration::from_millis(5)).await;
            baz().await;
        }
    };
    select! {
        _ = bar() => {},
        _ = baz_loop => {},
    }
}
```

This works, and it's nice that it doesn't require `pin!`. But a downside of
this approach is that `select!` makes it look like we're waiting for either
`bar` or the `baz_loop` to finish. We know that the `baz_loop` will never
finish, so the resulting behavior is correct, but `select!` doesn't really
capture our intent. It would also be awkward if we needed the return value of
`bar`.

We could imagine a small helper function that might fit better, though it's not
provided in `futures-rs` or Tokio today. It's job would be to drive two futures
concurrently, but to only wait on the first one to finish. Let's call it
`join_maybe`:

```rust
/// Run a "definitely" future and a "maybe" future concurrently. If the definitely future finishes
/// first, cancel the maybe future. Return the definitely output together with the maybe output if
/// it wasn't cancelled.
async fn join_maybe<Fut1: Future, Fut2: Future>(
    definitely: Fut1,
    maybe: Fut2,
) -> (Fut1::Output, Option<Fut2::Output>) { ... }
```

Here's what our `main` function looks like using `join_maybe` instead of `select!` ([playground link][join_maybe]):

[join_maybe]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Afuture%3A%3AMaybeDone%3B%0Ause+std%3A%3Apin%3A%3APin%3B%0Ause+std%3A%3Atask%3A%3A%7BContext%2C+Poll%7D%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++%2F%2F+Acquire+a+global+lock%2C+sleep+briefly%2C+and+release+it.%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+bar%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Aasync+fn+baz%28%29+%7B%0A++++foo%28%29.await%3B%0A%7D%0A%0Afn+join_maybe%3CFut1%3A+Future%2C+Fut2%3A+Future%3E%28definitely%3A+Fut1%2C+maybe%3A+Fut2%29+-%3E+JoinMaybe%3CFut1%2C+Fut2%3E+%7B%0A++++JoinMaybe+%7B%0A++++++++definitely%3A+Box%3A%3Apin%28definitely%29%2C%0A++++++++maybe%3A+Box%3A%3Apin%28MaybeDone%3A%3AFuture%28maybe%29%29%2C%0A++++%7D%0A%7D%0A%0Astruct+JoinMaybe%3CFut1%3A+Future%2C+Fut2%3A+Future%3E+%7B%0A++++%2F%2F+%60pin_project_lite%60+isn%27t+available+on+the+Playground%2C+so+just+use+%60Pin%3CBox%3C_%3E%3E%60.%0A++++definitely%3A+Pin%3CBox%3CFut1%3E%3E%2C%0A++++maybe%3A+Pin%3CBox%3CMaybeDone%3CFut2%3E%3E%3E%2C%0A%7D%0A%0Aimpl%3CFut1%3A+Future%2C+Fut2%3A+Future%3E+Future+for+JoinMaybe%3CFut1%2C+Fut2%3E+%7B%0A++++type+Output+%3D+%28Fut1%3A%3AOutput%2C+Option%3CFut2%3A%3AOutput%3E%29%3B%0A%0A++++fn+poll%28mut+self%3A+Pin%3C%26mut+Self%3E%2C+cx%3A+%26mut+Context%29+-%3E+Poll%3CSelf%3A%3AOutput%3E+%7B%0A++++++++let+definitely_poll+%3D+self.definitely.as_mut%28%29.poll%28cx%29%3B%0A++++++++_+%3D+self.maybe.as_mut%28%29.poll%28cx%29%3B%0A++++++++if+let+Poll%3A%3AReady%28definitely_output%29+%3D+definitely_poll+%7B%0A++++++++++++Poll%3A%3AReady%28%28definitely_output%2C+self.maybe.as_mut%28%29.take_output%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Poll%3A%3APending%0A++++++++%7D%0A++++%7D%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++%2F%2F+While+%60bar%60+is+running%2C+call+%60baz%60+every+5+ms.%0A++++let+baz_loop+%3D+async+%7B%0A++++++++loop+%7B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%285%29%29.await%3B%0A++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++baz%28%29.await%3B%0A++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++%7D%0A++++%7D%3B%0A++++join_maybe%28bar%28%29%2C+baz_loop%29.await%3B%0A++++println%21%28%22...and+then+we+exit.%22%29%3B%0A%7D>

```rust
#[tokio::main]
async fn main() {
    // While `bar` is running, call `baz` every 5 ms.
    let baz_loop = async {
        loop {
            sleep(Duration::from_millis(5)).await;
            baz().await;
        }
    };
    join_maybe(bar(), baz_loop).await;
}
```

That's a bit cleaner. But note that the `select!` macro is [extremely
flexible][mini_redis],[^flexible] and this is just one of its simplest use
cases. There's a wide open design space for other helper functions that mix
joining, selecting, cancellation, shared mutability, and `no_std` support in
different ways. For an example of another macro in this space, see
[`join_me_maybe::join!`][join_me_maybe].

[join_me_maybe]: https://docs.rs/join_me_maybe/latest/join_me_maybe/

[^flexible]: If we reject examples that violate the new/clarified `Future`
    contract in this RFC, `select!` becomes _considerably_ less flexible.
    However, the combination of `if` guards, `biased`/fair modes, and
    overlapping mutability in different arms (the arm bodies lower to a
    `match`, and different match arms can mutate the same variables) is still
    hard to compete with.

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

We could say that `future2` is snoozed here across the first await. On the
other hand, `future2` has never been polled (or even pinned), and it's not
likely to be holding any exclusive resources in its initial state. We could
imagine giving `foo` e.g. a `MutexGuard` argument, but in that case the caller
could see what's going on. To create a non-local problem, we'd need to write
`foo` in a sync-then-async style, like this:

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
[`select!`]: https://tokio.rs/tokio/tutorial/select
[`timeout`]: https://docs.rs/tokio/latest/tokio/time/fn.timeout.html
[`AsyncIterator`]: https://doc.rust-lang.org/core/async_iter/trait.AsyncIterator.html
[mini_redis]: https://smallcultfollowing.com/babysteps/blog/2022/06/13/async-cancellation-a-case-study-of-pub-sub-in-mini-redis/
