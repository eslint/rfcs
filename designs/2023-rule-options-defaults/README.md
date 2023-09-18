- Repo: eslint/eslint
- Start Date: 2023-09-18
- RFC PR: https://github.com/eslint/rfcs/pull/113
- Authors: [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Support for including options defaults in rules

## Summary

Enable rule authors to describe default values for rule options in a formal schema.

## Motivation

Right now, there's no programmatic way to determine what an ESLint rule's default options are.
This has resulted in a proliferation of per-repository or per-rule strategies for normalizing options with defaults.

- Some rules define functions with names like `normalizeOptions` ([`array-bracket-newline`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/array-bracket-newline.js#L102), [`object-curly-newline`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/object-curly-newline.js#L99)) or `parseOptions` ([`no-implicit-coercion`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/no-implicit-coercion.js#L22), [`use-before-define`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/no-use-before-define.js#L20)) that apply various, subtly different value defaults to the provided `context.options`.
- Some rules define individual option purely with inline logic ([`accessor-pairs`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/accessor-pairs.js#L177-L180), [`n/file-extension-in-import`](https://github.com/eslint-community/eslint-plugin-n/blob/150b34fa60287b088fc51cf754ff716e4862883c/lib/rules/file-extension-in-import.js#L67-L68)) or inline with a helper ([`array-bracket-spacing`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/array-bracket-spacing.js#L65-L74))
- `@typescript-eslint/eslint-plugin` has its own [`applyDefault`](https://github.com/typescript-eslint/typescript-eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/packages/utils/src/eslint-utils/applyDefault.ts#L10) used in a wrapping [`createRule`](https://github.com/typescript-eslint/typescript-eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/packages/utils/src/eslint-utils/RuleCreator.ts#L84)

In addition to increasing development complexity for rules, the lack of a standard options parsing strategy means users don't have a consistent or programmatic way to determine rule defaults.
The only user-facing method to determine a rule's default options is to read the documentation, or if it either doesn't exist or doesn't describe default values, read the rule's source.
Even if a plugin does have documentation, there is no guaranteed consistency in phrasing.

For example:

- [`array-callback-return`](https://eslint.org/docs/latest/rules/array-callback-return#options) phrases its options as _`"<key>": <value> (default) <explanation>`_
- [`accessor-pairs`](https://eslint.org/docs/latest/rules/accessor-pairs#options) phrases its options as _`<key> <explanation> (Default <default>)`_

This RFC proposes allowing rules to opt into declaring the default values of their options.
Doing so would:

- Streamline the process of creating and maintaining rules that take in options
- Standardize how both developers and end-users can reason about default rule options
- Allow tooling such as [`eslint-doc-generator`](https://github.com/bmish/eslint-doc-generator) to describe rule options programmatically - and therefore more consistently

This RFC proposes defaulting options passed to rules:

- Schemas that include `default` values will cause the `context.options` provided to rules to use that `default` if a value is `undefined`
- Options will always recursively default to at least an empty `{}`, to make behavior of nested object definitions with or without `default` properties consist
- The original, non-defaulted options object will be available as `context.optionsRaw` to reduce code churn in rules that currently rely on original (raw) values

Doing so would be a breaking change as it impacts what values are passed to the `create` method of rules.

## Detailed Design

This RFC proposes using the already-existing optional `default` field for each property in a rule's `meta.schema` when resolving what options values are provided to rules.
For each config value defined in `meta.schema`:

- If `default` is defined and the rule config's value is `undefined`, the value of `default` is used instead
- If the config value contains `type: "object"`, each property's `default` will recursively go through this same defaulting logic
  - If a rule config has not provided a value for the object, a new empty object will be created

Objects created by this defaulting logic should always be shallow cloned (`{ ...original }`) before being passed to rules.

The original `options` provided to rules will be available on a new `context.optionsRaw` property.
That property will be equivalent to what current versions of ESLint provide as `context.options`.

### Example: Primitive Value Default

Given a rule schema that defines a default value:

```json5
{
  default: "implicit",
  type: "string",
}
```

The proposed change would impact only cases when the provided value is `undefined`:

| Rule Config             | Current `context.options` | Proposed `context.options` |
| ----------------------- | ------------------------- | -------------------------- |
| `"error"`               | `[]`                      | `["implicit"]`             |
| `["error"]`             | `[]`                      | `["implicit"]`             |
| `["error", undefined]`  | `[undefined]`             | `["implicit"]`             |
| `["error", null]`       | `[null]`                  | `[null]`                   |
| `["error", "explicit"]` | `["explicit"]`            | `["explicit"]`             |

### Example: Nested Object Defaults

Using [`class-methods-use-this`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/class-methods-use-this.js)'s schema as an example:

```json5
[
  {
    type: "object",
    properties: {
      exceptMethods: {
        type: "array",
        items: {
          type: "string",
        },
      },
      enforceForClassFields: {
        type: "boolean",
        default: true,
      },
    },
    additionalProperties: false,
  },
]
```

- The rule's options parsing logic will no longer need to manually default to `{}`:

  ```diff
  - const config = Object.assign({}, context.options[0]);
  + const config = context.options[0];
  ```

- Options default checks, such as `!==` against `default: true` properties, will no longer be necessary:

  ```diff
  - const enforceForClassFields = config.enforceForClassFields !== false;
  + const enforceForClassFields = config.enforceForClassFields;
  ```

- Options whose provided values don't provide a default don't change behavior:

  ```diff
  const exceptMethods = new Set(config.exceptMethods || []);
  ```

The proposed behavior would impact cases when the provided value is `undefined` or is an object without an explicit value for `enforceForClassFields`:

| Rule Config                                   | Current `context.options`            | Proposed `context.options`             |
| --------------------------------------------- | ------------------------------------ | -------------------------------------- |
| `"error"`                                     | `[]`                                 | `[ { enforceForClassFields: true } ]`  |
| `["error"]`                                   | `[]`                                 | `[ { enforceForClassFields: true } ]`  |
| `["error", undefined]`                        | `[undefined]`                        | `[ { enforceForClassFields: true } ]`  |
| `["error", null]`                             | `[null]`                             | `[null]`                               |
| `["error", {}]`                               | `[{}]`                               | `[ { enforceForClassFields: true } ]`  |
| `["error", { enforceForClassFields: false }]` | `[{ enforceForClassFields: false }]` | `[ { enforceForClassFields: false } ]` |
| `["error", { enforceForClassFields: true }]`  | `[{ enforceForClassFields: true }]`  | `[ { enforceForClassFields: true } ]`  |

### Implementation

Currently, ESLint rule options are passed through a [`getRuleOptions` function](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/linter/linter.js#L740) whose only processing is to remove severity.
That function could be augmented to also take in the `rule.meta.schema`.
Its "worker" logic could recursively parse the rule config and schema:

1. If the rule config is `null`, return that directly
2. If the schema is an array:
   1. Recurse on the range of elements from 0 to the smaller of the rule config and schema's length
   2. If there are remaining rule config values (so, there were more rule configs than schema descriptions), add them directly to the resolved options
   3. If there are remaining schema (so, there were more schema descriptions than rule configs), fill in any options values for schemas containining either `default` or `type: "object"`
3. Create an options object that is a shallow clone of the first non-`undefined` from `ruleConfig` or `schema.default`, or `{}` if the schema contains `type: "object"`
4. If the options is truthy and schema is an object:
   1. Recursively apply defaulting logic for each of the schema's `properties`

Draft PR: [[Reference] feat!: factor in schema defaults for rule options](https://github.com/eslint/eslint/pull/17580)

### Out of Scope

#### Default `anyOf` Values

JSON schemas can use `anyOf` to indicate values whose types are analogous to TypeScript union types.
Resolving which `anyOf` constituent a runtime config aligns to can be complex.
It is out of scope for this RFC to investigate heuristics for that problem.

#### TypeScript Types

Right now, there are two community strategies for declaring rule options types, both with drawbacks:

- [`@types/eslint`](https://npmjs.com/package/@types/eslint) (the community-authored package describing ESLint's types for TypeScript users) [declares `RuleContext`'s `options` as `any[]`](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/ca231578e79ab1c2f01a308615e60a432061c97a/types/eslint/index.d.ts#L703), indicating it can be an array of any value types.
  - Drawback: `any` is generally considered a bad practice for TypeScript users as it provides no type descriptions or safety checks.
- [`@typescript-eslint/utils` > `RuleCreator`](https://typescript-eslint.io/developers/custom-rules#rulecreator) (the recommended TypeScript-enabled rule development utility) declares a [generic `RuleContext<TMessageIds, TOptions>` type with an `options: TOptions` property](https://github.com/typescript-eslint/typescript-eslint/blob/72149e1fe8f57fd217dfc59295a51df68187aad4/packages/utils/src/ts-eslint/Rule.ts#L175).
  - Drawback: `RuleContext` already doesn't factor in default options to the type system (https://github.com/typescript-eslint/typescript-eslint/issues/5439)

Resolving the type system type resulting from a fixed JSON schema -with or without `default`- and a user-provided object can be complex.
It is out of scope for this RFC to investigate heuristics for that problem.

## Documentation

We'll want to augment the following docs generators in:

- The core ESLint docs: automating and therefore standardizing how rule docs pages phrase default values
- typescript-eslint's docs: the same as the core ESLint docs
- [`eslint-doc-generator`](https://github.com/bmish/eslint-doc-generator)

We'll also want to mention this in the ESLint core custom rule documentation.

## Drawbacks

- This increases both conceptual and implementation complexity around rule options
- As a breaking change, this may break user rules - especially private rules we don't have visibility to

## Backwards Compatibility Analysis

This proposal contains two breaking changes:

- If a schema declares `default` values, rule options that were `undefined` will become those defaults instead
- Options will recursively default to `{}` based on schemas even if the schema has no `default` properties

Those breaking changes will impact rules that rely on the exact length or existing of properties in `context.options`.
Rules that wish to preserve the legacy options behavior can switch to referring to the new `context.optionsRaw` property.

In the draft PR containing this change, only 3 test files -rule tests for `indent`, `max-len`, and `no-multiple-empty-lines`- failed during `npm run test`.
All were fixed with a direct find-and-replace from `context.options` to `context.optionsRaw` in those rules' implementations.

## Alternatives

### Non-Recursive Object Defaults

An alternate, lower-impact change would be to _not_ options to `{}` if there is no `default` in its schema.
However:

- Defaulting to `{}` allows rules logic to not need to `|| {}` or `?? {}` as much in source code.
- Consider the case of an options schema declaring a `default` for a child property inside an object whose parent does not have a `default` introduces complexity:
  - If the defaulting logic didn't recursively create default objects, it could be confusing why the `default` value wasn't being used.
  - If the defaulting logic were to recursively create default objects only when a descendent property contains `default`, the inconsistency could also be confusing to users (and would be more code to implement in ESLint).

This RFC proposes always defaulting objects to `{}` for consistency.

### Prior Art

The only standardized options application I could find was `@typescript-eslint/eslint-plugin`'s `defaultOptions` property on rules.
It's visible in the plugin's [Custom Rules guide](https://typescript-eslint.io/developers/custom-rules#rulecreator), which suggests users wrap rule creation with a `createRule`:

```ts
import { ESLintUtils } from "@typescript-eslint/utils";

const createRule = ESLintUtils.RuleCreator(
  (name) => `https://example.com/rule/${name}`
);

export const rule = createRule({
  create(context) {
    /* ... */
  },
  defaultOptions: [
    /* ... */
  ],
  meta: {
    /* ... */
  },
  name: "...",
});
```

`createRule` applies a _deep_ merge rather than a shallow `Object.assign`: https://github.com/typescript-eslint/typescript-eslint/blob/324c4d31e46cbc95e8eb67d71792de9f49e65606/packages/utils/src/eslint-utils/applyDefault.ts.
This has the benefit of allowing user-provided options to "merge" into complex objects, rather than requiring rewriting larger objects.

However, deep merges are a more nuanced behavior, and may surprise users who assume the more common shallow merging or a different strategy in edge cases such as arrays.
This proposal goes with a less opinionated _shallow_ merge.

## Open Questions

1. Does the developer benefit of nested object defaulting work well enough to justify the added complexity?
2. Would we want to eventually deprecate/delete `context.optionsRaw`? Alternately, should we avoid it altogether, in favor of rewriting the options logic in the core rules?
3. Should ESLint eventually write its own schema parsing package?
4. Should figuring out TypeScript types block this proposal?

## Help Needed

I expect to implement this change.

I plan to file issues on popular community plugins suggesting using it and offering to implement.

## Frequently Asked Questions

### Does this change how we write rules?

Not significantly.
It should be a little less code to handle optional options.

### How does this impact typescript-eslint's `RuleCreator` and `defaultOptions`?

We can roll out this change over the current and next two major versions of typescript-eslint:

1. Current major version (v6): no code changes; mention ESLint's new options defaulting in documentation
2. Next major version (v7): mark `defaultOptions` as deprecated
3. Subsequent major version (v8): remove `defaultOptions`

This RFC should ideally at least have participation from typescript-eslint maintainers besides its author.

Note that [typescript-eslint's Building Custom Rules documentation](https://typescript-eslint.io/developers/custom-rules) only shows an empty `defaultOptions` in code snippets.
It never explains the presence of `defaultOptions` or suggests using it.

## Related Discussions

- <https://github.com/eslint/eslint/issues/17448>: the issue triggering this RFC
- <https://github.com/typescript-eslint/typescript-eslint/pull/6963>: typescript-eslint forking types for `JSONSchemaV4`
