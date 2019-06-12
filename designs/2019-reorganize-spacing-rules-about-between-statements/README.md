- Start Date: 2019-06-12
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima &lt;https://github.com/mysticatea&gt;

# Reorganizing Spacing Rules about between Statements

## Summary

This RFC reorganizes spacing rules about between two statements.

## Motivation

The rules which check the spacing between statements are lacking. The `semi-spacing` rule checks a part of that, but it doesn't fit semantics.

```js
function f() {}f();
//             ^- missing space, but no rule checks.

previousStat();f();
//             ^- missing space, `semi-spacing` rule checks.

   ;;;;
//  ^^^- missing spaces, but no rule checks.

if (condition)f();
//            ^- missing space, but no rule checks.

if (condition);
//            ^- missing space, but no rule checks.

if (condition){}
//            ^- missing space, `space-before-blocks` rule checks.
```

After this RFC, it will be:

```js
function f() {}f();
//             ^- missing space, `space-between-statements` rule checks.

previousStat();f();
//             ^- missing space, `space-between-statements` rule checks.

   ;;;;
//  ^^^- missing spaces, `space-between-statements` rule checks.

if (condition)f();
//            ^- missing space, `space-before-dangling-statement` rule checks.

if (condition);
//            ^- missing space, `space-before-dangling-statement` rule checks.

if (condition){}
//            ^- missing space, `space-before-blocks` rule checks.
```

## Detailed Design

- Change the options of `semi-spacing` rule
- Add `space-between-statements` rule
- Add `space-before-dangling-statement` rule

### § Change the options of `semi-spacing` rule

Deprecate `after` option in order to separate the check about the spacing between statements.
After this RFC, `semi-spacing` rule will focus on inside of each statement.

```jsonc
{
    "semi-spacing": ["error", "always" | "never"]
}
```

- `"never"` (default) ... disallows the spacing between each statement and the statement's `;`. This is the same as `before:false` option of the current `semi-spacing` rule, but it doesn't check spacing after `;`.
- `"always"` ... requires the spacing between each statement and the statement's `;`. This is the same as `before:true` option of the current `semi-spacing` rule, but it doesn't check spacing after `;`.

People can use existing `{ after: boolean; before: boolean }` form after this RFC as following [our deprecation policy], but it's deprecated.

#### Migration Steps

1. (minor) Add `"always" | "never"` option support and soft-deprecate `{ after: boolean; before: boolean }` form.
1. (major) Change the default option to `"never"` from `{ after: true, before: false }`. This is a breaking change that reduces errors (it no longer checks spaces after semicolons).
1. (major) Add the deprecation warning to `{ after: boolean; before: boolean }` form.

### § Add `space-between-statements` rule

The `space-between-statements` rule checks the spacing between two statements. This rule checks `Program#body` and `BlockStatement#body`.

This rule has a string option and an object option.

```jsonc
{
    "space-between-statements": [
        "error",
        "always" | "never",
        { "leadingSemi": "always" | "never" }
    ]
}
```

- `"always"` (default) ... requires one or more spaces between two statements. This is similar to `after:true` option of the current `semi-spacing` rule, but covers also the cases that the previous statement doesn't have that semicolon.
- `"never"` ... disallows any space between two statements. This is similar to `after:false` option of the current `semi-spacing` rule, but covers also the cases that the previous statement doesn't have that semicolon.
- `"leadingSemi"` ... this setting is used if the semicolon of the previous statement was at the beginning of the line. (E.g., `;[head, ...rest] = list`). Default is the same value as the first option.

**Bad code:**

```js
/* space-between-statements: [error] */

function f() {}f();
//             ^- missing space

previousStat();f();
//             ^- missing space
```

**Good code:**

```js
/* space-between-statements: [error] */

function f() {} f();
//             ^- space

previousStat(); f();
//             ^- space
```

### § Add `space-before-dangling-statement` rule

The `space-before-dangling-statement` rule checks the spacing between each statement head and the statement's body only if the body was not a block. If the body was a block, the `space-before-blocks` rule checks it.

This rule checks the following statements.

- `DoWhileStatement`
- `ForStatement`
- `IfStatement` (both `consequent` and `alternate`)
- `LabeledStatement`
- `WhileStatement`
- `WithStatement`

This rule has a string option and an object option.

```jsonc
{
    "space-before-dangling-statement": [
        "error",
        "always" | "never",
        { "empty": "always" | "never" }
    ]
}
```

- `"always"` (default) ... requires one or more spaces between the head and the body.
- `"never"` ... disallows any space between the head and the body.
- `empty` ... this setting is used if the body was an `EmptyStatement`. Default is the same value as the first option.

**Bad code:**

```js
/* space-before-dangling-statement: [error] */

if (condition)f();
//            ^- missing space

if (condition);
//            ^- missing space
```

**Good code:**

```js
/* space-before-dangling-statement: [error] */

if (condition) f();
//            ^- space

if (condition) ;
//            ^- space
```

## Documentation

Rules documentation should be updated.

## Drawbacks

The change of `semi-spacing` rule forces users to migrate their configuration.

## Backwards Compatibility Analysis

The change of `semi-spacing` rule is a breaking change. We should leave the existing option as following [our deprecation policy].

## Alternatives

- Maybe we should add new rule instead of `semi-spacing` because deprecating rule is easier than deprecating option. But I have not a good idea about the rule name.
- Maybe we can merge new `space-before-dangling-statement` and existing `space-before-blocks`. But `space-before-blocks` covers more cases such as class bodies.

## Open Questions

## Frequently Asked Questions

## Related Discussions

- https://github.com/eslint/eslint/issues/11721
- https://github.com/eslint/tsc-meetings/blob/master/notes/2019/2019-05-23.md#semi-spacing-rule-doesnt-work-after-while

[our deprecation policy]: https://eslint.org/docs/user-guide/rule-deprecation
