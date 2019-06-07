- Start Date: 2019-03-29
- RFC PR: https://github.com/eslint/rfcs/pull/19
- Authors: Toru Nagashima &lt;[@mysticatea](https://github.com/mysticatea)&gt;

# Recoverable Error Handling

## Summary

ESLint cannot verify source code if the code has a syntax error. However, we can make valid AST even if several kinds of syntax errors existed. For example, conflict of variable names doesn't break AST. This RFC calls such a syntax error as <a name="recoverable-errors" href="#recoverable-errors">"Recoverable Errors"</a>.

This RFC adds handling of [Recoverable Errors] into ESLint.

## Motivation

The goal of this RFC is that improve ESLint experience by reducing "fixing an error makes more errors."

This feature intends to be used for the syntax errors which don't affect AST shape. For example, name conflicts, type errors, etc. This feature doesn't intend to support invalid AST.

## Detailed Design

### § Handling [Recoverable Errors] in ESLint

- `Linter` class passes `parserOptions.recoverableErrors` option with `true` to `espree` or custom parsers.
- If the object the parser returned has `recoverableErrors` property with an array, or if the error the parser thrown has `recoverableErrors` property with an array, `Linter` class converts the errors to messages. 

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
        "fatal": true,
        "ruleId": null,
        "severity": 2,
        "message": "Identifier 'foo' has already been declared",
        "line": 1,      // 1-based line number.
        "column": 11,   // 1-based column number.
        "endLine": 1,   // Optional. 1-based line number.
        "endColumn": 14 // Optional. 1-based column number.
    }
    ```

- Directive comments such as `/*eslint-disable*/` cannot hide the messages of recoverable errors.
- ESLint doesn't run any rules if a recoverable error existed.

Practically, this is the support for multiple syntax errors.

#### `verifyOnRecoverableParsingErrors` option

`verifyOnRecoverableParsingErrors` option is the following three:

- `--verify-on-recoverable-parsing-errors` CLI option
- `verifyOnRecoverableParsingErrors` in `CLIEngine` constructor option (`boolean`, default is `false`)
- `verifyOnRecoverableParsingErrors` in `Linter#verify()` option (`boolean`, default is `false`)

<table><tr><td>
<b>An aside:</b><br>
And <a href="https://github.com/eslint/rfcs/pull/22">#22</a> <code>coreOptions.verifyOnRecoverableParsingErrors</code> in config files if both RFCs accepted.
</td></tr></table>

If the `verifyOnRecoverableParsingErrors` option was given, ESLint runs configured rules even if the parser returned recoverable errors. In that case, ESLint additionally controls lint messages to avoid confusion.

If the parser returned any recoverable errors:

- the `Linter` disables autofix by making [`disableFixes` option](https://eslint.org/docs/6.0.0/developer-guide/nodejs-api#linterverify) `true` internally as autofix is considered not safe.
- the `Linter` catches exceptions which were thrown from rules and reports the exceptions as regular messages rather than crash in order to provide linting messages as best effort basis. For example,

    ```jsonc
    {
        "fatal": true,
        "ruleId": "a-rule",
        "severity": 2,
        "message": "'a-rule' failed to lint the code because of parsing error(s).",
        "line": 1,
        "column": 1,
        "endLine": 1,
        "endColumn": 1
    }
    ```

    If a rule has an exception and regular messages, the `Linter` drops the regular messages to avoid confusion of wrong messages.

### § Handling [Recoverable Errors] in Espree

Acorn, the underlying of `espree`, has `raiseRecoverable(pos, message)` method to customize handling of recoverable errors.

If `options.recoverableErrors` was `true` then `espree` collects recoverable errors and returns the errors along with AST. Otherwise, `espree` throws syntax errors on recoverable errors as is currently.

In `acorn@6.1.1`, there are the following recoverable errors:

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

<table><tr><td>
<b>An aside:</b><br>
A crazy idea is that we can make the parsing errors which are caused by older <code>ecmaVersion</code> recoverable. The parser parses code with the latest <code>ecmaVersion</code> always, then reports newer syntaxes as recoverable errors with understandable messages such as "async functions are not supported in ES5. Please set '2017' to 'parserOptions.ecmaVersion'."
</td></tr></table>

### § Handling [Recoverable Errors] in Other Parsers

This RFC doesn't contain the update of custom parsers. But this section considers if some popular custom parsers can applicate this feature.

- `babel-eslint`<br>
  I don't have enough knowledge about `babel-eslint` and recoverable errors.
- `@typescript-eslint/parser`<br>
  TypeScript parser parses source code loosely, then provides syntax/semantic errors by API along with AST. So currently `@typescript-eslint/parser` manually throws syntax errors if the parse result has syntax/semantic errors. Therefore, it can provide recoverable errors.
- `vue-eslint-parser`<br>
  It reports [HTML parse errors](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors) and JavaScript syntax errors in `<template>` as recoverable errors, then [vue/no-parsing-error](https://eslint.vuejs.org/rules/no-parsing-error.html) rule converts the errors to messages. Therefore, it can provide recoverable errors.

### How should custom parsers use AST with [Recoverable Errors]?

This feature intends to be used for the syntax errors which don't affect AST shape. For example, name conflicts, type errors, etc. This feature doesn't intend to support invalid AST.

If recovering of a syntax error makes invalid AST as ESTree or unexpected mapping between AST and tokens, it should not be recoverable errors.

### How should rules support AST with [Recoverable Errors]?

If given AST is valid along with recoverable errors, rules should support that case.

If mapping between AST nodes and tokens was broken, probably rules cannot support that case.

## Documentation

- [Disabling Rules with Inline Comments](https://eslint.org/docs/user-guide/configuring#disabling-rules-with-inline-comments) section should note about recoverable errors. The directive comments cannot hide the recoverable errors.
- [Working with Custom Parsers](https://eslint.org/docs/developer-guide/working-with-custom-parsers) page should describe about `options.recoverableErrors` and `recoverableErrors` property of the returned value. Custom parsers can use `recoverableErrors` property instead of throwing fatal errors to report syntax errors.
- [Command Line Interface](https://eslint.org/docs/user-guide/command-line-interface) page should describe the newly `--verify-on-recoverable-parsing-errors` option.
- [Node.js API](https://eslint.org/docs/6.0.0/developer-guide/nodejs-api) page should describe the newly `verifyOnRecoverableParsingErrors` option.

## Drawbacks

- The value of this feature is relatively small because people can verify source code after they fixed syntax errors. This feature provides just efficient.
- This feature makes core rules more complex if we wanted to support widely situations.

## Backwards Compatibility Analysis

This is not a breaking change.

- `Linter` class will handle a new property.
- `espree` package will recognize `recoverableErrors` option and change that behavior if the `recoverableErrors` option was `true`.

If users are depending `parserOptions.recoverableErrors`, possibly it will be broken. But I believe that we don't need to be worried about the case.

## Alternatives

- `vue-eslint-parser`'s way is an alternative; A custom parser returns `errors` property along with AST and a plugin rule reports the errors in the property. But people can disable plugin rules, so it's not proper as the way that shows syntax errors.

## Open Questions

Nothing in particular.

## Frequently Asked Questions

Nothing in particular.

## Related Discussions

- https://github.com/eslint/eslint/issues/3815
- https://github.com/eslint/espree/issues/368
- https://github.com/eslint/eslint/pull/11509#pullrequestreview-220174854

[Recoverable Errors]: #recoverable-errors
