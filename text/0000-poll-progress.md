- Feature Name: `poll_progress`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Add a required `poll_progress` method to the [`AsyncIterator`] trait ([`RFC
2996`]). Make `for await` loops ([#118898][for_await]) call `poll_progress`
while an `.await` in their body is `Pending`. Also, in `async gen` blocks and
functions ([#117078][gen_blocks]), make `for await` loops call `poll_progress`
while control is paused at a `yield` in their body. Expand the documented
`AsyncIterator` contract to require other adapters and consumers to behave the
same way. To emphasize the contract requirements of `poll_next`, define a new
`PollNext` enum for it to return, replacing `Poll<Option<_>>`.

[`AsyncIterator`]: https://doc.rust-lang.org/std/async_iter/trait.AsyncIterator.html
[`RFC 2996`]: https://rust-lang.github.io/rfcs/2996-async-iterator.html
[for_await]: https://github.com/rust-lang/rust/issues/118898
[gen_blocks]: https://github.com/rust-lang/rust/issues/117078

## Motivation
[motivation]: #motivation

Since the `AsyncIterator` trait is still unstable, iteration in the async
ecosystem today is organized around the [`Stream`] trait from the `futures`
crate. Apart from the name, the two traits currently have the same shape:

[`Stream`]: https://docs.rs/futures/latest/futures/prelude/trait.Stream.html

```rs
pub trait AsyncIterator { // or `pub trait Stream`
    type Item;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) { ... }
}
```

In this interface, everything an iterator does happens in `poll_next`. Consider
how that works in the context of a `for await` loop:

```rs
for await _ in my_iter {
    do_work().await;
}
```

When control is at the top, the `for await` loop calls `my_iter.poll_next`
until it either yields an item with `Ready(Some(_))` or indicates that it's
done with `Ready(None)`. Once control moves into `do_work`, though, the loop
stops driving `my_iter` entirely. That applies necessary "backpressure" to
async iterators, and it's mostly by design. But it can be a problem if
`my_iter` wraps multiple concurrent futures or other iterators internally,
because suspending some of those at arbitrary `.await` points isn't generally
correct. Here's an example where this causes a deadlock that looks like it
should be impossible in a straight-line reading of the code:

```rs
// `do_work` takes a private lock, sleeps briefly, and releases it. A deadlock here shouldn't be possible.
async fn do_work() {
    static LOCK: Mutex<()> = Mutex::const_new(());
    let _guard = LOCK.lock().await;
    sleep(Duration::from_millis(10)).await;
}

#[tokio::main]
async fn main() {
    // `my_iter` combines two child iterators, each of which wraps a `do_work` future.
    let my_iter = once(do_work()).merge(once(do_work()));
    for await _ in my_iter {
        // This deadlocks! One of the `do_work` futures above is holding `LOCK`, but we've stopped polling it.
        do_work().await;
    }
}
```

[`Merge`] doesn't implement `AsyncIterator` today, so this example doesn't
compile as written, but you can run it on stable by replacing `for await` with
`for_each` ([playground link][for_each_playground]). When control enters the
loop body, the right side of the `Merge` still holds a `do_work` future, which
is suspended at the point where it has tried to acquire `LOCK` and taken a spot
in its waiters queue. The call to `do_work` in the body tries to acquire `LOCK`
again, but the waiter at the front of the queue never wakes up, so it's
deadlocked. This problem often comes up with ["fancy" async iterators like
`FuturesUnordered` or `buffered` streams][barbara], but the example above uses
`merge` to emphasize that simple concurrent combinators have the same problem.

[for_each_playground]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3A%7BStreamExt+as+_%2C+once%7D%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0Ause+tokio_stream%3A%3AStreamExt+as+_%3B%0A%0A%2F%2F+%60do_work%60+takes+a+private+lock%2C+sleeps+briefly%2C+and+releases+it.%0A%2F%2F+A+deadlock+here+shouldn%27t+be+possible.%0Aasync+fn+do_work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+my_iter+%3D+once%28do_work%28%29%29.merge%28once%28do_work%28%29%29%29%3B%0A++++my_iter%0A++++++++.for_each%28%7C_%7C+async+%7B%0A++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++do_work%28%29.await%3B%0A++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++%7D%29%0A++++++++.await%3B%0A%7D>

To avoid these sorts of deadlocks, and other hard-to-diagnose hangs and
latencies, concurrent async iterators need to continuously drive the futures
they contain. That means that `for await` and other consumers need a way to let
iterators make internal progress, even when they're not ready to take the next
item.

This is where `poll_progress` comes in. How does it break the deadlock above?
There's a lot of async machinery involved, most of which already exists today,
but here's a partial sequence of events ([playground
link][poll_progress_playground]):

[poll_progress_playground]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=4774708bc6f57fc8a65d8af430ce2440

1. When `do_work` in the loop body sees that `LOCK` is already taken, it adds
   its `Waker` to the lock's waiters queue before reporting `Pending`.
2. **New:** Because control in `main` is in the body of a `for await` loop,
   `main` calls `poll_progress` on the loop's iterator before reporting
   `Pending` itself.
3. **New:** `Merge::poll_progress` polls its children, and its second child
   finishes acquiring `LOCK`, registers its own `Waker` with the runtime's
   sleep/timer system, and returns `Pending`. `Merge::poll_progress` then
   returns `Pending` itself, and `main`'s `poll` function also returns
   `Pending`.
4. After 10 ms, the runtime invokes the sleeping `Waker` from step 3 and polls
   `main` again.
5. `do_work` in the loop body tries to acquire the lock again, fails again, and
   returns `Pending` again.
6. **New:** As in step 2, `main` calls `Merge::poll_progress` again. It polls
   its children, and this time the second child drops its lock guard and yields
   an item. `Merge` buffers the item, `()` in this case.
7. Dropping that guard invokes the `Waker` that the body registered in step 1
   (and reregistered in step 5), so the runtime re-polls `main` immediately.
8. This time `do_work` in the loop body successfully acquires `LOCK` and starts
   its own sleep. There's no further lock contention, and after a couple
   iterations the loop finishes, and `main` returns.

This is a lot of low-level detail, but at a high level, `poll_progress` gives
us a new guarantee: Whenever we start running an `async fn`, it will _keep
running_ until it's done (or cancelled). That makes reasoning about async
locking "merely" as difficult as regular locking plus cancellation, as opposed
to even more difficult than that.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `AsyncIterator` intro

As its name suggests, `AsyncIterator` is the async version of [`Iterator`].
`Iterator` is the trait at the heart of `for` loops and other forms of
synchronous iteration, and `AsyncIterator` is the trait at the heart of `for
await` loops and other forms of asynchronous iteration.

[`Iterator`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html

Async code can and often does use regular iterators and `for` loops too, as
long as their next item is always immediately available, like loops over a
range or a collection. But async functions shouldn't use iterators like
[`std::io::Lines`] or [`std::sync::mpsc::Iter`] that sometimes block waiting
for input. The main purpose of `AsyncIterator` is to support iterators that
wait for input in a way that's compatible with async code.

[`std::io::Lines`]: https://doc.rust-lang.org/std/io/struct.Lines.html
[`std::sync::mpsc::Iter`]: https://doc.rust-lang.org/std/sync/mpsc/struct.Iter.html

Async iterators also have a superpower that regular iterators generally do not.
They can keep working "in the background" while the caller is processing an
item. For example:

```rs
for await jpeg in fetch_images() {
    save_image(jpeg).await;
}
```

Depending on how it's implemented, `fetch_images` could start downloading the
next `jpeg` concurrently while control is inside `save_image`. A regular
iterator could do that with threads, but threads complicate borrowing and
short-circuiting and usually require heap allocation. Async iterators can do
concurrent background work without threads or allocations, with full support
for local borrowing and intuitive behavior for `break` and `return` (cancelling
the background work).

### Implementing `AsyncIterator`

The core of the `AsyncIterator` trait is the `poll_next` and `poll_progress`
methods, which look like this:

```rs
trait AsyncIterator {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> PollNext<Self::Item>;
    fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()>;
}

enum PollNext<Item> {
    Item(Item),
    Pending,
    Done,
}
```

`poll_next` is a cross between `Iterator::next` and `Future::poll`. Like
`Iterator::next`, it can return an item or indicate that the iterator is done.
And like `Future::poll`, it can also return `Pending`, meaning that no item is
currently ready and a wakeup has been registered. When a `for await` loop wants
the next item from an async iterator, it calls `poll_next`.

`poll_progress` is what allows an async iterator to do background work when the
caller isn't ready for another item. It returns `Pending` when more progress
might be possible in the future, and it also registers a wakeup in that case.
When no more internal progress is possible without calling `poll_next` again,
`poll_progress` returns `Ready`. When an `.await` in the body of a `for await`
loop is pending, the loop calls `poll_progress` on its iterator before
reporting pending itself.

Like `Future::poll`, the return values of these methods impose certain
requirements on their callers. But while `Future` only has two possible returns
to contend with, `AsyncIterator` has _five_:

| Previous return | Requirements |
| --- | --- |
| `poll_next` returned `PollNext::Item` | You must <ins><strong>poll again promptly,</strong></ins> either `poll_next` if you want another item or `poll_progress` if not. |
| `poll_next` returned `PollNext::Pending` | You must `poll_next` again after wakeup. <ins><strong>`poll_progress` is not allowed,</strong></ins> and it might panic or otherwise misbehave (within the bounds of safe code). |
| `poll_next` returned `PollNext::Done` | You must not poll again. Either `poll_next` or `poll_progress` might panic or misbehave. |
| `poll_progress` returned `Poll::Pending` | You can `poll_next` when you want another item, otherwise you must `poll_progress` again after wakeup. |
| `poll_progress` returned `Poll::Ready` | No requirements. You can `poll_next` when you want another item, or you can stop polling. `poll_progress` is allowed but usually has no effect. |

An `AsyncIterator` begins life in the first state above, expecting a prompt
call to either `poll_next` or `poll_progress`. Although a `for await` loop will
always start by calling `poll_next`, other consumers (e.g. [`Chain`], on its
right side) may start with `poll_progress`. Also, cancelling an async iterator
by dropping it is allowed at any time.

[`Chain`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.chain

The first two requirements in the table above are underlined, because they're
the most surprising. Expanding on those:

- **The `PollNext::Item` rule:** When `poll_next` returns an item, it's not
  required to register any wakeups. That's important for performance, because
  there might be many items ready to return, and we don't want to trigger
  redundant wakeups for each of them or cause an extra round trip through the
  executor. (This much is also true of `Stream` today.) On the other hand,
  if/when we don't want to call `poll_next` anymore, we need to call
  `poll_progress` before returning pending ourselves, to give the iterator a
  chance to finish its own polling responsibilities and register wakeups.
- **The `PollNext::Pending` rule:** When `poll_next` is pending, we have to
  `poll_next` again at the next wakeup; we can't change our minds about wanting
  an item and switch to `poll_progress`. (The only valid way to change our
  minds is to cancel the whole iterator by dropping it.) This rule is aimed at
  concurrent combinators that "merge" multiple async iterators together. After
  a child iterator yields an item, those combinators should keep driving their
  other children with `poll_next` internally until each of them has yielded an
  item. That ensures the smooth flow of control through chains of adapters and
  combinators, and it means that most async iterators don't need to allocate
  buffer space for an item. The rationale section [discusses this rule
  further](#why-not-allow-poll_progress-at-any-time).

Let's look at the implementation of an async iterator adapter, `Buffer1`, which
wraps another `AsyncIterator` and pre-fetches its next item in `poll_progress`.
This turns any `AsyncIterator` into a concurrent one that does work in the
background:

```rs
struct Buffer1<Iter: AsyncIterator> {
    #[pin] // TODO: a standard way to do structural pinning in examples?
    inner: Fuse<Iter>,
    item: Option<Iter::Item>,
}

impl<Iter: AsyncIterator> AsyncIterator for Buffer1<Iter> {
    type Item = Iter::Item;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> PollNext<Iter::Item> {
        let this = self.project();
        if let Some(item) = this.item.take() {
            PollNext::Item(item) // If we have a buffered `item`, return it.
        } else {
            this.inner.poll_next(cx) // Otherwise poll the `inner` async iterator.
        }
    }

    fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
        let mut this = self.project();
        if this.item.is_none() {
            match this.inner.as_mut().poll_next(cx) { // We don't have a buffered `item`. Try to get one.
                PollNext::Item(item) => {
                    *this.item = Some(item);
                    this.inner.poll_progress(cx) // `poll_progress` is necessary.
                }
                PollNext::Pending => Poll::Pending, // `poll_progress` is forbidden.
                PollNext::Done => Poll::Ready(()), // `inner` is fused.
            }
        } else {
            this.inner.poll_progress(cx) // We already have a buffered `item`.
        }
    }
}
```

Look at how `Buffer1::poll_next` returns the result of `inner.poll_next`
directly, without doing any further polling. That might seem like it violates
the "`PollNext::Item` rule" about polling again promptly, but note that the
same rule applies _to the caller_. The caller must poll again, and when they
do, `inner` will get polled again. However, `Buffer1::poll_progress` doesn't
impose the same requirements on its caller, so it does need to call
`inner.poll_progress` in its `Item` branch. (The `inner` iterator is
["fused"][`Fuse`], so the `Done` state is handled automatically.)

[`Fuse`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.fuse

Note also that `Buffer1::poll_progress` follows the "`PollNext::Pending` rule"
and doesn't call `inner.poll_progress` after `inner.poll_next` returns
`Pending`. The next poll will be either `Buffer1::poll_next` if the caller
wants another item, or else `Buffer1::poll_progress` calling `inner.poll_next`
again after a wakeup. When we do have a buffered item, we know the last call to
`inner.poll_next` returned `Item`, and we forward `poll_progress`.

### [`FuturesUnordered`] docs

`FuturesUnordered` is an async iterator over the results of the futures it
contains. Looping over it gives you each of the results as they arrive, and
pending futures keep making progress concurrently with the loop body. For
example:

```rs
use futures::stream::FuturesUnordered;
use tokio::time::{Duration, sleep};

async fn foo(secs: u64, message: &str) -> &str {
    sleep(Duration::from_secs(secs)).await;
    println!("foo {message} woke up");
    message
}

#[tokio::main]
async fn main() {
    let futures = FuturesUnordered::new();
    futures.push(Box::pin(foo(0, "A")));
    futures.push(Box::pin(foo(1, "B")));
    for await message in futures {
        println!("got {message}");
        sleep(Duration::from_secs(2)).await;
    }
}
```

The first `foo` future does a zero-length sleep, so we see `foo A woke up` and
`got A` immediately. The second `foo` future runs concurrently, so we see `foo
B woke up` after one second, while the loop body is in the middle of its own
two-second sleep. The output of the second `foo` future is buffered, and once
the two-second sleep is finished, we loop around and print `got B` immediately.

(See "What about `.next()`?" below for a more complicated case.)

### [`tokio_stream::StreamExt::merge`] docs

[`tokio_stream::StreamExt::merge`]: https://docs.rs/tokio-stream/latest/tokio_stream/trait.StreamExt.html#method.merge

Combine two streams into one by interleaving the outputs of both as they're
produced. When one side of the merge yields an item, the other side continues
making progress in the background until it also yields an item. For example:

```rs
use futures::stream::once;
use tokio::time::{Duration, sleep};
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let stream_a = once(async { "A" });
    let stream_b = once(async {
        sleep(Duration::from_secs(1)).await;
        println!("stream_b woke up");
        "B"
    });
    for await item in stream_a.merge(stream_b) {
        println!("got {item}");
        sleep(Duration::from_secs(2)).await;
    }
}
```

Here we see `got A` immediately, and then the loop body starts its two-second
sleep. The other stream continues in the meantime, so we see `stream_b woke up`
after one second, and the item `"B"` is buffered. When the two-second sleep is
finished, we loop around and print `got B` immediately.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `for await`

We can't desugar this RFC's version of `for await` syntax into a loop (the way
the Reference does for [the `for` keyword][for]), because we'd need to treat
the body as a future (e.g. `async { /* loop body */ }`) to intercept `Pending`
and call `poll_progress` in the right place, but that's incompatible with
`return`, `?`, `break`, or `continue` in the body.

[for]: https://doc.rust-lang.org/reference/expressions/loop-expr.html#r-expr.loop.for.desugar

Instead we can describe the steps `for await` takes internally, [the same way
`.await` is described][await]. To evaluate the expression:

[await]: https://doc.rust-lang.org/reference/expressions/await-expr.html#r-expr.await.effects

```
'label: for await PATTERN in ITER_EXPR {
    /* loop body */
}
```

We execute the following steps:

1. Create an async iterator with
   `IntoAsyncIterator::into_async_iter(ITER_EXPR)`.
2. Pin it using `Pin::new_unchecked`.
3. Poll it by calling the `AsyncIterator::poll_next` method and passing it the
   current task context.
4. If the call to `poll_next` returns `PollNext::Pending`, then the surrounding
   async context returns `Poll::Pending` (or in an `async gen fn`,
   `PollNext::Pending`), suspending its state so that, when it's re-polled,
   execution returns to step 3.
5. If the call to `poll_next` returns `PollNext::Done`, then the loop drops the
   async iterator and evaluates to `()`. Skip the remaining steps.
6. If the call to `poll_next` returns `PollNext::Item(item)`, then `item` is
   matched against the irrefutable `PATTERN`.
7. At the top of the loop body, declare a local `bool` variable (not visible to
   user code, but let's call it `progress_pending`) and initialize it to
   `true`. This variable is redeclared in each loop iteration and doesn't
   logically persist across iterations.
8. Control proceeds through the loop body, with the bindings from `PATTERN` in
   scope. When any `.await` expression or `for await` item (i.e. step 3 above,
   if another `for await` loop is nested within this one) in the body is
   pending, if `progress_pending` is still `true`, then call `poll_progress` on
   the async iterator before reporting pending from the surrounding async
   context.
9. If `poll_progress` returned `Ready`, set `progress_pending` to `false`.
10. \[`break`, `continue`, and looping and diverging control flow as usual\]

Note that if `for await` loops are nested, a pending expression in an inner
loop triggers steps 8 and 9 above for _all_ the containing loops, starting with
the innermost. Each loop tracks its own `progress_pending` flag reflecting the
state of its own iterator.

The `progress_pending` flag avoids unnecessary polling after no more progress
can be made, and the `ForEach` implementations in this RFC's playground
examples use a similar flag. However, the `AsyncIterator` contract doesn't
strictly require this sort of tracking. Continuing to call `poll_progress`
after it returns `Ready` is allowed, and `AsyncIterator` implementations should
tolerate it, generally with no effect.

### `async gen fn`

The `AsyncIterator` contract limits the cases where it's valid to call
`poll_progress`. There are only two:

1. `poll_next` has never been called.
2. The previous call to `poll_next` returned `Item`.

In an `async gen fn` -- that is, in the anonymous `AsyncIterator`
implementation returned by an `async gen fn` -- those cases correspond to the
initial state where control hasn't yet entered the body, and to the states
where control is suspended at a `yield` expression. Importantly, in the states
where control is suspended at a pending `.await` or `for await`, we know that
the previous call to `poll_next` returned `Pending`, so calling `poll_progress`
is not valid. (The implementation panics if it receives such a call, though
that isn't required by the contract.) This gives us an important
simplification: **Only `poll_next` advances control through the body of an
`async gen fn`.**

In contrast, `poll_progress` has a limited responsibility: If control is
suspended at a `yield` in the body of a `for await` loop, then the `async gen
fn`'s implementation of `poll_progress` calls `poll_progress` on that loop's
iterator. If `for await` loops are nested, there may be multiple such
iterators, and `poll_progress` gets forwarded to all of them, starting with the
innermost. If any of the forwarded calls returns `Pending`, then
`poll_progress` returns `Pending`, otherwise it returns `Ready`. (See the
unresolved questions section for how `progress_pending` interacts with this.)

Note that when control reaches a `yield`, that by itself does not necessarily
imply a call to `poll_progress` on any active iterators. If the caller doesn't
encounter a pending `.await`, then nothing will call `poll_progress`, and
control will resume from the `yield` promptly instead (unless cancelled). In
terms of the `AsyncIterator` contract, that's the case where a call to
`poll_next` that returns `Item` is followed by another prompt call to
`poll_next`, rather than a call to `poll_progress`.

For example:

```rs
async gen fn foo() {
    for await _ in bar() {
        for await _ in baz() {
            // While this expression is pending, `foo`'s `poll_next` function calls `poll_progress` on the
            // `baz` and `bar` async iterators before reporting pending itself. The `AsyncIterator` contract
            // forbids calling `foo`'s `poll_progress` function in this state, and `foo` will panic if you do.
            do_work().await;

            // While control is suspended at this `yield`, `foo`'s `poll_progress` function calls
            // `poll_progress` on the `baz` and `bar` async iterators, and it reports pending if either of
            // those calls is pending. Note that reaching this `yield` does *not* automatically call
            // `poll_progress` on `baz` or `bar`. That call only comes if/when the caller encounters a pending
            // await in their own loop and calls `poll_progress` on `foo`.
            yield;
        }
    }
}
```

## Drawbacks
[drawbacks]: #drawbacks

### What about `.next()`?

The main way users interact with `Stream` in the async ecosystem today is the
[`StreamExt::next`] method, which returns a future representing the next item
in the stream. But the `AsyncIterator` contract proposed in this RFC isn't
compatible with `next`, because the `Next` future is short-lived, and it can't
keep polling its iterator after it yields an item. **`AsyncIterator` won't have
a `next` method,** and we'll need a migration plan for existing callers.

First, the problem. Here's a version of [the `merge` deadlock
above][motivation] that `poll_progress` can't fix ([playground link][next1]):

[`StreamExt::next`]: https://docs.rs/futures/latest/futures/prelude/stream/trait.StreamExt.html#method.next
[next1]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3Aonce%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0Ause+tokio_stream%3A%3AStreamExt+as+_%3B%0A%0A%2F%2F+%60do_work%60+takes+a+private+lock%2C+sleeps+briefly%2C+and+releases+it.%0A%2F%2F+A+deadlock+here+shouldn%27t+be+possible.%0Aasync+fn+do_work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+my_iter+%3D+pin%21%28once%28do_work%28%29%29.merge%28once%28do_work%28%29%29%29%29%3B%0A++++_+%3D+my_iter.next%28%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++do_work%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rs
let mut my_iter = pin!(once(do_work()).merge(once(do_work())));
_ = my_iter.next().await;
do_work().await; // Deadlock!
```

In theory we're supposed to call `my_iter.poll_progress` while
`do_work().await` is pending, but there isn't a `for await` loop doing that for
us, and the `Next` future is gone. On top of that, here's another deadlock
without the `merge`, where the problem is cancellation rather than concurrency
([playground link][next2]):

[next2]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3Aonce%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0Ause+tokio_stream%3A%3AStreamExt+as+_%3B%0A%0A%2F%2F+%60do_work%60+takes+a+private+lock%2C+sleeps+briefly%2C+and+releases+it.%0A%2F%2F+A+deadlock+here+shouldn%27t+be+possible.%0Aasync+fn+do_work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+my_iter+%3D+pin%21%28once%28do_work%28%29%29%29%3B%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+my_iter.next%28%29%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++do_work%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rs
let mut my_iter = pin!(once(do_work()));
_ = tokio::time::timeout(Duration::from_millis(1), my_iter.next()).await;
do_work().await; // Deadlock!
```

In both cases, there's an `AsyncIterator` that's ready to make progress -- in
the first case `LOCK` has already invoked a `Waker`, and in the second case
`sleep` eventually does -- and somebody needs to poll it or drop it promptly.
But who? The caller? What we're seeing is that, to satisfy the proposed
contract, whatever's driving an `AsyncIterator` generally needs to _own_ it.
That's the case with `for await` loops, and with terminal consumers like
[`for_each`] and [`collect`], but it's a problem for `next`.

[`for_each`]: https://docs.rs/futures/latest/futures/prelude/stream/trait.StreamExt.html#method.for_each
[`collect`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.collect

Now, the alternatives. The most common use case for `next` today is loops like
this:

```rs
let mut stream = pin!(...);
while let Some(item) = stream.next().await { ... }
```

These can be replaced with `for await` loops:

```rs
for await item in stream { ... }
```

The `for await` version is more concise and doesn't require pinning, so it's a
nice improvement where it works. Unfortunately, it doesn't work everywhere.
Here's an example of a `next` caller that can't easily switch to `for await`
([playground link][loop_select]):

[loop_select]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3AStreamExt%3B%0Ause+futures%3A%3Astream%3A%3AFuturesUnordered%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+work%28%29+%7B%0A++++sleep%28Duration%3A%3Afrom_secs%28rand%3A%3Arandom_range%280..5%29%29%29.await%3B%0A%7D%0A%0Aasync+fn+more_work%28%29+-%3E+impl+Future%3COutput+%3D+%28%29%3E+%7B%0A++++sleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A++++work%28%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+futures+%3D+FuturesUnordered%3A%3Anew%28%29%3B%0A++++loop+%7B%0A++++++++select%21+%7B%0A++++++++++++%2F%2F+Add+more+jobs+as+they+come+in.%0A++++++++++++job+%3D+more_work%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22got+a+job%22%29%3B%0A++++++++++++++++futures.push%28job%29%3B%0A++++++++++++%7D%0A%0A++++++++++++%2F%2F+Handle+the+outputs+of+running+jobs.%0A++++++++++++Some%28_%29+%3D+futures.next%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22finished+a+job%22%29%3B%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D>

```rs
let mut futures = FuturesUnordered::new();
loop {
    select! {
        // Add more jobs as they come in.
        job = more_work() => futures.push(job),

        // Handle the outputs of running jobs.
        Some(output) = futures.next() => ...
    }
}
```

This caller is iterating over a `FuturesUnordered` and also adding more work to
it in the loop body. We can't reorganize this around a `for await`, because
that would take ownership of `futures`, and `futures.push(job)` wouldn't
compile. Sometimes we can replace things like this with task spawning, but
there are cases where the borrowing and mutability details make that [very
difficult][mini_redis].

In the difficult cases, we can recreate the `next` method in a
`poll_progress`-compatible way using a macro. Here's a [working
proof-of-concept][drive], which takes ownership of an async iterator and
provides a handle with `next` and `with_mut` methods on it. Apart from easing
migration, the macro also fixes [potential deadlocks in the loop
above][loop_select_deadlock]. The macro is `no_std`-compatible, but it's also
quite complicated. We could consider adding something like it to `core`
someday, but this RFC doesn't propose doing that at first.

[loop_select_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3AStreamExt%3B%0Ause+futures%3A%3Astream%3A%3AFuturesUnordered%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_secs%28rand%3A%3Arandom_range%280..5%29%29%29.await%3B%0A%7D%0A%0Aasync+fn+more_work%28%29+-%3E+impl+Future%3COutput+%3D+%28%29%3E+%7B%0A++++sleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A++++work%28%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+futures+%3D+FuturesUnordered%3A%3Anew%28%29%3B%0A++++loop+%7B%0A++++++++select%21+%7B%0A++++++++++++%2F%2F+Add+more+jobs+as+they+come+in.%0A++++++++++++job+%3D+more_work%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22got+a+job%22%29%3B%0A++++++++++++++++futures.push%28job%29%3B%0A++++++++++++%7D%0A%0A++++++++++++%2F%2F+Handle+the+outputs+of+running+jobs.%0A++++++++++++Some%28_%29+%3D+futures.next%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22finished+a+job%22%29%3B%0A++++++++++++++++work%28%29.await%3B+%2F%2F+Deadlock%21%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D>

### What about the blanket impls?

`AsyncIterator` currently has blanket impls that include [`&mut
I`][async_iter_blanket_mut] and [`Pin<&mut I>`][async_iter_blanket_pin]. **We
should remove these impls,** because driving an `AsyncIterator` by reference is
deadlock-prone. (We'll keep the boxed ones.) The problem is similar to what we
just saw with `next` above ([playground link][blanket_deadlock]):

[async_iter_blanket_mut]: https://doc.rust-lang.org/std/async_iter/trait.AsyncIterator.html#impl-AsyncIterator-for-%26mut+S
[async_iter_blanket_pin]: https://doc.rust-lang.org/std/async_iter/trait.AsyncIterator.html#impl-AsyncIterator-for-Pin%3CP%3E
[blanket_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3A%7BStreamExt%2C+once%7D%3B%0Ause+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0Ause+tokio_stream%3A%3AStreamExt+as+_%3B%0A%0A%2F%2F+%60do_work%60+takes+a+private+lock%2C+sleeps+briefly%2C+and+releases+it.%0A%2F%2F+A+deadlock+here+shouldn%27t+be+possible.%0Aasync+fn+do_work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+my_iter+%3D+pin%21%28once%28do_work%28%29%29.merge%28once%28do_work%28%29%29%29%29%3B%0A++++%2F%2F+Replace+this+loop+with+a+%60Stream%60+equivalent+that+runs+today.%0A++++%2F%2F+for+await+_+in+my_iter+%7B%0A++++%2F%2F+++++break%3B%0A++++%2F%2F+%7D%0A++++StreamExt%3A%3Atake%28my_iter%2C+1%29.for_each%28async+%7C_%7C+%7B%7D%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++do_work%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rs
let mut my_iter = pin!(once(do_work()).merge(once(do_work())));
for await _ in my_iter {
    break;
}
do_work().await; // Deadlock!
```

Again `poll_progress` is no help, because the deadlock is happening after our
`for await` loop is finished. Normally `for await` would drop the iterator when
it short-circuits, but the `pin!` here means we're actually looping over a
`Pin<&mut _>` reference, and dropping that has no effect. This is another
example of how driving an `AsyncIterator` correctly requires _ownership_.
`Pin<&mut _>` should not implement `AsyncIterator`, and this loop should not
compile.

### What about "lending" async iterators?

A hypothetical `LendingAsyncIterator` would yield items that borrow the
iterator itself. That could let us share a mutable buffer between items, for
example, in exchange for not being allowed to `collect` those items. RFC 2996
[discussed this possibility][lending].

[lending]: https://rust-lang.github.io/rfcs/2996-async-iterator.html#lending-async-iterators

However, `poll_progress` isn't compatible with lending. If the loop body
borrows the iterator, but we also want to call `poll_progress` whenever the
body is pending, that's an unavoidable borrowck conflict. Probably any approach
to supporting "background work" would have the same problem. (Unless both
`poll_progress` and lending iterators used shared references and interior
mutability? Unlikely.)

The need for `poll_progress` came from the following assumptions:

1. We want async iterators like `Merge` and `FuturesUnordered` to wrap multiple
   concurrent iterators or futures and yield their results as they come.
2. We can't tolerate suspending async iterators or futures at random await
   points.

`LendingAsyncIterator` can't do much about the second assumption, but it might
be able to attack the first, either by not wrapping multiple futures, or by
waiting for all the futures it's running to finish before yielding control.
That could be a useful abstraction in many cases, but giving up on `Merge`
seems like too much of a sacrifice for the standard trait that will power `for
await` and `async gen fn`.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Is pausing futures at `.await` points so terrible? Could we instead agree to allow it?

The fundamental assumption of this RFC is that we need to guarantee smooth
control flow through async code. Concretely, `Future` and `AsyncIterator`
implementations should be able to assume that they'll be polled again promptly
(or dropped) whenever they invoke their `Waker`.

One argument for this assumption is an analogy to multithreading. ["Everybody
knows"](https://jacko.io/snooze.html#threads) that we can't pause or cancel
running threads. If a paused or cancelled thread happens to be holding any
locks, we'll probably deadlock ourselves. In the dark corners of systems
programming where we do pause threads, like Unix signal handlers, we have to
take extraordinary care not to touch any locks in that critical section. That
means onerous restrictions like no allocating memory and no printing.

Async Rust can handle cancellation better than threads do, because `Drop`
cleans up our lock guards. But _pausing_ a Rust future isn't much different
from pausing a thread. It can only happen at an `.await` point, so we don't
have to worry about the `malloc` lock, but for any exclusive resource that
might be held across an `.await`, the story is the same. In
["Futurelock"][futurelock], it was a semaphore buried inside Tokio's channel
implementation. If async Rust code has to tolerate indefinite pauses, then we
need to be defensive about futurelocks everywhere, and writing async
applications will start to feel like writing Unix signal handlers.

### Why not allow `poll_progress` at any time?

The "`PollNext::Pending` rule" says that `poll_progress` shouldn't be called
when `poll_next` is pending. That seems kind of arbitrary. What's the point of
that rule?

Consider the `poll_progress` implementation of a combinator like [`Merge`].
_Without_ the `PollNext::Pending` rule, it could be as short as this:

```rs
fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
    let this = self.project();
    // XXX: This version doesn't keep track of whether each child's `poll_next` is pending.
    let poll1 = this.left.poll_progress(cx);
    let poll2 = this.right.poll_progress(cx);
    any_pending([poll1, poll2])
}
```

That would be nice and simple (we'll call it a "lazy" `Merge`), but the
downside is that _every other_ async iterator would need to handle the case
where their caller abruptly switches from `poll_next` to `poll_progress` before
they yield an item. For example, think about this hypothetical `async gen fn`:

```rs
async gen fn foo() {
    do_work().await;
    yield;
}
```

If control is in the middle of `do_work`, we can't let it get stuck there, or
else we'll have the same deadlocks we saw above. If the caller could switch to
`poll_progress` at any time, then `poll_progress` would need to keep driving
control through the body up to the next `yield`. However, `poll_progress`
shouldn't _always_ drive control to the next `yield`. If it did, then `for
await` would drive _every_ async iterator concurrently with its loop body,
which isn't how it's supposed to work. Instead, `foo` would need to track
whether `poll_next` has been called _at least once_ before allowing control to
enter the body, and before allowing control to proceed after a `yield`. That's
doable, but then would we want equivalent behavior from adapters like [`Map`]
and [`Then`]? Those would need to add a state flag that they don't have today
-- call it `next_item_wanted` -- and they'd also need a rarely used buffer slot
for an item. (See the following section for detailed examples of all of this.)

[`Map`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.map
[`Then`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.then

In practice, the bookkeeping for "if `poll_next` has been called at least once,
`poll_progress` advances control to the next yield and buffers an item" would
look awfully similar to the bookkeeping for "don't call `poll_progress` when
`poll_next` is pending", except moved down a level from the caller to the
callee. Rather than adding complexity and buffer slots to concurrent async
iterators like `Merge` and [`StreamMap`], we'd add complexity and buffer slots
to _every_ async iterator.

With the `PollNext::Pending` rule, we don't get to write the "lazy" version of
`Merge::poll_progress` above, and instead we have to do state tracking and
conditional buffering there. That's a downside. The upside is that
`Then::poll_progress` looks like this:

```rs
fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
    assert!(self.future.is_none());
    self.project().stream.poll_progress(cx)
}
```

`Then::poll_progress` doesn't need to worry about what to do with
[`self.future`] or its output, because it can only be called when `self.future`
is `None`. Similarly, it never needs to call `stream.poll_next`, because it
will never be called after `poll_next` returns `Pending`. Simple adapters like
`Map` and `Then` are more common than concurrent ones like `Merge` and
`Buffer1`, both in terms of how many implementations we need to write and also
how many instances appear in chains of iterators. Keeping the simple things
simple is a good trade.

[`self.future`]: https://docs.rs/futures-util/0.3.33/src/futures_util/stream/stream/then.rs.html#14-20

Another upside of the `PollNext::Pending` rule compared to the alternative is
that it's clear when it's been violated, and we can write asserts like the one
above. An `async gen fn` should panic if we call `poll_progress` while it's
suspended at an `.await`, just like an `async fn` panics today if we poll it
again after it's returned.

### Implementing [`Map`] with and without the "`PollNext::Pending` rule"

The previous section claimed that adapters like `Map` get more complicated if
callers can switch from `poll_next` to `poll_progress` at any time. This
section illustrates that in detail. (If the previous section already makes
perfect sense, then this section will be a bit repetitive.) Consider the
following example, which doesn't use `Map` at first:

```rs
async gen fn slow_numbers() -> u32 {
    for i in 0..10 {
        sleep(Duration::from_millis(1)).await;
        yield i;
    }
}

async gen fn print_numbers() -> u32 {
    for await i in slow_numbers() {
        println!("NUMBER {i}");
        yield i;
    }
}

// prints "NUMBER 0" once, does *not* print "NUMBER 1"
for await _ in print_numbers() {
    sleep(Duration::from_millis(10)).await;
    break;
}

// prints "NUMBER 0" *twice*
for await _ in print_numbers().merge(print_numbers()) {
    sleep(Duration::from_millis(10)).await;
    break;
}
```

This program does one iteration in each of a couple of `for await` loops. The
first loop should print "NUMBER 0" once. (That's "obvious", but we'll come back
to it below.) The second loop should print "NUMBER 0" twice, initially when it
receives an item, and then again immediately when it starts its sleep. We
should expect this _regardless of the details of the `AsyncIterator` contract,_
because our design assumption is that we _never_ pause the flow of control at
an await point in an async function (or in this case, at a `for await` point in
an `async gen fn`). The sleep in `slow_numbers` is there to make sure that
`Merge` calls `poll_next` on both sides; we don't need to ask subtle questions
about what `Merge` does when one side is immediately ready.

With all that in mind, let's think about what would happen if we instead
implemented `print_numbers` like this, using `Map`:

```rs
fn print_numbers() -> impl AsyncIterator<Item = u32> {
    slow_numbers().map(|i| {
        println!("NUMBER {i}");
        i
    })
}
```

This is debatable, but let's take it for granted that we want the observable
behavior of that function to be *identical* to the `async gen fn` above.
Consider the following implementation of `Map` ([playground
link][playground_map]):

[playground_map]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=4c5d8c420500c6318f59db7d874a4dd9

```rs
struct Map<Iter, F> {
    #[pin] // TODO: a standard way to do structural pinning in examples?
    iter: Iter,
    f: F,
}

impl<Iter, F, T> AsyncIterator for Map<Iter, F>
where
    Iter: AsyncIterator,
    F: FnMut(Iter::Item) -> T,
{
    type Item = T;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> PollNext<Self::Item> {
        let this = self.project();
        this.iter.poll_next(cx).map(this.f) // `PollNext::map` is analogous to `Poll::map`.
    }

    fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
        self.project().iter.poll_progress(cx)
    }
}
```

That's nice and simple, but is it correct? Namely, does it print "NUMBER 0"
twice in our `Merge` loop? If we can assume the "`PollNext::Pending` rule",
then it is, and it does. Once `Merge::poll_next` has called `Map::poll_next`,
`Merge::poll_progress` must keep calling it until it gets an item. That means
this version of `Map` will print as we expect, without any other assumptions
about how `Merge` is implemented.

On the other hand, what if `Merge::poll_progress` only called `poll_progress`
on both its children? (I.e. the "lazy" version in the previous section.) Now
our simple `Map` implementation isn't equivalent to the `async gen fn` anymore.
We won't see the print where we expect it, because `Map::poll_progress` never
calls `f`. If we care about equivalent behavior here, then we'll need to fix
that, and we'll also need a buffer slot for the resulting item. However,
there's a footgun: `Map::poll_progress` shouldn't *always* try to fill that
buffer slot, or else the *first loop* above will start printing "NUMBER 1"
while its body is sleeping. If we want `Map` to behave exactly like the `async
gen fn`, but we have to tolerate a "lazy" `Merge`, then `Map` also needs a
`next_item_wanted` flag like this ([playground link][playground_map_lazy]):

[playground_map_lazy]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=34afc0a7a869083bfd0cc60565c514d7

```rs
struct Map<Iter, F, T> {
    #[pin]
    iter: Fuse<Iter>,
    next_item_wanted: bool,
    f: F,
    item: Option<T>,
}

impl<Iter, F, T> AsyncIterator for Map<Iter, F, T>
where
    Iter: AsyncIterator,
    F: FnMut(Iter::Item) -> T,
{
    type Item = T;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> PollNext<T> {
        let this = self.project();
        if let Some(item) = this.item.take() {
            return PollNext::Item(item);
        }
        match this.iter.poll_next(cx) {
            PollNext::Item(item) => {
                *this.next_item_wanted = false;
                PollNext::Item((this.f)(item))
            }
            PollNext::Pending => {
                *this.next_item_wanted = true;
                PollNext::Pending
            }
            PollNext::Done => PollNext::Done, // fused
        }
    }

    fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
        let mut this = self.project();
        if this.item.is_none() && *this.next_item_wanted {
            match this.iter.as_mut().poll_next(cx) {
                PollNext::Item(item) => {
                    *this.item = Some((this.f)(item));
                    *this.next_item_wanted = false;
                    this.iter.poll_progress(cx)
                }
                PollNext::Pending => Poll::Pending,
                PollNext::Done => Poll::Ready(()), // fused
            }
        } else {
            this.iter.poll_progress(cx)
        }
    }
}
```

This is complicated. And any other `AsyncIterator` that runs caller code that
might have observable side effects -- so maybe not e.g. `Take` or `Skip`, but
including e.g. `Then`, `Filter`, `TakeWhile`, and `SkipWhile` -- would also be
_at least_ this complicated. It's not clear whether we'd actually want the
`AsyncIterator` contract to require this sort of `async gen fn`-equivalent
behavior, because those requirements would be both hard to explain and also
hard to `assert!`. Realistically, we'd probably give up on the idea, and we'd
accept that control flow through chains of async iterators is inconsistent and
unpredictable, at least when callers care about exactly when side effects
happen. (It's also possible to come up with deadlock examples based on these
effects, but they're less realistic and more contrived than their `async fn`
counterparts, so we've stuck with printing in this section.)

The "`PollNext::Pending` rule" fixes this whole mess. It gives us a consistent,
high-level picture of how control flow in any `AsyncIterator` works: Control
"enters" an iterator when `poll_next` is initially called, and it "leaves" the
iterator when `poll_next` returns `Item` or `Done`. Once an iterator starts
"running", it continues uninterrupted (unless the whole thing is cancelled) to
its next yield point (if any). This works the same way whether it's an `async
gen fn` or a chain of combinators, and we can document it and teach it. For
non-concurrent iterators, all `poll_progress` needs to do is forward itself to
any children. The cost of enforcing this rule falls on concurrent combinators
like `Merge` and `StreamMap`, which must track which of their children are
"running" and keep running them until they give up control.

### Is it worth having a whole new `PollNext` enum?

Today's `Poll<Option<_>>` return type captures the three possible return states
(item, pending, and done), but it doesn't represent anything about the
`poll_next`/`poll_progress` contract. Compare that to the `Future::poll`
method. Rust could've defined `poll` to return `Option<Output>`, but the
`Future` contract is important and subtle enough that it was worth adding a
separate type to represent it.

The same is true of `poll_next` in the new contract. The "`PollNext::Item`
rule" and the "`PollNext::Pending` rule" are important and subtle enough that
it's worth defining a distinct return type to represent them.

### Could we weaken "`poll_progress` is not allowed" to "`poll_progress` is not sufficient"?

By forbidding `poll_progress` after `poll_next` returns `Pending`, we give
implementations permission to panic in that case. But if not for explicit
asserts to enforce the rule, the `poll_progress` call itself would probably be
harmless in practice. Some child futures might get polled before asking for a
wakeup, but that's fine. Why not allow it?

The main reason not to allow it is that (like the `Future` contract) the
`AsyncIterator` contract is subtle and not enforced by the compiler. Folks
writing low-level async iterators for the first time are unlikely to read all
the docs, and a panic that says "there's a rule you didn't know about" is
better than confusing control flow bugs. We can't easily do this for the
`PollNext::Item` rule, and callers who overlook that one will miss wakeups, but
for the `PollNext::Pending` rule we can do it.

## Prior art
[prior-art]: #prior-art

Various approaches to this feature, and the hangs and deadlocks that motivate
it, have been discussed for many years. Some points of reference:

- ["Footgun lurking in `FuturesUnordered` and other concurrency-enabling streams"](https://github.com/rust-lang/futures-rs/issues/2387)
- ["Barbara battles buffered streams"][barbara]
- ["`for await` and the battle of buffered streams"](https://tmandry.gitlab.io/blog/posts/for-await-buffered-streams/)
- https://without.boats/blog/poll-progress
- ["A discussion prototype for `AsyncIterator`"](https://hackmd.io/EplPcmBCTCSQ9LPOU6IZDw) and related [meeting notes](https://hackmd.io/bYaPiCdqR3WIyjKhFWzdtQ)
- [`futures_concurrency::concurrent_stream::Consumer::progress`](https://docs.rs/futures-concurrency/latest/futures_concurrency/concurrent_stream/trait.Consumer.html#tymethod.progress)
- ["Future's liveness problem"](https://skepfyr.me/blog/futures-liveness-problem/)
- ["Futurelock"][futurelock]
- ["Never snooze a future"](https://jacko.io/snooze.html)

## Unresolved questions
[unresolved-questions]: #unresolved-questions

### What other names should we consider for `poll_progress`?

`poll_progress` is the most common name folks use to refer to this feature, but
there have been other suggestions, including `poll_proceed` and `poll_bg`.
We'll probably want to bikeshed this a bit.

### Should we rename `AsyncIterator` to `Stream`?

RFC 2996 includes [a discussion of that question][rename], and this RFC should
avoid duplicating it. If the consensus on RFC 2996 changes, we can do a
find/replace here.

[rename]: https://rust-lang.github.io/rfcs/2996-async-iterator.html#naming

### How should the `Stream` ecosystem migrate?

It would be nice if the `futures-core` crate could add a blanket `impl<Iter:
AsyncIterator> Stream for Iter`. Unfortunately, that would overlap with
`Stream`'s existing blanket impls.

We might need to ask `Stream` implementations to also, separately, implement
`AsyncIterator`. That's a lot of boilerplate, but it should be straightforward
for non-concurrent streams, which only need to forward `poll_progress` to their
children. Implementing `AsyncIterator` will be necessary for compatibility with
`for await`. Consumers with `Stream` bounds will be in a tricky position,
because they can't migrate to `AsyncIterator` until all of their dependencies
have added impls, but also `async gen fn` iterators will not implement
`Stream`.

It's an open question whether `Stream` should add a `poll_progress` method with
a no-op default implementation. That could help some of the ecosystem get the
benefits of `poll_progress` before migrating. On the other hand, since `Stream`
callers frequently use the `next` method, the benefit might be small, and it
could be more confusing than helpful. Feedback needed from the `futures`
maintainers.

### Should `async gen fn` implementations of `poll_progress` interact with `progress_pending` flags?

As per the reference-level explanation section, `for await` loops will track
"progress pending" flags to reduce unnecessary calls to `poll_progress` once no
more progress can be made. In an `async fn` the generated `poll` method will
handle these flags (if the function has any `for await` loops), and in an
`async gen fn` the generated `poll_next` method will do the same. The open
question is, should the generated `poll_progress` method also look at them
and/or update them? If there's a `yield` in the body of a `for await` loop that
also contains other `.await` points, the flag will be there in any case. The
reason not to is that we'd rather not allocate these flags at every level of a
"chain" of `async gen fn`s that call each other; better to track it "once at
the top" in the loop where that iterator chain is consumed. It could make sense
to check and update the flag if it's there, but not to allocate it if the loop
body contains only `yield` points and no `.await` points?

In any case, `AsyncIterator` implementations should be able to tolerate either
answer to this question, so we could also leave this implementation detail
unspecified.

## Future possibilities
[future-possibilities]: #future-possibilities

### Clarifying the `Future` contract

The second deadlock in the "What about `.next()`?" section isn't really
specific to `AsyncIterator`. We can reproduce it using only `Future`
([playground link][future_deadlock]):

[future_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3Apin%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%2C+timeout%7D%3B%0A%0A%2F%2F+%60do_work%60+takes+a+private+lock%2C+sleeps+briefly%2C+and+releases+it.%0A%2F%2F+A+deadlock+here+shouldn%27t+be+possible.%0Aasync+fn+do_work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+my_future+%3D+pin%21%28do_work%28%29%29%3B%0A++++_+%3D+timeout%28Duration%3A%3Afrom_millis%281%29%2C+my_future%29.await%3B%0A++++println%21%28%22We+make+it+here...%22%29%3B%0A++++do_work%28%29.await%3B%0A++++println%21%28%22...but+not+here%21%22%29%3B%0A%7D>

```rs
let my_future = pin!(do_work());
_ = tokio::time::timeout(Duration::from_millis(1), my_future).await;
do_work().await; // Deadlock!
```

Who's "at fault" there? This example doesn't generate any warnings or fail any
lints today. Maybe it should. Above we wrote:

> To satisfy the proposed contract, whatever's driving an `AsyncIterator`
> generally needs to _own_ it.

The relevant part of the contract there is "poll again when you get a wakeup".
That's just as important for `Future` as it is for `AsyncIterator`. Should we
deprecate helpers that can't live up to that responsibility? Does that include
the blanket `Future` impls on [`&mut F`][future_blanket_mut] and [`Pin<&mut
F>`][future_blanket_pin]? They can't be removed at this point, but we could
warn or lint on code that uses them. We could also warn whenever an idle
`Future` lives across a suspension point. That might not cover e.g. `Vec<impl
Future>`, but most `Future` containers are themselves futures or async
iterators, and the corner cases might not matter much in practice.

[future_blanket_mut]: https://doc.rust-lang.org/std/future/trait.Future.html#impl-Future-for-%26mut+F
[future_blanket_pin]: https://doc.rust-lang.org/std/future/trait.Future.html#impl-Future-for-Pin%3CP%3E

A warning like that could've caught ["Futurelock"][futurelock] before it
happened. On the other hand, there are [many `select!` loops in the
wild][mini_redis] that would trigger the same warning, where there's no
widely-used _owning_ pattern that can easily replace them today. Improving this
situation might need to go hand-in-hand with [new
macros](https://github.com/oconnor663/join_me_maybe) or possibly new syntax.
Speaking of which...

### Concurrency syntax

Today we can only introduce concurrency into async iteration with helpers like
`Merge` or `Buffer1`, which are written "by hand" using the `AsyncIterator`
API. We can't write a concurrent program using just `for await` and `async gen
fn` by themselves. But it's interesting to consider how that could change.

An `async fn` (not a generator) can get concurrency in its body using a `join!`
macro, which is "almost like syntax". For example:

```rs
async fn foo() {
    join!(
        async {
            do_stuff().await;
        },
        async {
            do_other_stuff().await;
        },
    );
}
```

This works today, though you'll notice some rough edges if you try to `return`
or `?` in either of those blocks. But the `async gen fn` equivalent doesn't
work at all:

```rs
async gen fn foo() {
    join!(
        async {
            yield; // error
        },
        ...
    );
}
```

What if there was some hypothetical built-in syntax for writing the `async fn`
above? Maybe it could support error handling and other short-circuiting
operations (`break`, `continue`) more gracefully.

```rs
async fn bar() -> anyhow::Result<()> {
    await all {
        do_stuff().await?;
    } and {
        do_other_stuff().await?;
    }
    Ok(())
}
```

> Tangent #1: It could be sound to allow conflicting mutable borrows on both
> sides here, as long as they don't cross a suspension point. That would be a
> second example of control flow that only async code can express, along with
> cancellation.

> Tangent #2: The syntax bikeshedding for a feature like this would be intense,
> of course, but an interesting feature of `await all`...`and` is that it
> suggests a counterpart, `await any`...`or`. The former could have branches of
> different types, and it would evaluate to a tuple, while the latter would
> have branches of the same type, and it would evaluate to a single value. This
> would be "the built-in version of `select!`". It could also allow conflicting
> _moves_ after the final suspension point in each branch, since only one
> branch ever makes it that far.

What if that hypothetical syntax also supported _yielding_? What might _this_
do?

```rs
async gen fn bar() -> u32 {
    await all {
        yield do_stuff().await;
    } and {
        yield do_other_stuff().await;
    }
}
```

Could you implement `merge` like _this_?

```rs
async gen fn merge<Other>(self, other: Other) -> Self::Item
where
    Other: IntoAsyncIterator<Item = Self::Item>,
{
    await all {
        for await item in self {
            yield item;
        }
    } and {
        for await item in other {
            yield item;
        }
    }
}
```

### Generalized coroutines

We could imagine a more general "coroutine" version of `Iterator` and
`AsyncIterator`, which (as in e.g. Python) takes inputs in addition to yielding
items, and which also has a final return value. There are a couple of unused
value spots in the `for` and `for await` syntax:

```rs
for await item in iter {
    do_stuff(item).await;
    // The value of the body is always `()`.
} // The value of the loop itself is always `()`.
```

Similarly, there are a couple of unused value spots in the likely `gen` and
`async gen` syntax:

```rs
async gen fn foo() -> u32 {
    yield 1; // The value of the `yield` expression is always `()`.
    // The return value of the function is always `()`.
}
```

These spots are a surprisingly good fit for coroutine inputs and return values.
In the `gen fn` / `async gen fn` syntax, an input item would become the value
of the currently suspended `yield` expression, and the return value of the body
would be the final value of the coroutine. In the `for` / `for await` syntax,
the final value could become the value of the loop itself, and the input items
could come from the value of the body. (This would suggest that `break` would
need a value of the same type as the final value, and `continue` would need a
value of the same type as the inputs.)

The async coroutine equivalent of `PollNext` might have a second type parameter
for the final value:

```rs
enum CoroutinePollNext<Item, Return> {
    Item(Item),
    Pending,
    Done(Return),
}
```

An async coroutine trait might look very similar to the version of
`AsyncIterator` proposed here. It would need three type parameters and a third
method called something like `send`. Perhaps the contract would be that you
must call `send` once at some point after each `Item`, before calling
`poll_next` again. We might also define a "`send` rule" similar to the
"`PollNext::Pending` rule": once you `send` an input, the iterator is
"running", and you must call `poll_next` promptly. It wouldn't be as easy for a
`for await coroutine` loop or whatever it might be called to enable concurrency
between the coroutine and the body -- a wrapper type like `Buffer1` above would
need a way to come up with input values, which might be possible in some cases
-- but [more complicated macros][drive] would be able to do fun things.

[barbara]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html
[futurelock]: https://rfd.shared.oxide.computer/rfd/0609
[`FuturesUnordered`]: https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html
[`Merge`]: https://docs.rs/tokio-stream/latest/tokio_stream/trait.StreamExt.html#method.merge
[`StreamMap`]: https://docs.rs/tokio-stream/latest/tokio_stream/struct.StreamMap.html
[drive]: https://github.com/oconnor663/drive_async_iterator
[mini_redis]: https://smallcultfollowing.com/babysteps/blog/2022/06/13/async-cancellation-a-case-study-of-pub-sub-in-mini-redis/
