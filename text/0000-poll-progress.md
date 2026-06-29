- Feature Name: `poll_progress`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

## Summary
[summary]: #summary

Add a required `poll_progress` method to the `AsyncIterator` trait, and make
`for await` loops call this method whenever an `.await` in their loop body is
`Pending`. Expand the documented `AsyncIterator` contract to require async
iterator combinators to do the same.

## Motivation
[motivation]: #motivation

Since the [`AsyncIterator`] trait is still unstable, iteration in the async
ecosystem today is organized around the [`Stream`] trait from the `futures`
crate. Apart from the name, the two traits currently have the same shape:

[`AsyncIterator`]: https://doc.rust-lang.org/std/async_iter/trait.AsyncIterator.html
[`Stream`]: https://docs.rs/futures/latest/futures/prelude/trait.Stream.html

```rs
pub trait AsyncIterator { // or `pub trait Stream`
    type Item;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) { ... }
}
```

In this interface, all the work an iterator does is driven through `poll_next`.
Consider how that interacts with a `for await` loop:

```rs
for await item in my_iter {
    do_work(item).await;
}
```

When control is at the top, the `for await` loop calls `my_iter.poll_next`
until it either yields an item (`Ready(Some(_))` or indicates that it's done
(`Ready(None)`). Once control moves into `do_work`, the loop stops driving
`my_iter`. That applies necessary "backpressure" to the iterator, so it's
mostly by design. But it can be a problem if `my_iter` wraps multiple
concurrent futures or other iterators internally, because suspending some of
those at arbitrary `.await` points isn't generally correct. Here's an example
where that causes a deadlock that looks like it should be impossible in a
straight-line reading of the code:

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

`FuturesUnordered` doesn't implement `AsyncIterator` today, so this example
doesn't compile as written, but you can run it on stable by [replacing `for
await` with `for_each`][for_each]. When control enters the loop body, one of
the `foo` futures remains in the `FuturesUnordered` buffer, suspended at the
point where it's tried to acquire `LOCK` and taken a spot in its waiters queue.
The call to `foo` in the body tries acquire `LOCK` again, but the waiter at the
front of the queue never again makes progress, so it's deadlocked.

[for_each]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=ceccac95773cbba8aeddf162b5793a3f

To avoid these sorts of deadlocks, and other hard-to-diagnose hangs and
latencies, concurrent async iterators like `FuturesUnordered` need to
continuously drive futures they contain. That means that `for await` and other
combinators need a way to drive async iterators even when they're not yet ready
to accept another item.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

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
