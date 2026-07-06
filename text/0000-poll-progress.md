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

In this interface, all the work an iterator does is driven through `poll_next`.
Consider how that works in the context of a `for await` loop:

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
`for_each`][for_each]. When control enters the loop body, the right side of the
`Merge` still holds a `do_work` future, which is suspended at the point where it's
tried to acquire `LOCK` and taken a spot in its waiters queue. The call to
`do_work` in the body tries to acquire `LOCK` again, but the waiter at the front of
the queue never again makes progress, so it's deadlocked.

[for_each]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3Astream%3A%3A%7Bself%2C+StreamExt+as+_%7D%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0Ause+tokio_stream%3A%3AStreamExt+as+_%3B%0A%0A%2F%2F+%60do_work%60+takes+a+private+lock%2C+sleeps+briefly%2C+and+releases+it.%0A%2F%2F+A+deadlock+here+shouldn%27t+be+possible.%0Aasync+fn+do_work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++stream%3A%3Aonce%28do_work%28%29%29%0A++++++++.merge%28stream%3A%3Aonce%28do_work%28%29%29%29%0A++++++++.for_each%28%7C_%7C+async+%7B%0A++++++++++++println%21%28%22We+make+it+here...%22%29%3B%0A++++++++++++do_work%28%29.await%3B%0A++++++++++++println%21%28%22...but+not+here%21%22%29%3B%0A++++++++%7D%29%0A++++++++.await%3B%0A%7D>

Hangs and deadlocks like this are [more frequently discussed][barbara] in the
context of "fancy" async iterators like [`FuturesUnordered`] or [`buffered`]
streams, but this example uses `merge` to emphasize that "fundamental"
combinators have the same problem.

To avoid these sorts of deadlocks, and other hard-to-diagnose hangs and
latencies, concurrent async iterators like these need to continuously drive the
futures they contain. That means that `for await` and other combinators need a
way to let an iterator make progress, even when they're not ready to accept
another item. `poll_progress` is how they will do this. It will come with new
contract requirements, and to emphasize those we'll change the return type of
`poll_next` from `Poll<Option<_>>` to a dedicated `PollNext` enum.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### How `FuturesUnordered` might document its behavior

[`FuturesUnordered`] is an async iterator over the results of the futures it
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

### How [`tokio_stream::StreamExt::merge`] might document its behavior

[`tokio_stream::StreamExt::merge`]: https://docs.rs/tokio-stream/latest/tokio_stream/trait.StreamExt.html#method.merge

Combine two streams into one by interleaving the output of both as it is
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

### `AsyncIterator` API docs

#### `poll_next`

- `PollNext::Item(item)` means that the async iterator has successfully
  produced an item, `item`, and may produce further items on subsequent
  `poll_next` calls. In this case **the caller must arrange to call either
  `poll_next` or `poll_progress` again promptly.** If the caller is another
  `poll_next` method, it can trust its own caller to arrange this.

- `PollNext::Pending` means...

- `PollNext::Done` means that the async iterator has terminated, and neither
  `poll_next` nor `poll_progress` should be invoked again.

#### `poll_progress`

Allows this async iterator to make progress internally, even though the next
value is not yet needed.

##### Return value

- `Poll::Pending` means that more progress might be possible in the future.
  Implementations will ensure that the current task will be notified when more
  progress can be made.

- `Poll::Ready(())` means that the async iterator has made as much internal
  progress as it can, and `poll_progress` does not need to be invoked again
  until the next time `poll_next` returns an item. Continuing to call
  `poll_progress` is allowed but generally has no effect.

##### Panics

`poll_progress` **must not be called** after the most recent call to
`poll_next` has returned `Pending` or after any call to `poll_next` has
returned `Done`. Calling `poll_progress` in either of those cases may
panic, block forever, or cause other kinds of problems; the `AsyncIterator`
trait places no requirements on the effects of such a call. However, as the
`poll_progress` method is not marked `unsafe`, Rust's usual rules apply: calls
must never cause undefined behavior (memory corruption, incorrect use of
`unsafe` functions, or the like), regardless of the async iterator's state.

### Examples that might appear in "How to implement `AsyncIterator`"

`Once` is an adapter that converts a `Future` into an `AsyncIterator` of one
element. Its implementation might look like this:

```rs
pub struct Once<Fut> {
    future: Option<Pin<Box<Fut>>>,
}

impl<Fut: Future> AsyncIterator for Once<Fut> {
    type Item = Fut::Output;

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context) -> PollNext<Self::Item> {
        if let Some(fut) = &mut self.future {
            if let Poll::Ready(item) = fut.as_mut().poll(cx) {
                self.future = None;
                PollNext::Item(item)
            } else {
                PollNext::Pending
            }
        } else {
            PollNext::Done
        }
    }

    fn poll_progress(self: Pin<&mut Self>, _: &mut Context) -> Poll<()> {
        assert!(self.future.is_none(), "only called after yielding an item");
        Poll::Ready(())
    }
}
```

Note the `assert!` in `poll_progress`. Because `poll_progress` can only be
called after `poll_next` yields an item, we know that `self.future` should
already have been dropped, and `poll_progress` never needs to poll it. For the
same reason, the `Once` struct doesn't need space to buffer an item.

`MergeAll` is similar to `Once`, but it drives many futures instead of just
one, and it iterates over all their outputs as they come. In other words, it's
the iterator version of the future combinator [`JoinAll`]. Its implementation
might look like this:

[`JoinAll`]: https://docs.rs/futures/latest/futures/future/fn.join_all.html

```rs
pub struct MergeAll<Fut: Future> {
    futures: Vec<Pin<Box<Fut>>>,
    items: VecDeque<Fut::Output>,
}

impl<Fut: Future> AsyncIterator for MergeAll<Fut> {
    type Item = Fut::Output;

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context) -> PollNext<Self::Item> {
        // If we've buffered any items, yield one of those.
        if let Some(item) = self.items.pop_front() {
            return PollNext::Item(item);
        }
        // Poll the remaining futures and yield the first item we get.
        for i in 0..self.futures.len() {
            if let Poll::Ready(item) = self.futures[i].as_mut().poll(cx) {
                self.futures.swap_remove(i);
                return PollNext::Item(item);
            }
        }
        if self.futures.is_empty() {
            PollNext::Done
        } else {
            PollNext::Pending
        }
    }

    fn poll_progress(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
        let MergeAll { futures, items } = &mut *self;
        // Poll the remaining futures and buffer all the items we get.
        futures.retain_mut(|fut| {
            fut.as_mut()
                .poll(cx)
                .map(|item| items.push_back(item))
                .is_pending()
        });
        if self.futures.is_empty() {
            Poll::Ready(())
        } else {
            Poll::Pending
        }
    }
}
```

Note that `poll_next` yields the first output it sees, without necessarily
polling all the remaining futures. The caller is required to call either
`poll_next` or `poll_progress` again promptly, so one way or another all the
futures will get polled.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

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
let stream = pin!(...);
while let Some(item) = stream.next().await { ... }
```

That can replaced with a `for await` loop:

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
`poll_progress`-compatible way using a macro. Here is a [working
proof-of-concept](https://github.com/oconnor663/drive_async_iterator). The key
is that the macro takes ownership of the async iterator and provides a handle
with a `next` method on it, which lets the macro call `poll_progress`
concurrently with its body as needed. Apart from easing migration, a macro like
this also fixes [potential deadlocks in the loop above][loop_select_deadlock].

[loop_select_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3AStreamExt%3B%0Ause+futures%3A%3Astream%3A%3AFuturesUnordered%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+work%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+more_work%28%29+-%3E+impl+Future%3COutput+%3D+%28%29%3E+%7B%0A++++work%28%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+futures+%3D+FuturesUnordered%3A%3Anew%28%29%3B%0A++++loop+%7B%0A++++++++select%21+%7B%0A++++++++++++Some%28_%29+%3D+futures.next%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22finished+a+job%22%29%3B%0A++++++++++++%7D%0A++++++++++++job+%3D+more_work%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22got+a+job%22%29%3B%0A++++++++++++++++futures.push%28job%29%3B%0A++++++++++++++++work%28%29.await%3B+%2F%2F+Deadlock%21+%28after+a+few+iterations%29%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D>

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Is pausing futures at `.await` points so terrible? Could we instead agree to allow it?

The fundamental assumption of this RFC is that we need to guarantee continuous
control flow through async code. Async control flow can include cancellation
via `Drop`, but indefinite pauses shouldn't be possible.

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
["Futurelock"][futurelock] for example, it was a sempahore buried inside
Tokio's channel implementation. If async Rust code has to tolerate indefinite
pauses, then we need to be defensive about futurelocks everywhere, and writing
async applications starts to feel like writing Unix signal handlers.

Could there be a third way? Maybe there could be some sort of `Drop`-like hook
that tells futures and async iterators when they're about to be paused or
resumed. I haven't explored this in any detail, but my first question would be:
"If I'm holding a lock, what am I supposed to do in that hook? Release the
lock?" The point of locking is that it lets us group operations together
atomically. If a lock can be _stolen_ from us in the middle of our critical
section, then it isn't really a lock at all. Realistically, this third way
would have to look more like lock-free programming. Lock-free code is great,
but it's not the _only_ sort of code that async Rust aims to support.

### Why not allow `poll_progress` at any time?

The third part of the three-part `AsyncIterator` contract this RFC is proposing
is that `poll_progress` shouldn't be called when `poll_next` is pending. That
seems kind of arbitrary. What's the point of that rule?

Consider the `poll_progress` implementation of a combinator like [`Merge`].
_Without_ the third rule, it could be as short as this:

```rs
fn poll_progress(self: Pin<&mut Self>, cx: &mut Context) -> Poll<()> {
    let this = self.project();
    // XXX: This version doesn't keep track of whether `poll_next` is pending.
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
next `yield`. However, `poll_progress` should't _always_ drive control to the
next `yield`. If it did, then `for await` loops would always drive their
iterators concurrently with their loop bodies, which isn't how they're supposed
to work. Instead, `foo` would need to track whether `poll_next` has been called
_at least once_ before allowing control to enter the body, and before allowing
control to proceed after a `yield`. That's certainly doable, but consider
applying the same logic to existing combinators like [`Map`] and [`Then`].
Those would need to add a state flag that they don't have today (say
`next_item_wanted`), and they'd also need a rarely-used buffer slot for an
item.

[`Map`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.map
[`Then`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.then

In practice, the bookkeeping for "if `poll_next` has been called at least once,
`poll_progress` advances control to the next yield and buffers an item" would
look awfully similar to the bookkeeping for "don't call `poll_progress` when
`poll_next` is pending", except moved down a level from the caller to the
callee. Rather than adding complexity and buffer slots only to concurrent async
iterators like `Merge` and [`StreamMap`], we'd add complexity and buffer slots
to _every_ async iterator. Not a good trade.

Also, if the `AsyncIterator` contract is going to be somewhat subtle and
error-prone either way, a major upside of the rule as proposed is that it's
clear when it's been violated, and we can enforce it with `debug_assert!`s.
(`async gen fn` should handle violations by panicking, just like `async fn`
futures panic today if you poll them again after they've returned.) If any
interleaving of `poll_next` and `poll_progress` was valid, it would be a lot
harder to detect `AsyncIterator` contract violations programmatically.

### Does `poll_next` need a new return type enum?

Today's `Poll<Option<_>>` return type captures the three possible return states
(item, pending, and done), but it doesn't represent anything about the
`poll_next`/`poll_progress` contract. Compare that to the `Future::poll`
method. Rust could've defined `poll` to return `Option<Output>`, but the
`Future` contract is important and subtle enough that it was worth adding a
separate type to represent it.

I think the same is true of `poll_next` in the new contract. The "yielded an
item" state comes with both _permission_ to call `poll_progress` and a
_requirement_ to call either `poll_next` or `poll_progress` again. This is
important and subtle enough it's worth a dedicated return type.

## Prior art
[prior-art]: #prior-art

Various approaches to this feature, and the hangs and deadlocks that motivate
it, have been discussed for many years. Some points of reference:

- ["Barbara battles buffered streams"][barbara]
- ["`for await` and the battle of buffered streams"](https://tmandry.gitlab.io/blog/posts/for-await-buffered-streams/)
- https://without.boats/blog/poll-progress
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
combinators like `Merge`, which are written "by hand" using the `AsyncIterator`
API. We can't write a concurrent program using `for await` and `async gen fn`
by themselves. But it's interesting to consider how that could change.

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

What it there was some hypothetical built-in syntax for writing the `async fn`
above? Maybe it could support error handling and other short-circuiting
operations (`break`, `continue`) more gracefully:

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

What if that hypothetical syntax supported _yielding_? What might _this_ do?

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

...

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

[barbara]: https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html
[futurelock]: https://rfd.shared.oxide.computer/rfd/0609
[`FuturesUnordered`]: https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html
[`buffered`]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffered
[`Merge`]: https://docs.rs/tokio-stream/latest/tokio_stream/trait.StreamExt.html#method.merge
[`StreamMap`]: https://docs.rs/tokio-stream/latest/tokio_stream/struct.StreamMap.html
