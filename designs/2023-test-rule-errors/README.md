- Repo: eslint/eslint
- Start Date: 2023-01-01
- RFC PR: <https://github.com/eslint/rfcs/pull/103>
- Authors: [bmish](https://github.com/bmish)

# Support for testing invalid rule schemas and runtime exceptions in rules

## Summary

<!-- One-paragraph explanation of the feature. -->

Enable rule authors to write unit tests for invalid rule schemas and runtime exceptions in rules.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

In certain situations, an exception can be thrown when running an ESLint rule, including:

1. ESLint throws an exception because a rule is configured incorrectly. That is, the user-provided options did not conform to a rule's schema (`meta.schema`).
2. A rule implementation (in the `create()` function) throws an exception. Examples of why this could happen:
   - The rule threw a custom exception when additional input validation failed (typically used for a constraint that couldn't be represented/enforced by the schema).
   - The rule threw a custom exception because an unhandled or unexpected situation occurred, such as a type of node the rule wasn't expecting to occur in a particular position, perhaps due an oversight, or due to a new JavaScript language feature that the rule doesn't support yet.
   - The rule implementation has an unknown bug that causes it to unintentionally crash.

Other than for unanticipated cases like an unknown bug in the rule, the user may want to write tests for these situations, but is unable to do so today. It's currently only possible to write passing test cases (rule runs successfully and does not report violations) or failing test cases (rule runs successfully and reports violations).

As a result of this limitation, users wanting to promote rule quality by ensuring their rules have complete test coverage, i.e. all of the logic is exercised in tests, may be unable to do so.

### Rule schemas

Today, unit tests for a rule will ideally ensure that the rule behavior is correct for all possible combinations of valid rule options, but it is not currently possible to test that a rule correctly disallows invalid rules schemas.

For simple schemas, testing might not be as critical, but testing becomes more important when rule schemas grow more complex. Some schemas can even reach 100 lines long, and often allow various formats, such as in [no-restricted-imports](https://eslint.org/docs/rules/no-restricted-imports) which allows either an array of strings or an array of objects.

For example, with the rule [no-restricted-imports](https://eslint.org/docs/rules/no-restricted-imports), one might want to test that the rule schema fails validation when passed:

- Something that isn't an array
- An empty array
- An array containing an item that isn't a string nor object
- An array containing an object that is missing required properties like `name`, or has unknown properties
- Any other invalid combinations of input

Note that the goal is not to test that [ajv](https://github.com/ajv-validator/ajv) (which performs the validation using JSON Schema) itself works properly, as we can treat that third-party dependency as a blackbox, but instead to test that the rule author has written out their schema correctly.

It can be tricky to get schemas to perfectly represent what the allowed input should be, especially when no automated testing exists around this today. Schemas are often too lenient, allowing extra input that the rule implementation ignores or doesn't handle properly. This can result in rules silently misbehaving, not respecting the wishes or intentions of the user's configuration, or just crashing.

By enabling testing of schemas, we'll make it easier to write higher-quality schemas and rules, thus improving the user experience.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

Add support to the rule tester for a new `fatal` test case array in addition to `valid` and `invalid`.

Example with a schema validation error because a non-existent option is passed:

```json
{
    "fatal": [
        {
            "code": "foo();",
            "options": [{ "checkFoo": true, "nonExistentOption": true }],
            "error": {
                "message": "Value {\"checkFoo\":true,\"nonExistentOption\":true} should NOT have additional properties.",
                "name": "SchemaValidationError"
            }
        }
    ]
}
```

To test a custom exception thrown by a rule, simply use the exception text as the message: `"message": "Some custom exception message from the rule."` and/or the name of the exception class as the `name`.

- The `error` object groups the properties from the exception together nicely.
- Similar to `message` and `messageId` in `invalid` test cases, only one of `message` and `name` is required. In particular, if the user doesn't care to test the full exception `message` text, the exception class `name` can be used instead.
- `message` can be provided as a string or a regular expression (regexp), while `name` can only be provided as a string.
- `message` comes from `err.message`
- `name` comes from `err.name` or `err.constructor.name` in case either has a custom value (besides just `Error`)

Examples of how exceptions would be thrown with different exception classes that could be distinguished by `name`:

- `throw new SchemaValidationError('...should NOT have additional properties.')` (custom error type thrown by schema validation)
- `throw new SomeCustomRuleError('...')` (custom error type thrown by the rule implementation)
- `throw new Error('...')` (basic error thrown by the rule implementation)

Differences between `fatal` test cases and `invalid` test cases:

- `code` is not required (the code may be irrelevant when testing options)
  - If neither `code` nor `name` are provided, we'll use a placeholder name `(Test Case #x` as the test case's `it` title)
  - If `code` is not provided, we'll pass an empty string for it to `linter.verify()`
- There's a required `error` object instead of an `errors` array

This feature can only be used for testing the following user-facing exceptions:

- ESLint-generated exception during schema validation
- Rule-generated exceptions

Note that this feature cannot be used for testing exceptions thrown by the rule tester itself, nor for testing syntax/parsing errors.

This design is similar to the [API](https://github.com/ember-template-lint/ember-template-lint/blob/master/docs/plugins.md#rule-tests) that the linter [ember-template-lint](https://github.com/ember-template-lint/ember-template-lint) uses for testing rule exceptions, which could be referred to as prior art.

### Implementation

> Draft/WIP PR demonstrating a working implementation of the current design: [#16823](https://github.com/eslint/eslint/pull/16823)

The primary changes will be to the rule tester:

- lib/rule-tester/rule-tester.js
- lib/rule-tester/flat-rule-tester.js
- tests/lib/rule-tester/rule-tester.js
- tests/lib/rule-tester/flat-rule-tester.js

First, we'll follow the pattern of the existing `valid` and `invalid` test case arrays to add support for a new `fatal` test case array in the rule tester.

Then, when we are executing a test case with a fatal error, we need some way to convert any ESLint-generated exception during schema validation and any rule-generated exception during rule execution into an object that can be compared against the expected error message in the test case.

In `runRuleForItem()`, when executing a fatal test case, we'll surround the `validate()` and `linter.verify()` calls with a try/catch block, and convert the relevant exceptions to an error object to be returned.

However, there are a number of issues to solve with this:

1. Issue #1: We need some way to determine which exceptions to convert to messages, and which exceptions to let bubble up, as there are many irrelevant exceptions that can be thrown by these functions which users shouldn't be testing.
2. Issue #2: We need to omit the superfluous ESLint helper text from the error message returned, as ESLint helper text is subject to change, not part of the original exception from the schema validation or rule, and unnecessarily including these extra lines forces the user to always write out a tedious, multi-line string with them for the message in their test case (unless they use a regexp to match part of the message). Example of exceptions with the ESLint helper text:

    ```pt
    rule-tester:
        Configuration for rule "no-foo" is invalid:
        Value {"checkFoo":true,"nonExistentOption":true} should NOT have additional properties.
    ```

    ```pt
    Some custom exception message from the rule...
    Occurred while linting <input>:2
    Rule: "no-foo"
    ```

To solve these issues, we'll add a property to the relevant exceptions `err.messageForTest = '...';` at their source and then use this property to distinguish testable exceptions and what the original exception text was. The downside is we have to add a custom property to some exception objects, and this requires the source of these exceptions to have awareness of the rule tester, which is not ideal. Example code:

```js
// Inside the rule tester class.
function runRuleForItem(item) {
  // ...

  // Only surround these calls with a try/catch if the current test case is for a fatal error.
  try {
      validate(...);
      // or
      messages = linter.verify(...);
  } catch (err) {
      if (err.messageForTest) {
          // Return a message so this exception can be tested.
          return {
              messages: [{
                  ruleId: ruleName,
                  fatal: true,
                  message: err.messageForTest,
                  name: err.name === 'Error' ? err.constructor.name : err.name,
              }],
          };
      }

      // Not one of the relevant exceptions for testing.
      throw err;
  }
}
```

> Draft/WIP PR demonstrating a working implementation of the current design: [#16823](https://github.com/eslint/eslint/pull/16823)

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

We will document this change in the ESLint [RuleTester](https://eslint.org/docs/latest/developer-guide/nodejs-api#ruletester) section of the [Node.JS API](https://eslint.org/docs/latest/developer-guide/nodejs-api) page.

Ideally, we can also draw attention to it with a paragraph and example usage in the blog post / release notes for the version it ships in. This is the least we can do to raise awareness of it for rule authors who may be interested to take advantage of it in new or existing rules.

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

- A user testing that their rule schema catches invalid options may end up including the message from JSON Schema in their test case (e.g. `Value "bar" should be boolean.`). This could make it more difficult for ESLint to upgrade its version of ajv / JSON Schema in the future, as an upgraded version could tweak messages and necessitate updating any test cases that include the message. This is a relatively minor concern, as it's unlikely that messages will change often, and it's easy to update test cases if they do. A few factors that could mitigate this:
  - If [snapshot testing](https://github.com/eslint/eslint/issues/14936) is implemented in the future, it would become possible to fix test cases with a single command.
  - We can encourage the use of `error.name` property (discussed in [Detailed Design](#detailed-design)) for the majority of cases that don't care about including the exact exception text.
  - We can add a note in the documentation for this feature that JSON Schema text is subject to change (could theoretically happen during an ajv upgrade).

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This new feature is a non-breaking change and has no impact unless a user chooses to use it.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

### Reuse `invalid` test cases

> **Note:** During the RFC discussion, we decided against this alternative because fatal test cases are sufficiently-different from invalid test cases that they deserve to have their own separate array to avoid confusion.

Augment the error objects of invalid test cases to support a `fatal: true` property. This will indicate that the test case triggered an exception with the given message.

Example with a schema validation error because a non-existent option is passed:

```json
{
    "invalid": [
        {
            "code": "foo();",
            "options": [{ "checkFoo": true, "nonExistentOption": true }],
            "errors": [
                {
                    "message": "Value {\"checkFoo\":true,\"nonExistentOption\":true} should NOT have additional properties.",
                    "fatal": true
                }
            ]
        }
    ]
}
```

New constraints on test cases with a fatal error:

- `code` is not required (the code may be irrelevant when testing options)
- Only one error object can be provided (can't have multiple exceptions at the same time)
- Cannot have `output` in the top-level object (no violation/autofix can result from a fatal error)
- Cannot have `suggestions` in the error object (suggestions are only for violations)

Pros of a separate `fatal` error vs. reusing the `invalid` array:

- Avoid overloading existing invalid test cases to serve additional purposes which could complicate the usage, documentation, and implementation logic
- Avoid confusing third-party tooling that may be analyzing (e.g. linting in the case of [eslint-plugin-eslint-plugin](https://github.com/eslint-community/eslint-plugin-eslint-plugin)) or running invalid test cases with the assumption that they all report violations

Cons of a separate `fatal` error vs. reusing the `invalid` array:

- Not able to share as much code with invalid test cases
- Large change / addition to the API of rule tester, as opposed to just adding a single `error: true` property

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

1. Have we come up with the ideal [implementation](#implementation)?
2. Are there other kinds of exceptions we want to allow to be tested? The current approach essentially defines an allowlist of exceptions that are testable (invalid rule options, and custom exceptions thrown by a rule implementation) so that users can't test irrelevant exceptions (and so that we don't unnecessarily expose exceptions that are subject to change in our API). Note that we can add more exceptions later if we determine a need.

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I expect to implement this change.

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

- <https://github.com/eslint/eslint/issues/13434> - the issue triggering this RFC
- <https://github.com/eslint/rfcs/tree/main/designs/2021-stricter-rule-test-validation> - related, recent RFC to make RuleTester more strict
