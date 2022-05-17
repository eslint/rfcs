- Repo: eslint/eslint
- Start Date: 2021-11-30
- RFC PR: <https://github.com/eslint/rfcs/pull/84>
- Authors: [bmish](https://github.com/bmish)

# Stricter validation of rule tests

## Summary

<!-- One-paragraph explanation of the feature. -->

We will add additional assertions to the rule test runner to address several potential points of poor test coverage, especially related to [rule suggestions](https://eslint.org/docs/developer-guide/working-with-rules#providing-suggestions).

Assertions added to invalid test case error objects:

- Must contain `suggestions` if the test case produces suggestions
- Must contain the violation message (`message` or `messageId`, but not both)

Assertions added to invalid test case suggestion objects:

- Must contain the suggestion code `output`
- Must contain the suggestion message (`messageId` or `desc`, but not both)
- The suggestion code `output` must differ from the test case `code`

Assertions added to all test cases:

- Any autofix `output` must differ from the test case `code`
- Cannot have identical test cases
- The optional `filename` property must be a string
- The optional `only` property must be a boolean

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Poor rule test coverage can lead to:

- Bugs in rules or rule suggestions
- Increased burden on rule authors and rule reviewers to manually detect issues that better tests could have caught
- Reduced quality of the ESLint plugin ecosystem

To address these issues, we should automatically detect critical areas of missing test coverage from the rule test runner itself, and enforce that rule tests cover these areas.

The new assertions will fill in gaps and address inconsistencies in the existing rule test runner assertions.

This continues a trend in which `RuleTester` validation has become stricter with each major release of ESLint.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

This section contains a sub-section for each new category of assertion.

Code changes:

- Assertions will be added in the `lib/rule-tester/rule-tester.js` file in the `testInvalidTemplate()` function
- Tests will be added for each assertion in `tests/lib/rule-tester/rule-tester.js`

### Invalid test case error objects must contain `suggestions` if the test case produces suggestions

This would enforce:

```pt
AssertionError [ERR_ASSERTION]: The rule produced suggestion(s). Please add a 'suggestions' property.
```

This change has precedent in the existing behavior today where invalid test cases that produce an autofix require an `output` assertion (this behavior was added as a [breaking change](https://eslint.org/docs/user-guide/migrating-to-7.0.0#additional-validation-added-to-the-ruletester-class) in ESLint v7):

```pt
AssertionError [ERR_ASSERTION]: The rule fixed the code. Please add 'output' property.
```

More about motivation:

- It's easy to forget to assert what suggestions a test case produces, or that a test case produces no suggestions
- It's inconvenient to have to add `suggestions: []` to every single invalid test case that does not produce suggestions, in order to assert that they don't produce suggestions
- Suggestions are a newer feature, and many existing test cases may not have been updated to test newly-added suggestions

In addition, we will add a new shorthand `suggestions: <number>` that can be used to avoid testing the details of the suggestions, similar to the existing shorthand for errors `errors: <number>` which will continue to be allowed in all situations.

### Invalid test case error and suggestion objects must include the message

This would enforce:

```pt
Test error object must contain exactly one of 'message' or 'messageId'.
```

```pt
Test suggestion object must contain exactly one of 'messageId' or 'desc'.
```

This has the effect of enforcing that error and suggestion objects cannot be empty, which is regrettably allowed today. Note that if we decided that the message should not be required in test cases, we would instead want to add an assertion that test case objects are not empty at the minimum.

Note that there's an existing assertion that already enforces part of this:

```pt
AssertionError [ERR_ASSERTION]: Error should not specify both 'message' and a 'messageId'.
```

### Each suggestion object requires the `output` property

In addition to the message provided by the suggestion, the actual code output is another key part of the suggestion that tests should require:

```pt
AssertionError [ERR_ASSERTION]: Test suggestion object is missing 'output`.
```

### Autofix and suggestion `output` must differ from the test case `code`

Today, a non-fixable invalid test case can be written like this:

```js
{
    code: 'console.log("foo");',
    output: 'console.log("foo");',
    // ...
}
```

This is not ideal, especially when the code is longer and multi-line:

- It's not immediately clear whether the `code` differs from the `output`, making it difficult to determine whether or not the test case produces an autofix
- The maintenance burden is higher as changes to the test case need to be replicated in both `code` and `output`
- The test case is less concise, needlessly increasing the size/length of the test file

So we will disallow repeating the test case `code` as `output` for non-fixable test cases using a new assertion:

```pt
AssertionError [ERR_ASSERTION]: Test error object 'output' matches 'code'. Omit 'output' or use `output: null` for non-fixable test cases.
```

And similarly, suggestions must actually produce a change:

```pt
AssertionError [ERR_ASSERTION]: Test suggestion object 'output' matches 'code' and is thus a no-op.
```

To clearly and concisely indicate that a test case produces no autofix, it is recommended to omit the `output` property entirely. Note that we will also allow `output: null` or `output: undefined` which can be useful when dynamically generating test cases (e.g. `output: hasAutofix ? autofixedCode : null`), and because many existing test cases are written with `output: null` as that used to be necessary to assert that the test case had no autofix (it isn't necessary anymore as of [ESLint v7](https://eslint.org/docs/user-guide/migrating-to-7.0.0#additional-validation-added-to-the-ruletester-class)).

An existing lint rule [eslint-plugin/prefer-output-null](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin/blob/master/docs/rules/prefer-output-null.md) enforces the desired behavior and is autofixable which should help with migration.

### Disallow identical test cases

Duplicate test cases can cause confusion, can be hard to detect manually in a long file, and serve no purpose.

We will add an assertion when a duplicate test case is detected:

```pt
AssertionError [ERR_ASSERTION]: Detected duplicate test case.
```

As test cases end up represented as objects, the problem here is essentially to detect duplicate objects in an array. To handle this, here's one efficient algorithm (also mentioned in this [article](https://medium.com/programming-essentials/how-to-know-if-an-array-has-duplicates-in-javascript-27d99ab9a0d2)):

1. Maintain a set of the test cases we have seen so far.
2. For each test case, use our existing dependency [json-stable-stringify-without-jsonify](https://www.npmjs.com/package/json-stable-stringify-without-jsonify) to get a string representation of the test case object.
   - This library handles deep-sorting of object keys, unlike `JSON.stringify()`.
   - We need to skip test cases that contain non-serializable properties like functions or RegExp objects. Since rule options can contain any number of non-serializable properties, we will just skip all test cases with rule options (same goes for `settings` and `parserOptions`, or for flat config: `settings`, `languageOptions`, and `plugins`). Detecting duplicates isn't critical, so it's okay that we will skip some test cases, but we could potentially make this smarter later.
3. If the string is in the set already, assert.
4. If the string is not in the set, add it.

Here's an example [implementation](https://github.com/ember-template-lint/ember-template-lint/pull/2279) in a similar linter called [ember-template-lint](https://github.com/ember-template-lint/ember-template-lint).

An alternative implementation would be checking each test case against every other test case with our existing dependency [fast-deep-equal](https://www.npmjs.com/package/fast-deep-equal), but this could have potentially worse performance (`O(n^2)`) with large numbers of test cases.

An existing lint rule [eslint-plugin/no-identical-tests](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin/blob/master/docs/rules/no-identical-tests.md) enforces the desired behavior and is autofixable which should help with migration.

### Type checks

Most test properties already have asserts to type-check their values, or they will cause a test to fail if they are provided with the wrong type. But there are two properties that are missing type-checking and for which an invalid type won't cause tests to fail:

- `filename: 123`
- `only: 123`

So we will add assertions:

```pt
AssertionError [ERR_ASSERTION]: Optional test case property 'filename' must be a string
```

```pt
AssertionError [ERR_ASSERTION]: Optional test case property 'only' must be a boolean
```

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The [RuleTester](https://eslint.org/docs/developer-guide/nodejs-api#ruletester) section of the [Node.js API](https://eslint.org/docs/developer-guide/nodejs-api#ruletester) documentation page will be updated to mention when certain rule test case properties are required.

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

Plugin authors who left out key properties from their test cases will see a slight increase in testing burden.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This RFC describes a set of breaking changes that can be released in the next major version of ESLint. These changes will only affect ESLint plugin authors, and only when they perform the relevant major ESLint version upgrade internally in their repositories. Some plugin authors will need to add test coverage by filling in missing properties. Users of ESLint will not be affected.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

The alternative is not adding additional validation, allowing ESLint tests to be more of a free-for-all with varying levels of quality and coverage.

I also considered whether we could make validation even stricter than what's proposed here. In terms of required vs. optional test case properties, I didn't see any other properties that I thought should become mandatory. `type`, `line`, `column`, `endLine`, and `endColumn` are optional error object properties that can be useful but aren't critical to include in all test cases.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I will implement this change.

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

- [ESLint 5: RuleTester now uses strict equality checks in its assertions](https://eslint.org/docs/user-guide/migrating-to-5.0.0#rule-tester-equality)
- [ESLint 6: RuleTester now validates against invalid default keywords in rule schemas](https://eslint.org/docs/user-guide/migrating-to-6.0.0#ruletester-now-validates-against-invalid-default-keywords-in-rule-schemas)
- [ESLint 6: RuleTester now requires an absolute path on parser option](https://eslint.org/docs/user-guide/migrating-to-6.0.0#ruletester-now-requires-an-absolute-path-on-parser-option)
- [ESLint 7: Additional validation added to the RuleTester class](https://eslint.org/docs/user-guide/migrating-to-7.0.0#additional-validation-added-to-the-ruletester-class)
- [2019 RuleTester Improvements RFC](https://github.com/eslint/rfcs/tree/main/designs/2019-rule-tester-improvements)
- [Original issue proposing that suggestions must be tested](https://github.com/eslint/eslint/issues/15104)
