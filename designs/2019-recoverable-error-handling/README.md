- Start Date: 2019-03-29
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima &lt;[@mysticatea](https://github.com/mysticatea)&gt;

# Recoverable Error Handling

## Summary

ESLint cannot verify source code if the code has a syntax error. However, we can make valid AST even if several kinds of syntax errors existed. For example, conflict of variable names doesn't break AST. This RFC calls such a syntax error as <a name="recoverable-errors" href="#recoverable-errors">"Recoverable Errors"</a>.

This RFC adds handling of [Recoverable Errors] into ESLint.

## Motivation

The goal of this RFC is that improve ESLint experience by reducing "fixing an error makes more errors."

## Detailed Design

### Handling [Recoverable Errors] in ESLint

1. `Linter` class passes `parserOptions.recoverableErrors` option with `true` to `espree` or custom parsers.
1. If the object the parser returned has `recoverableErrors` property with an array, `Linter` class converts the errors to messages.

Each element of `recoverableErrors` array has the following form.

```jsonc
{
    "message": "Identifier 'foo' has already been declared",
    "line": 1,      // 1-based line number.
    "column": 10,   // 0-based column number.
    "endLine": 1,   // Optional. 1-based line number.
    "endColumn": 13 // Optional. 0-based column number.
}
```

Then `Linter` class converts that to a message:

```jsonc
{
    "fatal": false,
    "ruleId": null,
    "severity": 2,
    "message": "Identifier 'foo' has already been declared",
    "line": 1,      // 1-based line number.
    "column": 11,   // 1-based column number.
    "endLine": 1,   // Optional. 1-based line number.
    "endColumn": 14 // Optional. 1-based column number.
}
```

Directive comments such as `/*eslint-disable*/` cannot hide the messages of [Recoverable Errors].

### Handling [Recoverable Errors] in Espree

Acorn, the underlying of `espree`, has `raiseRecoverable(pos, message)` method to customize handling of [Recoverable Errors].

If `options.recoverableErrors` was `true` then `espree` collects [Recoverable Errors] and returns the errors along with AST. Otherwise, `espree` throws syntax errors on [Recoverable Errors] as is currently.

In `acorn@6.1.1`, there are the following [Recoverable Errors]:

- "Comma is not permitted after the rest element"
- "Parenthesized pattern"
- "Redefinition of `__proto__` property"
- "Redefinition of property"
- "Binding XXX in strict mode"
- "Assigning to XXX in strict mode"
- "Argument name clash"
- "Export 'XXX' is not defined"
- "Multiple default clauses"
- "Identifier 'XXX' has already been declared"
- "Escape sequence in keyword XXX"
- "Invalid regular expression: /a regexp/: An error description"

> https://github.com/acornjs/acorn/search?q=raiseRecoverable

### Other parsers

This RFC doesn't contain the update of custom parsers. But this section considers if some popular custom parsers can applicate this feature.

- `babel-eslint`<br>
  I don't have enough knowledge about `babel-eslint` and [Recoverable Errors].
- `@typescript-eslint/parser`<br>
  TypeScript parser parses source code loosely, then provides syntax/semantic errors by API along with AST. So currently `@typescript-eslint/parser` manually throws syntax errors if the parse result has syntax/semantic errors. Therefore, it can provide [Recoverable Errors].
- `vue-eslint-parser`<br>
  It reports [HTML parse errors](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors) and JavaScript syntax errors in `<template>` as [Recoverable Errors], then [vue/no-parsing-error](https://eslint.vuejs.org/rules/no-parsing-error.html) rule converts the errors to messages. Therefore, it can provide [Recoverable Errors].

## Documentation

- [Disabling Rules with Inline Comments](https://eslint.org/docs/user-guide/configuring#disabling-rules-with-inline-comments) section should note about [Recoverable Errors]. The directive comments cannot hide the [Recoverable Errors].
- [Working with Custom Parsers](https://eslint.org/docs/developer-guide/working-with-custom-parsers) should describe about `options.recoverableErrors` and `recoverableErrors` property of the returned value. Custom parsers can use `recoverableErrors` property instead of throwing fatal errors to report syntax errors.

## Drawbacks

- I think the value of this feature is relatively small because people can verify source code after they fixed syntax errors. This feature provides just efficient.

## Backwards Compatibility Analysis

This is not a breaking change.

- `Linter` class will handle a new property.
- `espree` package will recognize `recoverableErrors` option and change that behavior if the `recoverableErrors` option was `true`.

If users are depending `parserOptions.recoverableErrors`, possibly it will be broken. But I believe that we don't need to be worried about the case.

## Alternatives

- `vue-eslint-parser`'s way is an alternative.

## Open Questions

Nothing in particular.

## Frequently Asked Questions

Nothing in particular.

## Related Discussions

- https://github.com/eslint/eslint/issues/3815
- https://github.com/eslint/espree/issues/368
- https://github.com/eslint/eslint/pull/11509#pullrequestreview-220174854

[Recoverable Errors]: #recoverable-errors
