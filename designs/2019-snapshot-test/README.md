- Start Date: 2019-07-25
- RFC PR: https://github.com/eslint/rfcs/pull/36
- Authors: Eric Wang

# Allow snapshot test

## Summary
Allow snapshot test for the linting result

## Motivation
The current test framework is not ideal for scale.
When there are quite a few rules, it becomes hard to see what's the expected result, especially for the `line` and `column`.
![B5898077-6ACC-4288-B415-1ECD06F94A96](https://user-images.githubusercontent.com/10626756/61182538-f7c96200-a677-11e9-9483-69b9374f1ab3.png)

A snapshot test will be much more intuitive for some case
![8E2DD8E6-1763-4D32-89EF-07DFC8FEDC00](https://user-images.githubusercontent.com/10626756/61182536-f009bd80-a677-11e9-834d-c4fd7a91c183.png)


## Detailed Design
Refactor the `rule-tester` to extract the `runRuleForItem` method out so that it can be consumed.
There will be a new class `rule-snapshot-tester` which consumes the `runRuleForItem` to generate the output.
Mark the error position with "~~~" in one snapshot and the fix in another.
The result will be pure string and it's user's responsibility to decide what kind of snapshot test tool to compare them.

### Difference between the existing one
The existing Api gives non-intuitive information only such as `row` and `column`
The new one will return a snapshot.

### New Apis
A `RuleSnapshotTester` which takes the same constructor parameter as the existing `RuleTester` class.
The `RuleSnapshotTester` will expose two public methods
- `generateErrorMarkingSnapshot: (rule: ESLintRule, testCase: TestCase) => string`. It will return the snapshot with error marking. Error will be marked as "~~~" under the code as a new line. If there are multiple errors with same message, "~" will be shown for all errors but only the last error message will be added as the text.
- `generateFixSnapshot:(rule: ESLintRule, testCase: TestCase) => string`. It will return the snapshot after the autofix. Errors that are not fixable will not be marked.

The user can consume it as
```
const ruleSnapshotTester = new RuleSnapshotTester();
const snapshot = ruleSnaphostTester.generateErrorMarkingSnapshot(require("../../fixtures/testers/rule-tester/no-eval"), {
  code: `
    eval(foo)
  `
})
// It's user's decision which test framework they want to use to compare the snapshot
// The Api only return the string
expect(snapshot).toMatchSnapshot();
```
## Documentation
There will be a new class `rule-snapshot-tester`.
Yes, it need a formal announcement on the ESLint blog to explain the motivation.

## Drawbacks
It won't change the existing behavior so should be OK.

## Backwards Compatibility Analysis
It doesn't change any existing Api.

## Alternatives
Can't think of any.

## Open Questions


## Help Needed
It will be a large change, how should I split the PR into smaller ones so that it's easier to review?

## Frequently Asked Questions

## Related Discussions
