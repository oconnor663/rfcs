- Feature Name: `poll_progress`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Add a required `poll_progress` method to the [`AsyncIterator`] trait, and make
`for await` loops call this method whenever an `.await` in their loop body is
`Pending`. Expand the documented `AsyncIterator` contract to require async
iterator combinators to do the same.

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
driving `my_iter` entirely. That applies necessary "backpressure" to the
iterator, so it's mostly by design. But it can be a problem if `my_iter` wraps
multiple concurrent futures or other iterators internally, because suspending
some of those at arbitrary `.await` points isn't generally correct. Here's an
example where that causes a deadlock that looks like it should be impossible in
a straight-line reading of the code:

```rs
use futures::stream::FuturesUnordered;
use tokio::sync::Mutex;
use tokio::time::{Duration, sleep};

async fn foo() {
    static LOCK: Mutex<()> = Mutex::const_new(());
    let _guard = LOCK.lock().await;
    sleep(Duration::from_millis(10)).await;
}

#[tokio::main]
async fn main() {
    let futures = FuturesUnordered::new();
    futures.push(foo());
    futures.push(foo());
    for await _ in futures {
        foo().await; // Deadlock!
    }
}
```

[`FuturesUnordered`] doesn't implement `AsyncIterator` today, so this example
doesn't compile as written, but you can run it on stable by [replacing `for
await` with `for_each`][for_each]. When control enters the loop body, one of
the `foo` futures remains in the `FuturesUnordered` buffer, suspended at the
point where it's tried to acquire `LOCK` and taken a spot in its waiters queue.
The call to `foo` in the body tries to acquire `LOCK` again, but the waiter at
the front of the queue never again makes progress, so it's deadlocked.

[`FuturesUnordered`]: https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html
[for_each]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=ceccac95773cbba8aeddf162b5793a3f

To avoid these sorts of deadlocks, and other hard-to-diagnose hangs and
latencies, concurrent async iterators like `FuturesUnordered` need to
continuously drive the futures they contain. That means that `for await` and
other combinators need a way to let an iterator make progress, even when
they're not ready to accept another item.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### How `FuturesUnordered` might document its behavior

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

- `Poll::Pending` means... \[no change\]

- `Poll::Ready(Some(val))` means that the async iterator has successfully
  produced a value, `val`, and may produce further values on subsequent
  `poll_next` calls. In this case **the caller must arrange to call either
  `poll_next` or `poll_progress` again promptly.** If the caller is another
  `poll_next` method, it can trust its own caller to arrange this.

- `Poll::Ready(None)` means that the async iterator has terminated, and neither
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
returned `Ready(None)`. Calling `poll_progress` in either of those cases may
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

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<Option<Self::Item>> {
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

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context) -> Poll<Option<Self::Item>> {
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
adding more work to it in the loop body. There are several problems with
switching to `for await _ in futures` here, but the biggest one is that it
would take ownership of `futures`, and `futures.push(job)` wouldn't compile
after that. Some callers might opt to replace this whole exercise with channels
and task spawning, or with the [`moro`] crate, which allows for local
borrowing. Alternatively, callers who'd rather keep `FuturesUnordered` (or
[`StreamMap`], or some embedded/`no_std` cousin of those) could consider a loop
macro that provides temporary mutable access to the iterator within the loop
body. [`join_me_maybe`] has [experimental support for this][experiment] today,
but I'm not aware of anything like it in common use.

[`moro`]: https://github.com/nikomatsakis/moro
[`StreamMap`]: https://docs.rs/tokio-stream/latest/tokio_stream/struct.StreamMap.html
[`join_me_maybe`]: https://docs.rs/join_me_maybe/latest/join_me_maybe/
[experiment]: https://docs.rs/join_me_maybe/latest/join_me_maybe/#mutable-access-to-futures-and-streams

There's a lot of code in the wild that looks like this example, and changing or
replacing these patterns will be difficult. On the other hand, these examples
are often subtly broken today, not only because rapid-fire cancellation is
tricky, but also more importantly because `select!`-`.next()`-in-a-loop stops
driving its iterator inside the `select!` bodies. (The borrow checker can see
that it stops; that's why we're allowed to re-borrow `futures` in the loop
above.) That's exactly the sort of deadlock-prone behavior this RFC is trying
to fix! The motivating examples in the first sections are about iterators with
internal concurrency, but here we have an iterator being driven concurrently
with something else, and if either side stops making progress we get all the
same problems. We can deadlock this with a `Mutex` that's touched by both the
loop and the work it's driving ([playground link][loop_select_deadlock]).
["Futurelock"](https://rfd.shared.oxide.computer/rfd/0609) is a good case study
in how mind-numbingly hard these `select!` loops are to debug when they
deadlock in the wild.

[loop_select_deadlock]: <https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+futures%3A%3AStreamExt%3B%0Ause+futures%3A%3Astream%3A%3AFuturesUnordered%3B%0Ause+tokio%3A%3Aselect%3B%0Ause+tokio%3A%3Async%3A%3AMutex%3B%0Ause+tokio%3A%3Atime%3A%3A%7BDuration%2C+sleep%7D%3B%0A%0Aasync+fn+foo%28%29+%7B%0A++++static+LOCK%3A+Mutex%3C%28%29%3E+%3D+Mutex%3A%3Aconst_new%28%28%29%29%3B%0A++++let+_guard+%3D+LOCK.lock%28%29.await%3B%0A++++sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%7D%0A%0Aasync+fn+more_work%28%29+-%3E+impl+Future%3COutput+%3D+%28%29%3E+%7B%0A++++foo%28%29%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+futures+%3D+FuturesUnordered%3A%3Anew%28%29%3B%0A++++loop+%7B%0A++++++++select%21+%7B%0A++++++++++++Some%28_%29+%3D+futures.next%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22finished+a+job%22%29%3B%0A++++++++++++%7D%0A++++++++++++job+%3D+more_work%28%29+%3D%3E+%7B%0A++++++++++++++++println%21%28%22got+a+job%22%29%3B%0A++++++++++++++++futures.push%28job%29%3B%0A++++++++++++++++foo%28%29.await%3B+%2F%2F+Deadlock%21+%28after+a+few+iterations%29%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D>

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

## Prior art
[prior-art]: #prior-art

Various approaches to this feature, and the hangs and deadlocks that motivate
it, have been discussed for many years. Some points of reference:

- ["Barbara battles buffered streams"](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html)
- ["`for await` and the battle of buffered streams"](https://tmandry.gitlab.io/blog/posts/for-await-buffered-streams/)
- https://without.boats/blog/poll-progress
- ["Futurelock"](https://rfd.shared.oxide.computer/rfd/0609)
- ["Never snooze a future"](https://jacko.io/snooze.html)

## Unresolved questions
[unresolved-questions]: #unresolved-questions

### Should `poll_next` get its own return type enum?

The `poll_next` method currently returns `Poll<Option<Item>>`, and most of this
RFC keeps that unchanged. That return type captures the three possible return
states (item, pending, and done), but it doesn't represent anything about the
`poll_next`/`poll_progress` contract. Compare that to the `Future::poll`
method. Rust could've defined `poll` to return `Option<Output>`, but the
`Future` contract is important/subtle enough that it's worth adding a separate
type to represent it.

The same could be said of `poll_next`. The "yielded an item" state comes with
both _permission_ to call `poll_progress` and a _requirement_ to call either
`poll_next` or `poll_progress`. This is important/subtle enough that I do think
we should change `poll_next` to return a new enum:

```rs
enum PollNext<Item> {
    Item(Item),
    Pending,
    Done,
}
```

This RFC is disruptive enough that I wanted to start with the minimal possible
rendition of it, but if there does turn out to be consensus in favor of a
`PollNext` enum over `Poll<Option<_>>`, it would make sense to make that change
at the same time.

### What other names should we consider for `poll_progress`?

In my experience `poll_progress` is the most common name folks use to refer to
this feature, but I've also seen `poll_proceed` and `poll_bg`. We'll probably
want to bikeshed this a bit.

### Should we rename `AsyncIterator` to `Stream`?

RFC 2996 includes a [substantial discussion] of that question, and this RFC
should avoid duplicating it. If the consensus on RFC 2996 changes, we can do a
find/replace here.

[substantial discussion]: https://github.com/rust-lang/rfcs/blob/master/text/2996-async-iterator.md#naming

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
