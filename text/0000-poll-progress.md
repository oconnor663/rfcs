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
The call to `foo` in the body tries acquire `LOCK` again, but the waiter at the
front of the queue never again makes progress, so it's deadlocked.

[`FuturesUnordered`]: https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html
[for_each]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=ceccac95773cbba8aeddf162b5793a3f

To avoid these sorts of deadlocks, and other hard-to-diagnose hangs and
latencies, concurrent async iterators like `FuturesUnordered` need to
continuously drive the futures they contain. That means that `for await` and
other combinators need a way to let an iterator make progress, even when
they're not ready to accept another item. `poll_progress` fills this gap.

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
finished, we loop aroud and print `got B` immediately.

### `AsyncIterator`

#### `poll_next`

- `Poll::Pending` means... \[no change\]

- `Poll::Ready(Some(val))` means that the async iterator has successfully
  produced a value, `val`, and may produce further values on subsequent
  `poll_next` calls. In this case **the caller must arrange to call either
  `poll_next` or `poll_progress` again promptly.** (If the caller is another
  `poll_next` method, it can trust its own caller to arrange this.)

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
