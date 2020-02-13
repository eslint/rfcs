- Start Date: 2019-06-07
- RFC PR: https://github.com/eslint/rfcs/pull/25
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# `RuleTester` Improvements

## Summary

This RFC improves `RuleTester` to check more mistakes.

## Motivation

`RuleTester` overlooks some mistakes.

- Using non-standard properties of AST ([typescript-eslint/typescript-eslint#405](https://github.com/typescript-eslint/typescript-eslint/issues/405)).<br>
  Especially, `node.start` and `node.end` exist in AST `espree` made, but it's not standardized and some custom parsers don't make those properties. But `node.loc` has `start`/`end` properties, so it's hard to detect `node.start`/`node.end` with static analysis. Therefore, `RuleTester` should detect those.
- Untested autofix.<br>
  If people forgot to write `output` property in test cases, `RuleTester` doesn't test autofix silently.
- `errors` property with a number (found in [eslint/eslint#11798](https://github.com/eslint/eslint/pull/11798)).<br>
  `errors` property with a number ignores syntax errors in test code. We overlooked the mistake of [tests/lib/rules/complexity.js#L84](https://github.com/eslint/eslint/blob/cb1922bdc07e58de0e55c13fd992dd8faf3292a4/tests/lib/rules/complexity.js#L84) due to this. The number `errors` property cannot check the reported error was the expected error.
- Typo property names in `errors` property with objects.<br>
  [eslint/eslint#12096](https://github.com/eslint/eslint/pull/12096).

## Detailed Design

1. Disallowing `node.start` and `node.end`
1. Ensuring to test autofix
1. Changing the `errors` property of a number to fail on syntax errors
1. Changing the `errors` property of objects to fail on unknown properties

### 1. Disallowing `node.start` and `node.end`

`RuleTester` fails test cases if a rule implementation used `node.start` or `node.end` in the test case.

#### Implementation

- In `RuleTester`, it registers an internal custom parser that wraps `espree` or the parser of `item.parser` to `Linter` object.
- The internal custom parser fixes the AST that the original parser returned, as like [test-parser.js](https://github.com/eslint/eslint/blob/21f3131aa1636afa8e5c01053e0e870f968425b1/tools/internal-testers/test-parser.js).

### 2. Ensuring to test autofix

`RuleTester` fails test cases if a rule implementation fixed code but `output` property was not defined in the test case.

#### Implementation

- If `output` property didn't exist but the rule fixed the code, `RuleTester` fails the test case as "The rule fixed the code. Please add 'output' property." It's implemented around [lib/rule-tester/rule-tester.js#L594](https://github.com/eslint/eslint/blob/21f3131aa1636afa8e5c01053e0e870f968425b1/lib/rule-tester/rule-tester.js#L594).

### 3. Changing the `errors` property of a number to fail on syntax errors

`RuleTester` fails test cases always if the `code` has a syntax error.

#### Implementation

- Unwrap [lib/rule-tester/rule-tester.js#L414-L419](https://github.com/eslint/eslint/blob/02d7542cfd0c2e95c2222b1e9e38228f4c19df19/lib/rule-tester/rule-tester.js#L414-L419).

### 4. Changing the `errors` property of objects to fail on unknown properties

`RuleTester` fails test cases if any item of `errors` has unknown properties.

#### Implementation

- [eslint/eslint#12096](https://github.com/eslint/eslint/pull/12096)

## Documentation

[RuleTester](https://eslint.org/docs/developer-guide/nodejs-api#ruletester) should be updated.

- `output` ("optional" → "required if the rule fixes code")

## Drawbacks

This change may enforce plugin owners to fix their tests.

## Backwards Compatibility Analysis

This is a breaking change that can break existing tests.

But the breaking cases may indicate that the rule was not tested enough.

## Alternatives

- About "Disallowing `node.start` and `node.end`", we can standardize those properties. But it's a breaking change for custom parser owners. On the other hand, using `node.start` and `node.end` breaks the rule if users used custom parsers, so the impact of this disallowing is limited.

## Related Discussions

- https://github.com/eslint/eslint/issues/8956
- https://github.com/eslint/eslint/pull/8984
- https://github.com/eslint/eslint/pull/11798
- https://github.com/eslint/eslint/pull/12096
- https://github.com/typescript-eslint/typescript-eslint/issues/405
