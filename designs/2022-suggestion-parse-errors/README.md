- Repo: eslint/eslint
- Start Date: 2022-12-10
- RFC PR: <https://github.com/eslint/rfcs/pull/101>
- Authors: [bmish](https://github.com/bmish)

# Check for parsing errors in suggestion fixes

## Summary

<!-- One-paragraph explanation of the feature. -->

Check for parsing errors in [suggestion fixes](https://eslint.org/docs/latest/developer-guide/working-with-rules#providing-suggestions) when running rule tests, the same way we do with rule [autofix](https://eslint.org/docs/latest/developer-guide/working-with-rules#applying-fixes) output.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Over time, `RuleTester` has become stricter about verifying the validity of test cases and rule behavior, including in the recent, related [RFC-84: Stricter validation of rule tests](../2021-stricter-rule-test-validation/README.md).

One likely oversight and gap in our validation is that we don't currently check for parsing/syntax errors in suggestion fixes. This is a problem because a user applying an invalid suggestion will end up with broken code. This is confusing and a poor user experience.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

We should apply the same validation to suggestions as we do to autofixes, and throw an error if a suggestion produces invalid code.

In the rule testers:

- lib/rule-tester/flat-rule-tester.js
- lib/rule-tester/rule-tester.js

We will follow the same pattern that is used for validating autofix output. When encountering a test case that is testing suggestion output, after the existing call to `SourceCodeFixer.applyFixes()`, we need to call `linter.verify()` on the code with the applied suggestion, and then assert that there are no fatal errors.

Note that as a result of [RFC-84: Stricter validation of rule tests](../2021-stricter-rule-test-validation/README.md), test cases are required to test all suggestions, so we know all suggestions will be covered by our new check.

In the tests for the rule testers:

- tests/lib/rule-tester/flat-rule-tester.js
- tests/lib/rule-tester/rule-tester.js

We need to add a test case with an invalid suggestion, and assert that it throws.

Draft implementation: <https://github.com/eslint/eslint/pull/16639>

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

This doesn't need to be documented as its one of many checks in the rule tester for invalid rule conditions or test cases. Users should intuitively expect that parsing errors will cause failures. It can simply be called out in the migration guide for the major version it's implemented in.

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

1. There may be a slight performance overhead to checking for parsing errors when running tests. This is likely to be negligible and worthwhile to catch real bugs that will impact users.
2. It's theoretically possible that a rule author intended to provide a broken suggestion, and this RFC would prevent that. It is true that suggestions are not intended to be held to the same standards are autofixes. Suggestions can be used to provide incomplete or unsafe fixes that change code behavior, which is why a user must choose to apply a suggestion manually. However, I don't believe suggestions should be allowed to provide syntactically-invalid/unparsable fixes. Producing broken code puts an unnecessary burden on the user to figure out how to un-break their code, and prevents ESLint and other tools from running on the code in the meantime. It should be possible to turn any desired suggestion into valid code/syntax, even if it's still incomplete or not production ready.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

Plugin authors whose rules produce invalid suggestions will experience a breaking change when running tests due to the new test failure. This is desirable so that the plugin author is forced to fix the issue and release a patch.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

1. Do nothing -- continue to allow invalid suggestions. But there are minimal [drawbacks](#drawbacks) to checking for invalid suggestions and real-world benefits to users.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

1. Should we check for parsing errors when applying suggestions in core too (not just in rule tester)?

   Pros:

   - Showing an exception to end-users about the parsing error could be more clear than simply allowing the suggestion to produce broken code for them, and could encourage the end-user to file a ticket to have the issue fixed with the plugin author.

   Cons:

   - However, this additional assertion may be of limited value, as the vast majority of invalid suggestions would have already been caught by rule tests (as long as the rule actually has tests).
   - Autofixes and suggestions should both be held to the same validation standards, i.e. we can validate them both in rule tester but not in core.
   - Validating suggestions could cause a performance hit for end-users.

   In discussion, we decided not to validate suggestions in core.

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I am open to implementing this.

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

- <https://github.com/eslint/eslint/issues/15735> - the issue triggering this RFC
- <https://github.com/eslint/rfcs/pull/30> - original suggestions RFC
