- Repo: eslint/eslint
- Start Date: 2021-05-10
- RFC PR: (leave this empty, to be filled in later)
- Authors: Nicholas C. Zakas

# Strict package exports

## Summary

Node.js 12 introduced the ability to strictly define [package exports](https://nodejs.org/api/packages.html#packages_exports) that ensure internal package modules cannot be accessed externally. This RFC proposes implementing package exports for the `eslint` package to prevent unexpected use of internal modules.

## Motivation

Since ESLint was first introduced, developers have attempted to build plugins and other tools on top of the `eslint` package, and in doing so, accessed internal modules. Despite documenting that only APIs published on our [Node.js API page](https://eslint.org/docs/developer-guide/nodejs-api) are officially supported and other APIs may change or disappear without notice, we continue to receive complaints when we change or remove an internal-only module.

By eliminating access to internal modules, we can prevent this situation from happening in the future and be able to move forward with changes to our internal structure without fear of breaking existing tools that are built on top of undocumented ESLint APIs.

## Detailed Design

This design is made up of four steps:

1. Add an `exports` field to `package.json`
1. Remove the `CLIEngine` class
1. Update the `ESLint` class
1. Remove the `linter` object

These steps will lock down the external API for the `eslint` package.

### An an `exports` field to `package.json`

Currently, ESLint defines [`api.js`](https://github.com/eslint/eslint/blob/master/lib/api.js), which is intended to define the public-facing API of the `eslint` module. This design assumes we keep `api.js` as the source of truth for the public-facing API and just add an `exports` field in `package.json` that points to `api.js`:

```json
{
    "name": "eslint",
    "main": "./lib/api.js",
    "exports": "./lib/api.js"
}
```

For most users, this change will have no effect. For some users who are relying on undocumented APIs, this may be a breaking change. (Discussion on the nature of the breaking changes is documented in the **Backwards Compatibility Analysis** section below.)

### Remove the `CLIEngine` class

The `CLIEngine` class is the old public interface for ESLint functionality that was superseded by the `ESLint` class. When `ESLint` was introduced last year, `CLIEngine` was deprecated and it was announced that it would be removed at a later date. Because we are already going to be breaking the public interface of the `eslint` module, it makes sense to formalize our API going forward by eliminating a redundant class.

### Update the `ESLint` class

In order to facilitate removing `CLIEngine`, we need to implement a replacement for the `CLIEngine#getRules()` method, which was [missing](https://github.com/eslint/eslint/issues/13454#issuecomment-653362104) when the `ESLint` class was introduced. The `CLIEngine#getRules()` method itself produced unexpected results frequently because it's return value was based on the previous `CLIEngine#executeOnFiles()` or `CLIEngine#executeOnText()` calls, meaning it could return different results depending on when it was called.

Instead of duplicating how `CLIEngine#getRules()` works, we will add a new `ESLint#getRulesForReport(report)` method that accepts a single argument, the report object returned from `ESLint#lintFiles()` or `ESLint#lintText()`, and return a map of ruleIds to rule meta data for every rule represented in the report.

### Remove the `linter` object

The `linter` object was deprecated in favor of the `Linter` class but was never removed from the public API. As with `CLIEngine`, this is a good time to remove deprecated APIs, so we should remove this as well.

### Remaining public API

The remaining public API after all of these changes are:

* `Linter`
* `ESLint`
* `RuleTester`
* `SourceCode`

Nothing outside of these classes will be accessible from outside of the `eslint` package.

## Documentation

Most of the documentation changes will take place on the [Node.js API page](https://eslint.org/docs/developer-guide/nodejs-api). We will also need to announce this change in a blog post ahead of time to give developers a heads-up.

## Drawbacks

The biggest drawbacks for this proposal are the potential to break an unknown number of existing packages that are based on undocumented APIs and those still relying on `CLIEngine`.

## Backwards Compatibility Analysis

The primary concern with regards to backwards compatibility is to ensure that there is either another way to accomplish the same thing as with the undocumented/deprecated APIs or the use case is something that should never have been supported in the first place. After some analysis, here are the top most concerning uses and how we can address them.

### `CLIEngine#getRules()`

The most import consumer of the `CLIEngine#getRules()` method is the [VS Code ESLint extension](https://github.com/microsoft/vscode-eslint), which is one of the most popular extensions for the editor. Currently, the extension [uses `CLIEngine#getRules()`](https://github.com/microsoft/vscode-eslint/blob/e4b2738e713b7523824e0c72166f5cdd44f47052/server/src/eslintServer.ts#L1395) after running ESLint on a file in order to display additional information about the rule. In this case, it should be easy to switch to the new `ESLint#getRulesForResult()` method to maintain current functionality.

### `linter` Object

Any utility using the deprecated `linter` object can update their code to use the `Linter` class:

```js
// before
const linter = require("eslint").linter;

// after
const Linter = require("eslint").Linter;
const linter = new Linter();
```

### Access to rules

One of the more common cases of accessing undocumented API is when plugins access core ESLint rules using `require()`, such as `require("eslint/lib/rules/eqeqeq")`. This appears to be fairly common for any plugins intended to work with custom ESLint parsers, such as:

* [eslint-plugin-vue](https://github.com/vuejs/eslint-plugin-vue/blob/62f577dcfcb859c24c6e0d4615ad880f5e1d4688/lib/utils/index.js#L120)
* [typescript-eslint](https://github.com/typescript-eslint/typescript-eslint/blob/25ea953cc60b118bd385c71e0a9b61c286c26fcf/packages/eslint-plugin/src/rules/no-loss-of-precision.ts#L7)

The most common case is to use the existing rule as a base upon which to create a modified rule for the specific parser. This is a use case we never intended to support, and maintainers have acknowledged that seemingly small changes to core rules can introduce breaking changes to their derived rules. 

Going forward, these `require()` calls will no longer work. The recommended way for plugin to adapt to this change is to copy the rules they are interested in into their own repository where they will have completely control over their functionality and can always re-sync with the ESLint repository on their own schedule.


## Alternatives

The primary alternative is to "bless" some of the currently undocumented APIs as public, document them, and support them going forward. While this would reduce the risk of disrupting the ecosystem, it also adds more maintenance burden to the ESLint project as a whole, making it more difficult to make significant changes going forward.

## Open Questions

1. Do we need both `main` and `exports`? Or can we just do `exports`?
1. Is there another use case of `CLIEngine#getRules()` that hasn't been discussed?

## Help Needed

n/a

## Frequently Asked Questions

TBD

## Related Discussions

https://github.com/eslint/eslint/issues/13654
