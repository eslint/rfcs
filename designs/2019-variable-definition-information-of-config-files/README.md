- Start Date: 2019-03-04
- RFC PR: https://github.com/eslint/rfcs/pull/17
- Authors: Toru Nagashima ([@mysticatea])

# Variable Definition Information of Config Files

## Summary

This proposal adds variable definition information of config files to `Variable` objects.

## Motivation

To make rules being able to report matters about variable definitions related to config files.

Especially, this proposal minds [no-redeclare] rule to report redeclaration by a mix of regular variable declarations, `/*globals*/` comments, and config files. ([eslint/eslint#11370])

## Detailed Design

Currently, `Variable` objects have the following properties to express `/*globals*/` definitions.

- `variable.eslintExplicitGlobal` (`boolean`) ... `true` if this variable was defined by `/*globals*/` comment.
- `variable.eslintExplicitGlobalComment` (`Comment` object) ... The `/*globals*/` comment that defines this variable. If two or more comments defined this variable, it adopts the last one.
- `variable.writeable` (`boolean`) ... `true` if the final setting is `"writable"`.

This proposal adds the following properties.

- `variable.eslintExplicitGlobalComments` (an array of `Comment` objects) ... All `/*globals*/` comments that define this variable.
- `variable.eslintImplicitGlobalSetting` (`"readonly"` | `"writable"` | `null`) ... The setting in config files. This is `null` if config files didn't define it.

And this proposal removes existing `variable.eslintExplicitGlobalComment` property in favor of new `variable.eslintExplicitGlobalComments` property.

Both `variable.eslintExplicitGlobalComments` and `variable.eslintImplicitGlobalSetting` are instantiated even if regular variable declarations (E.g. `let foo` syntax) exist. Therefore, [no-redeclare] rule can verify redeclaration by a mix of regular variable declarations, `/*globals*/` comments, and config files.

This will be implemented into [`addDeclaredGlobals(globalScope, configGlobals, commentDirectives)`][lib/linter.js#L73-L131] function.

## Documentation

This proposal is related to making rules. So [Working with Rules](https://eslint.org/docs/developer-guide/working-with-rules) page should describe those properties, including undocumented existing properties.

- `variable.writeable`
- `variable.eslintExplicitGlobal`
- `variable.eslintExplicitGlobalComments`
- `variable.eslintImplicitGlobalSetting`

And, we should make an item in the migration guide. One undocumented property is removed.

- `variable.eslintExplicitGlobalComment`

## Drawbacks

Rules get accessible to `globals` setting values. It might restrict us to change the system which declares global variables.

## Backwards Compatibility Analysis

If a plugin rule used `variable.eslintExplicitGlobalComment` property, because this proposal removes it, the rule will be broken. But that property was undocumented and [GitHub search](https://github.com/search?p=1&q=eslintExplicitGlobalComment+language%3Ajavascript&type=Code) shows only copies of `no-unused-vars` rule or ESLint core, so this removal is no danger. (One point, `eslint-plugin-closure` package has [the type definition of escope](https://github.com/google/eslint-closure/blob/bf5c0d4d2a67ea3e8394c228717ae23d1a1ae4ba/packages/eslint-plugin-closure/lib/externs/escope.js#L163). That will need to be tweaked regardless of the removal.)

If a plugin rule used `variable.eslintExplicitGlobalComments` property or `variable.eslintImplicitGlobalSetting` property for some reason, the rule will be broken. But it's a pretty little possibility.

## Alternatives

- Adding `context.getImplicitGlobalSettings()` method to `RuleContext` object to retrieve normalized `globals` setting. In this case, rules can know settings in config files even if `/*globals foo:off*/` comment existed. On the other hand, rules have to search and parse `/*globals*/` comments manually although ESLint core has handled those once.

## Related Discussions

- [eslint/eslint#11370]


[@mysticatea]: https://github.com/mysticatea
[eslint/eslint#11370]: https://github.com/eslint/eslint/issues/11370
[lib/linter.js#L73-L131]: https://github.com/eslint/eslint/blob/b00a5e9d8dc6c5f77eb0e4e0c58dfaf12a771d7b/lib/linter.js#L73-L131
[no-redeclare]: https://eslint.org/docs/rules/no-redeclare
[eslint-plugin-eslint-comments]: https://github.com/mysticatea/eslint-plugin-eslint-comments
