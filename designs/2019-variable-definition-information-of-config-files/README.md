- Start Date: 2019-03-04
- RFC PR: (leave this empty, to be filled in later)
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

Both `variable.eslintExplicitGlobalComments` and `variable.eslintImplicitGlobalSetting` are instantiated even if regular variable declarations (E.g. `let foo` syntax) exist. Therefore, [no-redeclare] rule can verify redeclaration by a mix of regular variable declarations, `/*globals*/` comments, and config files.

This will be implemented into [`addDeclaredGlobals(globalScope, configGlobals, commentDirectives)`][lib/linter.js#L73-L131] function.

<table><td>
<a id="question1">❓</a> <b>Open Question</b>:<br>
(<a href="question1a">link</a>) Should we handle de-declaration such as <code>/*globals foo:off*/</code>? Because variable objects don't exist for de-declared variables, rules cannot know de-declared variables under this proposal.
</td></table>

## Documentation

All of the existing `variable.eslintExplicitGlobal`, `variable.eslintExplicitGlobalComment`, and `variable.writeable` are undocumented. So new properties also can be undocumented.

<table><td>
<a id="question2">❓</a> <b>Open Question</b>:<br>
(<a href="question2a">link</a>) Should we document those properties as public API officially?
</td></table>

If we decided to make those properties public, it will be in [context.getScope()
](https://eslint.org/docs/developer-guide/working-with-rules#contextgetscope) section so plugins can use those properties.

## Drawbacks

Rules get accessible to `globals` setting values. It might restrict us to change the system which declares global variables.

## Backwards Compatibility Analysis

This proposal doesn't change the existing things.

If a plugin rule used `variable.eslintExplicitGlobalComments` property or `variable.eslintImplicitGlobalSetting` property for some reason, the rule will be broken. But it's a pretty little possibility.

## Alternatives

- Adding `context.getImplicitGlobalSettings()` method to `RuleContext` object to retrieve normalized `globals` setting. In this case, rules can know settings in config files even if `/*globals foo:off*/` comment existed. On the other hand, rules have to search and parse `/*globals*/` comments manually although ESLint core has handled those once.

## Open Questions

- (<a href="#question1" id="question1a">link</a>) Should we handle de-declaration such as `/*globals foo:off*/`? Because variable objects don't exist for de-declared variables, rules cannot know de-declared variables under this proposal.<br>
  [@mysticatea]'s preference is no. The [no-redeclare] rule will ignore such a removed declaration because it's a corner case and ECMAScript doesn't have the analogy of that. Probably [eslint-plugin-eslint-comments] plugin can report such a de-declaration.

- (<a href="#question2" id="question2a">link</a>) Should we document those properties as public API officially although all of the existing `variable.eslintExplicitGlobal`, `variable.eslintExplicitGlobalComment`, and `variable.writeable` are undocumented?<br>
  [@mysticatea]'s preference is yes.

## Frequently Asked Questions

Nothing in particular yet.

## Related Discussions

- [eslint/eslint#11370]


[@mysticatea]: https://github.com/mysticatea
[eslint/eslint#11370]: https://github.com/eslint/eslint/issues/11370
[lib/linter.js#L73-L131]: https://github.com/eslint/eslint/blob/b00a5e9d8dc6c5f77eb0e4e0c58dfaf12a771d7b/lib/linter.js#L73-L131
[no-redeclare]: https://eslint.org/docs/rules/no-redeclare
[eslint-plugin-eslint-comments]: https://github.com/mysticatea/eslint-plugin-eslint-comments
