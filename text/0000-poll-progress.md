- Feature Name: `poll_progress`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Add a required `poll_progress` method to the [`AsyncIterator`] trait, and make
`for await` loops call this method whenever an `.await` in their loop body is
`Pending`. Expand the documented `AsyncIterator` contract to require
combinators and consumers to do the same. To emphasize the contract
requirements of `poll_next`, define a new `PollNext` enum for it to return,
replacing `Poll<Option<_>>`.

[`AsyncIterator`]: https://doc.rust-lang.org/std/async_iter/trait.AsyncIterator.html

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
for await item in my_iter {
    do_work(item).await;
}
```

When control is at the top, the `for await` loop calls `my_iter.poll_next`
until it either yields an item with `Ready(Some(_))` or indicates that it's
done with `Ready(None)`. Once control moves into `do_work`, the loop stops
driving `my_iter` entirely. That applies necessary "backpressure" to async
iterators, and it's mostly by design. But it can be a problem if `my_iter`
wraps multiple concurrent futures or other iterators internally, because
suspending some of those at arbitrary `.await` points isn't generally correct.
Here's an example where this causes a deadlock that looks like it should be
impossible in a straight-line reading of the code:

```rs
use futures::stream::once;
use tokio::sync::Mutex;
use tokio::time::{Duration, sleep};
use tokio_stream::StreamExt;

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
compile as written, but you can run it on stable by [replacing `for await` with
`for_each`][for_each_playground]. When control enters the loop body, the right
side of the `Merge` still holds a `do_work` future, which is suspended at the
point where it has tried to acquire `LOCK` and taken a spot in its waiters
queue. The call to `do_work` in the body tries to acquire `LOCK` again, but the
waiter at the front of the queue never again makes progress, so it's
deadlocked.

[for_each_playground]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3A%7Bself%2C+StreamExt+as+_%7D%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0Ause+tokio_stream%3A%3AStreamExt+as+_%3B%0A%0A%2F%2F+%60do_work%60+takes+a+private+lock%2C+sleeps+briefly%2C+and+releases+it.%0A%2F%2F+A+deadlock+here+shouldn%27t+be+possible.%0Aasync+fn+do_work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++stream%3A%3Aonce%28do_work%28%29%29%0A++++++++.merge%28stream%3A%3Aonce%28do_work%28%29%29%29%0A++++++++.for_each%28%7C_%7C+async+%7B%0A++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++do_work%28%29.await%3B%0A++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++%7D%29%0A++++++++.await%3B%0A%7D>

We usually talk about hangs and deadlocks like these [in the context of "fancy"
async iterators like `FuturesUnordered` or `buffered` streams][barbara], but
the example above uses `merge` to emphasize that simpler combinators have the
same problem.

To avoid these sorts of deadlocks, and other hard-to-diagnose hangs and
latencies, concurrent async iterators like these need to continuously drive the
futures they contain. That means that `for await` and other consumers and
combinators need a way to let an iterator make progress, even when they're not
ready to accept another item. The new `poll_progress` method is how they can do
this. It comes with new contract requirements, and to emphasize those we'll
change the return type of `poll_next` from `Poll<Option<_>>` to a dedicated
`PollNext` enum.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `AsyncIterator` intro

As its name suggests, `AsyncIterator` is the async version of [`Iterator`].
Just like `Iterator` is the trait at the heart of `for` loops and other forms
of iteration, `AsyncIterator` is the trait at the heart of `for await` loops
and other forms of asynchronous iteration.

[`Iterator`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html

Async code can and often does use regular iterators and `for` loops too, as
long as their next item is always immediately available, like loops over a
range or a collection. But async functions aren't allowed to do blocking IO,
and they can't use regular iterators that sometimes block, for example
[`std::io::Lines`] or [`std::sync::mpsc::Iter`]. The main purpose of
`AsyncIterator` is to support iterators like these that sometimes need to wait
on input, in a way that's compatible with async code.

[`std::io::Lines`]: https://doc.rust-lang.org/std/io/struct.Lines.html
[`std::sync::mpsc::Iter`]: https://doc.rust-lang.org/std/sync/mpsc/struct.Iter.html

In addition, async iterators have a superpower that regular iterators generally
do not. They can keep working "in the background" while the caller is
processing an item. For example:

```rs
for await jpeg in fetch_images() {
    save_image(jpeg).await;
}
```

Depending on how it's implemented, `fetch_images` could start downloading the
next `jpeg` concurrently while control is inside `save_image`. A regular
iterator would need to use threads to do that, which complicates borrowing and
short-circuiting and usually requires heap allocation. But async iterators can
do concurrent background work without threads or allocations, and with full
support for local borrowing and intuitive behavior for `break` and `return`
(cancelling the background work).

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
currently ready, and a wakeup has been registered. When a `for await` loop
wants to get the next item from an async iterator, it calls `poll_next`.

`poll_progress` is unique to `AsyncIterator`, and it's what allows the iterator
to do background work when the caller isn't ready for another item. It returns
`Poll::Pending` (not `PollNext::Pending`) when more progress might be possible
in the future, and it also registers a wakeup in that case. When no more
internal progress is possible without calling `poll_next` again,
`poll_progress` returns `Ready`. When an `.await` in the body of a `for await`
loop is pending, the loop calls `poll_progress` on its iterator before
reporting pending itself.

[`Poll::Pending`]: https://doc.rust-lang.org/std/task/enum.Poll.html

Like `Future::poll`, the return values of these methods impose certain
requirements on their callers. But while `Future` only has two possible returns
to contend with, `AsyncIterator` has _five_:

| Previous return | You are required to... |
| --- | --- |
| `poll_next` returned `PollNext::Item` | Poll again **promptly**, either `poll_next` if you want another item or `poll_progress` if not. |
| `poll_next` returned `PollNext::Pending` | `poll_next` again after wakeup. `poll_progress` is **not allowed**. |
| `poll_next` returned `PollNext::Done` | Never poll again. Drop promptly. |
| `poll_progress` returned `Poll::Pending` | `poll_next` whenever you want another item, otherwise `poll_progress` again after wakeup. |
| `poll_progress` returned `Poll::Ready` | `poll_next` whenever you want another item, otherwise stop polling. `poll_progress` is allowed but has no further effect. |

An `AsyncIterator` begins life in the first state above, expecting a prompt
call to either `poll_next` or `poll_progress`. Although a `for await` loop will
always start by calling `poll_next`, other consumers may start with
`poll_progress`. Also, cancelling an async iterator by dropping it is allowed
at any time.

The first two requirements in the table above are bolded, because they're the
most surprising. Expanding on those:

- **The `PollNext::Item` rule:** When `poll_next` returns an item, we don't
  expect it to register any wakeups. That's important for performance, because
  there might be many items ready to return, and we don't want to trigger
  redundant wakeups for each of them or cause an extra round trip through the
  executor. On the flip side, if we don't want to call `poll_next` any more, we
  need to call `poll_progress` to give the iterator a chance to finish its own
  polling responsibilities and register wakeups.
- **The `PollNext::Pending` rule:** When `poll_next` is pending, we have to
  `poll_next` again at the next wakeup; we can't "change our minds" about
  wanting an item and switch to `poll_progress`. (The only valid way to change
  our minds is to cancel the whole iterator by dropping it.) This rule is aimed
  at concurrent combinators that "merge" multiple async iterators together.
  After a child iterator yields an item, such a combinator should keep driving
  its other children with `poll_next` internally until each of them has yielded
  an item. This ensures the smooth flow of control through chains of
  combinators, and it means that non-concurrent combinators don't need to
  allocate buffer space for an item. The rationale section [discusses this rule
  further](#why-not-allow-poll_progress-at-any-time).

Let's look at the implementation of an async iterator combinator, `Buffer1`,
which wraps another `AsyncIterator` and pre-fetches the next item in
`poll_progress`. This surprisingly powerful combinator turns _any_
`AsyncIterator` into a concurrent iterator that does work in the background:

```rs
struct Buffer1<Iter: AsyncIterator> {
    #[pin]  // TODO: a standard way to do structural pinning in examples?
    inner: Option<Iter>,
    item: Option<Iter::Item>,
}

impl<Iter: AsyncIterator> AsyncIterator for Buffer1<Iter> {
    type Item = Iter::Item;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> PollNext<Iter::Item> {
        let mut this = self.project();
        if let Some(item) = this.item.take() {
            // If we have a buffered `item`, return it.
            PollNext::Item(item)
        } else if let Some(inner) = this.inner.as_mut().as_pin_mut() {
            // Otherwise poll the `inner` async iterator.
            inner.poll_next(cx)
        } else {
            // ...unless it was already dropped when we tried to buffer an item. In that case we're done.
            PollNext::Done
        }
    }

    fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
        let mut this = self.project();
        let Some(mut inner) = this.inner.as_mut().as_pin_mut() else {
            // The `inner` iterator is already done.
            return Poll::Ready(());
        };
        if this.item.is_none() {
            // We don't have a buffered `item`. Try to get one.
            match inner.as_mut().poll_next(cx) {
                PollNext::Item(item) => {
                    *this.item = Some(item);
                    inner.poll_progress(cx) // required
                }
                PollNext::Pending => Poll::Pending,
                PollNext::Done => {
                    this.inner.set(None); // required
                    Poll::Ready(())
                }
            }
        } else {
            // We already have a buffered `item`.
            inner.poll_progress(cx)
        }
    }
}
```

Note that when `inner.poll_next` returns `Item`, `Buffer1::poll_next` returns
immediately without doing any further polling. That might look like it violates
the "`PollNext::Item` rule" about polling again promptly, however, the same
rule applies _to the caller_. The caller will poll again, and when they do,
`inner` will get polled again. Similarly, `Buffer1::poll_next` doesn't need to
drop `inner` when it returns `Done`, because the caller will drop the whole
`Buffer1`. However, `Buffer1::poll_progress` doesn't impose the same rule on
its caller, so it must call `inner.poll_progress` in its `Item` branch and drop
`inner` in its `Done` branch.

Note also that `Buffer1::poll_progress` doesn't call `inner.poll_progress`
after `inner.poll_next` returns `Pending`. That's the "`PollNext::Pending`
rule". The next poll will be either `Buffer1::poll_next` if the caller asks for
another item, or else `Buffer1::poll_progress` calling `inner.poll_next` again
after a wakeup.

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

We can't de-sugar this RFC's version of `for await` syntax into a loop (the way
the Reference does for [the `for` keyword][for]), because we'd need to treat
the body as a future (e.g. `await { /* body */ }`) to intercept `Pending` and
call `poll_progress` in the right place, but that's incompatible with `return`,
`?`, `break`, or `continue` in the body.

[for]: https://doc.rust-lang.org/reference/expressions/loop-expr.html#r-expr.loop.for.desugar

Instead we can describe the steps `for await` takes internally, [the same way
`.await` is described][await]:

[await]: https://doc.rust-lang.org/reference/expressions/await-expr.html#r-expr.await.effects

1. Create an async iterator by calling `IntoAsyncIterator::into_async_iter` on
   the iterator expression.
2. Pin the async iterator using `Pin::new_unchecked`.
3. Poll it by calling the `AsyncIterator::poll_next` method and passing it the
   current task context.
4. If the call to `poll_next` returns `PollNext::Pending`, then the surrounding
   async context returns `Poll::Pending` (or in an `async gen fn`,
   `PollNext::Pending`), suspending its state so that, when it's re-polled,
   execution returns to step 3.
5. If the call to `poll_next` returns `PollNext::Done`, then the loop drops the
   async iterator and evaluates to `()`.
6. If the call to `poll_next` returns `PollNext::Item(item)`, then `item` is
   matched against the irrefutable `PATTERN`.
7. Control proceeds through the loop body, with the bindings from `PATTERN` in
   scope. If any `.await` expression or `for await` item (i.e. step 3 above, if
   another `for await` loop is nested within this one) in the body is pending,
   call `poll_progress` on the async iterator before reporting pending from the
   surrounding async context.
8. \[`break`, `continue`, and diverging control flow as usual\]

Note that if `for await` loops are nested, a pending expression in an inner
loop triggers step 7 above for _all_ the containing loops, starting with the
innermost.

### `async gen fn`

An `async gen fn` returns an `AsyncIterator` implementation. If control in the
`async gen fn` is at a `yield` (not an `.await`) in the body of a `for await`
loop, then the `poll_progress` method on the returned `AsyncIterator` calls
`poll_progress` on that loop's iterator. Note that if `for await` loops are
nested, there may be multiple such iterators, and `poll_progress` gets
forwarded to all of them, starting with the innermost. If any of the forwarded
`poll_progress` calls is pending, then `poll_progress` on the returned
`AsyncIterator` also returns `Pending`, otherwise it returns `Ready`.

Note that when control is suspended at a pending `.await` in an `async gen fn`,
`poll_progress` will not be called on the returned `AsyncIterator`, because the
last call to its `poll_next` method did not return `Item`. (Concretely, if its
caller is using `for await`, the caller is currently in step 3 above, not step
7.) In that case, it's the returned `AsyncIterator`'s `poll_next` method that's
responsible for driving progress in any other async iterators it's looping
over, following the rules in the previous section.

For example:

```rs
async gen fn foo() {
    for await _ in bar() {
        for await _ in baz() {
            // While this expression is pending, `foo`'s `poll_next` function calls `poll_progress` on the
            // `bar` and `baz` async iterators before reporting pending itself. The `AsyncIterator` contract
            // guarantees that `foo`'s `poll_progress` function will not be called in this state.
            do_work().await;
            // While control is suspended at this `yield`, `foo`'s `poll_progress` function calls
            // `poll_progress` on the `bar` and `baz` async iterators, and it reports pending if either of
            // those calls is pending.
            yield;
        }
    }
}
```

## Drawbacks
[drawbacks]: #drawbacks

### What about `.next()`?

The main way users interact with `Stream` in the async ecosystem today is the
[`StreamExt::next`] async function/method, which returns a future representing
the next item in the stream. But the API contract proposed in this RFC isn't
generally compatible with `next`, because the `Next` future is short-lived, and
it can't drive the stream after it yields its item. (Contrast this with
consumers like [`for_each`] and [`fold`], which take ownership of their input
and can implement the new contract internally.)

[`StreamExt::next`]: https://docs.rs/futures/latest/futures/prelude/stream/trait.StreamExt.html#method.next
[`for_each`]: https://docs.rs/futures/latest/futures/prelude/stream/trait.StreamExt.html#method.for_each
[`fold`]: https://docs.rs/futures/latest/futures/prelude/stream/trait.StreamExt.html#method.fold

The most common use case for `next` today is a loop like this:

```rs
let mut stream = pin!(...);
while let Some(item) = stream.next().await { ... }
```

That can be replaced with a `for await` loop:

```rs
for await item in stream { ... }
```

The `for await` version is more concise and doesn't require pinning, so it's a
nice improvement when it works. Unfortunately, it doesn't always work. Here's
an example of a `next` caller that can't easily switch to `for await`
([playground link][loop_select]):

[loop_select]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3AStreamExt%3B%0Ause+futures%3A%3Astream%3A%3AFuturesUnordered%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+work%28%29+%7B%0A++++sleep%28Duration%3A%3Afrom_secs%28rand%3A%3Arandom_range%280..5%29%29%29.await%3B%0A%7D%0A%0Aasync+fn+more_work%28%29+-%3E+impl+Future%3COutput+%3D+%28%29%3E+%7B%0A++++sleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A++++work%28%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+futures+%3D+FuturesUnordered%3A%3Anew%28%29%3B%0A++++loop+%7B%0A++++++++select%21+%7B%0A++++++++++++Some%28_%29+%3D+futures.next%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22finished+a+job%22%29%3B%0A++++++++++++%7D%0A++++++++++++job+%3D+more_work%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22got+a+job%22%29%3B%0A++++++++++++++++futures.push%28job%29%3B%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D>

```rs
let mut futures = FuturesUnordered::new();
loop {
    select! {
        Some(_) = futures.next() => {
            println!("finished a job");
        }
        job = more_work() => {
            println!("got a job");
            futures.push(job);
        }
    }
}
```

In this example, the caller is iterating over a `FuturesUnordered` and also
adding more work to it in the loop body. We can't reorganize this around a `for
await`, because that would take ownership of `futures`, and `futures.push(job)`
wouldn't compile. Patterns like this aren't the most common, but sometimes they
sit at the core of larger application loops, and migrating away from them can
be impractical.

In these difficult cases, it's possible to recreate the `next` method in a
`poll_progress`-compatible way using a macro. Here's a [working
proof-of-concept][drive], which takes ownership of an async iterator and
provides a handle with `next` and `with_mut` methods on it. Apart from easing
migration, the macro also fixes [potential deadlocks in the loop
above][loop_select_deadlock].

[loop_select_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3AStreamExt%3B%0Ause+futures%3A%3Astream%3A%3AFuturesUnordered%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+more_work%28%29+-%3E+impl+Future%3COutput+%3D+%28%29%3E+%7B%0A++++work%28%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+futures+%3D+FuturesUnordered%3A%3Anew%28%29%3B%0A++++loop+%7B%0A++++++++select%21+%7B%0A++++++++++++Some%28_%29+%3D+futures.next%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22finished+a+job%22%29%3B%0A++++++++++++%7D%0A++++++++++++job+%3D+more_work%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22got+a+job%22%29%3B%0A++++++++++++++++futures.push%28job%29%3B%0A++++++++++++++++work%28%29.await%3B+%2F%2F+Deadlock%21+%28after+a+few+iterations%29%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D>

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Is pausing futures at `.await` points so terrible? Could we instead agree to allow it?

The fundamental assumption of this RFC is that we need to guarantee smooth
control flow through async code. Concretely, `Future` and `AsyncIterator`
implementations should be able to assume that they'll be polled again promptly
(or dropped) whenever they invoke their `Waker`.

Is that a good assumption? Do we have consensus on that? What are the other
options?

I think the clearest argument for this assumption is an analogy to
multithreading. ["Everybody knows"](https://jacko.io/snooze.html#threads) that
we can't pause or cancel running threads. If a paused or cancelled thread
happens to be holding any locks, we'll probably deadlock ourselves. In the dark
corners of systems programming where we do pause threads, like Unix signal
handlers, we have to take extraordinary care not to touch any locks in that
critical section, which means e.g. no allocating memory and no printing.

Async Rust can handle cancellation better than threads do, because `Drop`
cleans up our lock guards. But _pausing_ a Rust future isn't much different
from pausing a thread. It can only happen at an `.await` point, so at least we
don't have to worry about the `malloc` lock, but for any exclusive resource
that might be held across an `.await`, the story is the same. In
["Futurelock"][futurelock] for example, it was a semaphore buried inside
Tokio's channel implementation. If async Rust code has to tolerate indefinite
pauses, then we need to be defensive about futurelocks everywhere, and writing
async applications will start to feel like writing Unix signal handlers.

Could there be a third way? Maybe there could be some sort of `Drop`-like hook
that tells futures and async iterators when they're about to be paused or
resumed. I haven't explored this in any detail, but my first question would be:
"If I'm holding a lock, what am I supposed to do in that hook? Release the
lock?" The point of locking is that it lets us group operations together
atomically. If a lock can be _stolen_ from us in the middle of our critical
section, then it isn't a lock at all. Realistically, this third way would look
more like lock-free programming. Lock-free code is great, but it's not the
_only_ sort of code that async Rust aims to support.

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

That would be nice and simple, but the downside is that _every other_ async
iterator would need to handle the case where their caller abruptly switches
from `poll_next` to `poll_progress` before they yield an item. For example,
think about this hypothetical `async gen` function:

```rs
async gen fn foo() {
    do_work().await;
    yield;
}
```

If control is in the middle of `do_work`, `foo` can't just leave it stuck
there, or else we'll have the same deadlocks we saw at the top. If `foo` had to
tolerate a switch from `poll_next` to `poll_progress` at any time, then
`poll_progress` would need to keep driving control through the body up to the
next `yield`. However, `poll_progress` shouldn't _always_ drive control to the
next `yield`. If it did, then `for await` loops would always drive their
iterators concurrently with their loop bodies, which isn't how they're supposed
to work by default. Instead, `foo` would need to track whether `poll_next` has
been called _at least once_ before allowing control to enter the body, and
before allowing control to proceed after a `yield`. That's doable, but then
would we want equivalent behavior from combinators like [`Map`] and [`Then`]?
Those would need to add a state flag that they don't have today (say
`next_item_wanted`), and they'd also need a rarely-used buffer slot for an
item.

[`Map`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.map
[`Then`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.then

In practice, the bookkeeping for "if `poll_next` has been called at least once,
`poll_progress` advances control to the next yield and buffers an item" would
look awfully similar to the bookkeeping for "don't call `poll_progress` when
`poll_next` is pending", except moved down a level from the caller to the
callee. Rather than adding complexity and buffer slots to concurrent async
iterators like `Merge` and [`StreamMap`], we'd add complexity and buffer slots
to _every_ async iterator.

With the `PollNext::Pending` rule, we don't get to write the simple version of
`Merge::poll_progress` above, and instead we have to do some tricky state
tracking and conditional buffering there. That's a downside. The upside is that
`Then::poll_progress` gets to look like this:

```rs
fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
    let this = self.project();
    debug_assert!(this.future.is_none());
    this.stream.poll_progress(cx)
}
```

`Then::poll_progress` doesn't need to worry about what to do with `self.future`
or its output, because it can only be called when `self.future` is `None`.
Similarly, it never needs to call `stream.poll_next`, because it will never be
called after `poll_next` returns `Pending`. Simple combinators like `Map` and
`Then` (and `Filter` and `Flatten` and `Skip` and `Take`) are more common than
concurrent ones like `Merge` and `Buffer1`, both in terms of how many
implementations we need to write and also how many instances appear in chains
of combinators. Keeping the simple things simple is a good trade.

Another upside of the `PollNext::Pending` rule compared to the alternative is
that it's clear when it's been violated, and we can write asserts like the one
above. An `async gen fn` should probably panic if we call `poll_progress` while
it's suspended at an `.await`, just like an `async fn` panics today if we poll
it again after it's returned.

### Is it worth having a whole new `PollNext` enum?

Today's `Poll<Option<_>>` return type captures the three possible return states
(item, pending, and done), but it doesn't represent anything about the
`poll_next`/`poll_progress` contract. Compare that to the `Future::poll`
method. Rust could've defined `poll` to return `Option<Output>`, but the
`Future` contract is important and subtle enough that it was worth adding a
separate type to represent it.

I think the same is true of `poll_next` in the new contract. The
"`PollNext::Item` rule" and the "`PollNext::Pending` rule" are important and
subtle enough that it's worth defining a distinct return type to represent
them.

## Prior art
[prior-art]: #prior-art

Various approaches to this feature, and the hangs and deadlocks that motivate
it, have been discussed for many years. Some points of reference:

- ["Footgun lurking in `FuturesUnordered` and other concurrency-enabling streams"](https://github.com/rust-lang/futures-rs/issues/2387)
- ["Barbara battles buffered streams"][barbara]
- ["`for await` and the battle of buffered streams"](https://tmandry.gitlab.io/blog/posts/for-await-buffered-streams/)
- https://without.boats/blog/poll-progress
- ["A discussion prototype for `AsyncIterator`"](https://hackmd.io/EplPcmBCTCSQ9LPOU6IZDw) and related [meeting notes](https://hackmd.io/bYaPiCdqR3WIyjKhFWzdtQ)
- ["Future's liveness problem"](https://skepfyr.me/blog/futures-liveness-problem/)
- ["Futurelock"][futurelock]
- ["Never snooze a future"](https://jacko.io/snooze.html)

## Unresolved questions
[unresolved-questions]: #unresolved-questions

### What other names should we consider for `poll_progress`?

In my experience `poll_progress` is the most common name folks use to refer to
this feature, but I've also seen `poll_proceed` and `poll_bg`. We'll probably
want to bikeshed this a bit.

### Should we rename `AsyncIterator` to `Stream`?

RFC 2996 includes a [substantial discussion] of that question, and this RFC
should avoid duplicating it. If the consensus on RFC 2996 changes, we can do a
find/replace here.

[substantial discussion]: https://github.com/rust-lang/rfcs/blob/master/text/2996-async-iterator.md#naming

### Will anyone ever look at the `poll_progress` return value?

`poll_progress` returns `Poll::Pending` if it's registered a wakeup, and
`Poll::Ready(())` otherwise. We treat it as "fused" in the sense that calling
it again is allowed and _kinda sorta_ guaranteed to return `Ready` again. (We
don't document that guarantee, but if `poll_progress` spontaneously started
returning `Pending` again, that would probably mean it failed to register a
wakeup for itself earlier, and it only got polled again by chance. That's an
implementation bug?)

In theory we could make it a logic error to call `poll_progress` again after it
returns `Ready` -- like it is to call `Future::poll` again after it returns
`Ready`, or `Iterator::next` after it returns `None` -- but in practice it's
hard to imagine an implementation that would benefit from that requirement,
while it would be a bookkeeping burden on many callers. (`Iterator` combinators
that would need extra state to fuse themselves are also quite rare, but there
are a couple standard ones, including `map_while` and `scan`.) It seems simpler
overall to make `poll_progress` idempotent after it returns `Ready`.

But that raises the question: If callers don't need to track the previous
return value, then they probably won't look at it at all. It could save a
couple lines in a lot of `poll_progress` implementations if it just returned
nothing. Would any callers care? Does anyone really need to know whether
`poll_progress` registered a wakeup?

## Future possibilities
[future-possibilities]: #future-possibilities

### Concurrency syntax for the body of an `async gen fn`

Today we can only introduce concurrency into async iteration by using
combinators like `Merge` or `Buffer1`, which are written "by hand" using the
`AsyncIterator` API. We can't write a concurrent program using just `for await`
and `async gen fn` by themselves. But it's interesting to consider how that
could change.

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
operations (`break`, `continue`) more gracefully. (Tangent: Maybe it could even
allow conflicting mutable borrows on both sides, as long as they don't cross an
`.await`!?)

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
impl AsyncIteratorExt {
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
}
```

### Generalized coroutines

We can imagine a more general "coroutine" version of `Iterator` and
`AsyncIterator`, which (as in e.g. Python) takes inputs in addition to yielding
items, and which also has a final return value. Not that we should necessarily
_implement_ such a feature, but it might be useful to keep the idea in mind.

There are a couple of unused value spots in the `for` and `for await` syntax:

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

These spots map surprisingly cleanly to the concept of a "coroutine" in
languages like Python. In the `gen fn` / `async gen fn` syntax, input items
become the value of the currently suspended `yield` expression, and the final
value comes from the return value of the body. In the `for` / `for await`
syntax, the final value could become the value of the loop itself, and the
input items could come from the value of the body. (This would suggest that
`break` would need a value of the same type as the final value, and `continue`
would need a value of the same type as the inputs.)

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
`AsyncIterator` proposed here. It would need a third method called something
like `send`, and perhaps the contract would be that you must call `send` once
at some point after each `Item`, before calling `poll_next` again. A `for await
coroutine` loop or whatever it might be called wouldn't have an obvious way to
enable concurrency between the coroutine and the body (a wrapper type like
`Buffer1` above would need a way to come up with input values, which might be
possible in some cases), but [more complicated macros][drive] might be able to
do fun things.

[barbara]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html
[futurelock]: https://rfd.shared.oxide.computer/rfd/0609
[`FuturesUnordered`]: https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html
[`buffered`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffered
[`Merge`]: https://docs.rs/tokio-stream/latest/tokio_stream/trait.StreamExt.html#method.merge
[`StreamMap`]: https://docs.rs/tokio-stream/latest/tokio_stream/struct.StreamMap.html
[drive]: https://github.com/oconnor663/drive_async_iterator
