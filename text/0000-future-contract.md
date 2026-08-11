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

Consider this contrived deadlock example:

```rust
async fn foo() {
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
broader than `select!`, let's use `timeout` instead:

```rust
#[tokio::main]
async fn main() {
    let mut future1 = pin!(foo());
    _ = timeout(Duration::from_millis(1), &mut future1).await;
    foo().await; // Deadlock!
}
```

This still isn't very realistic, because nothing is forcing us to use `pin!`
here. (And passing `future1` to `timeout` by value would fix the deadlock,
because then we'd drop it when the timeout expires.) Usually we only need
`pin!` when we're driving a future in a loop. Let's put the loop in, and while
we're add it we'll add a couple layers of abstraction around `foo`:

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
