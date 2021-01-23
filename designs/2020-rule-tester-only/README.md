- Repo: eslint/eslint
- Start Date: 2020-12-27
- RFC PR: (leave this empty, to be filled in later)
- Authors: Brandon Mills ([@btmills](https://github.com/btmills))

# `RuleTester` test isolation with `only`

## Summary

<!-- One-paragraph explanation of the feature. -->

`RuleTester` currently lacks a built-in way for developers to isolate individual tests during development.
Temporarily deleting or commenting out the other tests is tedious.
This adds an optional `only` property to run individual tests in isolation.

## Motivation

<!-- Why are we doing this? What uses does it support? What is the expected
outcome? -->

When developers are working on a rule, we can run its tests with `npm run test:cli tests/lib/rules/my-rule.js`.
Debugging sometimes requires focusing on a particular test and running it in isolation to avoid unrelated noise from other tests.

Tools like Mocha already offer `it.only()` for low-friction test isolation.
Developers may already be familiar with that API.
Exposing a similar API via `RuleTester` would improve the ergonomics of debugging tests.
This API is beneficial to any approach from `console.log("HERE");` to interactive debugging with breakpoints.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corners, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

Add an optional boolean `only` property to `RuleTester`'s `ValidTestCase` and `InvalidTestCase` types.
Add `only` to the other `RuleTester`-only properties in the `RuleTesterParameters` array to exclude it from configuration schema validation.

To allow using `RuleTester` with a custom test framework other than Mocha, parallel `RuleTester`'s existing `describe` and `it` implementations for `itOnly`:

1. Declare an `IT_ONLY` `Symbol`, set it as a static property on `RuleTester`, and initialize it to `null`.
1. Add an `itOnly` `set` accessor that sets `RuleTester[IT_ONLY]`.
1. Add an `itOnly` `get` accessor.
   1. If `RuleTester[IT_ONLY]` is set, return it.
   2. If global `it` and `it.only` are functions, return `Function.bind.call(it.only, it)`.
   3. Throw an error:
      1. If either `RuleTester[DESCRIBE]` or `RuleTester[IT]` is customized, recommend setting a custom `RuleTester.itOnly`.
      2. If global `it` is a function, the current test framework does not support `only`.
      3. Otherwise recommend installing a test framework like Mocha so that `only` can be used.

At the end of `RuleTester`'s `run()` method, for each valid and invalid item, if `only` is `true`, call `RuleTester.itOnly` instead of `RuleTester.it`.

Add [`--forbid-only`](https://mochajs.org/#-forbid-only) to the [`mocha` target in `Makefile.js`](https://github.com/eslint/eslint/blob/cc4871369645c3409dc56ded7a555af8a9f63d51/Makefile.js#L548) to prevent accidentally merging tests before removing `only`.

Add a static `only()` convenience method on `RuleTester`.
This adds `only: true` to a test and automatically converts string tests to objects if necessary.

```js
/**
 * Adds the `only` property to a test to run it in isolation.
 * @param {string | ValidTestCase | InvalidTestCase} item A single test to run by itself.
 * @returns {ValidTestCase | InvalidTestCase}
 */
static only(item) {
    if (typeof item === "string") {
        return { code: item, only: true };
    }

    return { ...item, only: true };
}
```

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The [RuleTester section](https://eslint.org/docs/developer-guide/nodejs-api#ruletester) in our Node.js API docs will need to include a description of the `only` property on test case objects.

The [Unit Tests](https://eslint.org/docs/developer-guide/unit-tests) page in the developer guide will also include a note about this next to the [Running Individual Tests](https://eslint.org/docs/developer-guide/unit-tests#running-individual-tests) section.
This can also recommend enabling [`--forbid-only`](https://mochajs.org/#-forbid-only) to prevent accidentally merging tests before removing `only`.

## Drawbacks

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think 
    about any opposing viewpoints that reviewers might bring up. 

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

This exposes new API surface.
That is mitigated by the implementation still being a wrapper around Mocha's existing API like the rest of `RuleTester`.

Deleted or commented-out tests are hard to miss in diffs.
An extra `only: true` could more easily sneak through, accidentally disabling all other tests.
Calling Mocha with [`--forbid-only`](https://mochajs.org/#-forbid-only) will prevent that on CI runs, but it still requires third-party developers to opt in.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This change is backwards compatible.
`RuleTester` currently fails any tests that include unrecognized properties like `only`, so we have no risk of breaking existing tests.

If someone has customized `RuleTester` with a custom `it()` method, we cannot assume that `it.only()` has the same semantics as Mocha.
Their existing tests will still work, but if they wish to use this new `only` property, they would need to provide a custom `itOnly()` alongside `it()`.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

This builds upon Mocha's established `it.only()` API as prior art.

The status quo has two alternatives:

1. We can temporarily delete or comment out the other tests. This is tedious.
1. We can filter through noise by scrolling through `console.log()` output or using conditional breakpoints, though the latter is not always possible.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

1. We could define a static `only()` convenience method on `RuleTester`.
Given, for example, a valid string test `"var foo = 42;"`, it is easier to convert to `RuleTester.only("var foo = 42;")`, which is equivalent to `{ code: "var foo = 42;", only: true }`.
The helper sets `only: true` on a test case argument after converting string cases to objects if necessary.
For invalid cases that are already objects, adding `only: true` is likely easier than using the helper.

    A: The convenience is worth the simple implementation, so I've added this to the detailed design.

1. ~~Should using `only` cause the process to exit `1` even if tests pass as described in "Drawbacks"?~~
Our Makefile can call Mocha with [`--forbid-only`](https://mochajs.org/#-forbid-only) instead.
3. In the `RuleTester.itOnly` `get` accessor, if `RuleTester[IT_ONLY]` is not customized and the global `it.only` is not a function, is throwing an error the right choice?
We could instead pass through to `RuleTester.it`, ignoring `only` if it's not supported.
1. Do we need to support `skip`, the inverse of `only`?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I can implement this myself.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- This was suggested as an alternative solution to [RFC67](https://github.com/eslint/rfcs/pull/67) and [eslint/eslint#13625](https://github.com/eslint/eslint/issues/13625) because that change may not be possible as currently described.
