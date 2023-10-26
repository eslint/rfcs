- Repo: eslint/eslint
- Start Date: 2023-09-18
- RFC PR: https://github.com/eslint/rfcs/pull/113
- Authors: [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Support for `meta.defaultOptions` on rules

## Summary

Enable rules to provide default options programmatically with a new `meta.defaultOptions` property.

## Motivation

Right now, most popular ESLint rules do not adhere to a single programmatic way to determine their default options.
This has resulted in a proliferation of per-repository or per-rule strategies for normalizing options with defaults.

- Some rules define functions with names like `normalizeOptions` ([`array-bracket-newline`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/array-bracket-newline.js#L102), [`object-curly-newline`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/object-curly-newline.js#L99)) or `parseOptions` ([`no-implicit-coercion`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/no-implicit-coercion.js#L22), [`use-before-define`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/no-use-before-define.js#L20)) that apply various, subtly different value defaults to the provided `context.options`.
- Some rules define individual option purely with inline logic ([`accessor-pairs`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/accessor-pairs.js#L177-L180), [`n/file-extension-in-import`](https://github.com/eslint-community/eslint-plugin-n/blob/150b34fa60287b088fc51cf754ff716e4862883c/lib/rules/file-extension-in-import.js#L67-L68)) or inline with a helper ([`array-bracket-spacing`](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/rules/array-bracket-spacing.js#L65-L74))
- `@typescript-eslint/eslint-plugin` has its own [`applyDefault`](https://github.com/typescript-eslint/typescript-eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/packages/utils/src/eslint-utils/applyDefault.ts#L10) used in a wrapping [`createRule`](https://github.com/typescript-eslint/typescript-eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/packages/utils/src/eslint-utils/RuleCreator.ts#L84)

Although the currently used Ajv package does provide a [`useDefaults` option](https://ajv.js.org/guide/modifying-data.html#assigning-defaults), rule developers typically have not used it.
Ajv does not recursively fill in objects for rule defaults - an ergonomics issue for rule authors.

In addition to increasing development complexity for rules, the lack of ergonomic options defaulting means end-users haven't had a consistent or programmatic way to determine rule defaults.
The only user-facing method to determine a rule's default options is to read the documentation, or if it either doesn't exist or doesn't describe default values, read the rule's source.
Even if a plugin does have documentation, there is no guaranteed consistency in phrasing.

For example:

- [`array-callback-return`](https://eslint.org/docs/latest/rules/array-callback-return#options) phrases its options as _`"<key>": <value> (default) <explanation>`_
- [`accessor-pairs`](https://eslint.org/docs/latest/rules/accessor-pairs#options) phrases its options as _`<key> <explanation> (Default <default>)`_

This RFC proposes adding a `defaultOptions` property to rules' `meta`.
Doing so would:

- Streamline the process of creating and maintaining rules that take in options
- Standardize how both developers and end-users can reason about default rule options
- Encourage writing rules that allow tooling such as [`eslint-doc-generator`](https://github.com/bmish/eslint-doc-generator) to describe rule options programmatically - and therefore more consistently

Options provided to rules would be the result of a deep (recursive) clone and merge of the rule's default options and user-provided options.

## Detailed Design

Currently, ESLint rule options are passed through a [`getRuleOptions` function](https://github.com/eslint/eslint/blob/da09f4e641141f585ef611c6e9d63d4331054706/lib/linter/linter.js#L740) whose only processing is to remove severity.
That function could be augmented to also take in the `rule.meta.defaultOptions`, if it exists.

Defaulting logic could go with the following logic to approximate user intent:

1. If the user-provided value is `undefined`, go with the default option
2. If the user-provided value is an object, a new `{}` object is created with values recursively merged
3. Arrays and non-`undefined` literals -including `null`- are used as-is

See [[Reference] feat: add meta.defaultOptions](https://github.com/eslint/eslint/pull/17656) > [`function deepMerge`](https://github.com/eslint/eslint/pull/17656/files#diff-9bbcc1ca9625b99554a055ac028190e5bd28432207a7f1a519690c5d0484287fR24) and [`describe("deepMerge")`](https://github.com/eslint/eslint/pull/17656/files#diff-fa70d7d4c142a6c3e727cf7280f981cb80a1db10bd6cc85c205e47e47da05cfeR25) for the proposed defaulting logic.

### Rules with Differing Options Behavior

[[Reference] feat: add meta.defaultOptions](https://github.com/eslint/eslint/pull/17656) implements `meta.defaultOptions` on rules that include options, are not deprecated, and match any of:

- Are mentioned earlier in this RFC
- Have a name starting with `A-C` and include the search string `context.options`
- Include the search string `Object.assign(`
- Experience test failures when Ajv's `useDefaults` is option is disabled (see [Schema Defaults](#schema-defaults))

<details>
<summary>
The following matched rules will not use <code>meta.defaultOptions</code> because their current defaulting logic is incompatible with the new merging logic:
</summary>

- `array-bracket-newline`:
  - Presumed `defaultOptions: [{ minItems: null, multiline: true }]`
  - When given `options: [{ minItems: null }]`:
    - Current normalized options: `[ { minItems: null } ]`, with implicit `multiline: false`
    - Updated normalized options: `[ { minItems: null, multiline: true } ]`
- `object-curly-newline`:
  - Presumed `defaultOptions: [{ consistent: true }]`
  - When given `options: [{ multiline: true, minProperties: 2 }]`:
    - Current normalized `options.ObjectExpression`: `{ multiline: true, minProperties: 2, consistent: false`
    - Updated normalized `options.ObjectExpression`: `{ multiline: true, minProperties: 2, consistent: true`
- `operator-linebreak`:
  - Presumed `defaultOptions: ["after", { "overrides": { "?": "before", ":": "before" } }]`
  - When given `options: ["none"]`: - Current normalized `styleOverrides`: `{}` - Updated normalized `styleOverrides`: `{ '?': 'before', ':': 'before' }`

</details>

<details>
<summary>
Additionally, the following rule has default logic that doesn't map well to <code>meta.defaultOptions</code>.
</summary>

- `strict`: The mode depends on -and can be overridden based on- `ecmaFeatures.impliedStrict`

</details>

### Schema Defaults

Ajv provides its own support for `default` values with a [`useDefaults` option](https://ajv.js.org/guide/modifying-data.html#assigning-defaults).
It's used in ESLint core already, in [`lib/shared/ajv.js`](https://github.com/eslint/eslint/blob/22a558228ff98f478fa308c9ecde361acc4caf20/lib/shared/ajv.js#L21).
Some core ESLint rules describe `default:` values in that schema.

Having two places to describe `default` values can be confusing for developers.
It can be unclear where a value comes from when its default may be specified in two separate locations.
This RFC proposes removing the `useDefaults: true` setting in a future major version of ESLint.

### Out of Scope

#### Preserving Original Options

The original `options` provided to rules could be made available on a new property with a name like `context.optionsRaw`.
That property could exist solely as a legacy compatibility layer for rules that would want the behavioral and/or documentation benefits of `meta.defaultOptions` but have nonstandard options parsing logic.

However:

- The overlap of rules that would want the documentation benefits of `meta.defaultOptions` but would not be able to work with the new defaulting logic is quite small.
- Using non-standard defaulting logic is detrimental to users being able to reliably understand how rules' options generally work.

This RFC considers `context.optionsRaw` or an equivalent as not worth the code addition.

#### TypeScript Types

Right now, there are two community strategies for declaring rule options types, both with drawbacks:

- [`@types/eslint`](https://npmjs.com/package/@types/eslint) (the community-authored package describing ESLint's types for TypeScript users) [declares `RuleContext`'s `options` as `any[]`](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/ca231578e79ab1c2f01a308615e60a432061c97a/types/eslint/index.d.ts#L703), indicating it can be an array of any value types.
  - Drawback: `any` is generally considered a bad practice for TypeScript users as it provides no type descriptions or safety checks.
- [`@typescript-eslint/utils` > `RuleCreator`](https://typescript-eslint.io/developers/custom-rules#rulecreator) (the recommended TypeScript-enabled rule development utility) declares a [generic `RuleContext<TMessageIds, TOptions>` type with an `options: TOptions` property](https://github.com/typescript-eslint/typescript-eslint/blob/72149e1fe8f57fd217dfc59295a51df68187aad4/packages/utils/src/ts-eslint/Rule.ts#L175).
  - Drawback: `RuleContext` doesn't factor in root-level `defaultOptions` to the type system (https://github.com/typescript-eslint/typescript-eslint/issues/5439)

This change is limited to the ESLint core repository, which doesn't provide TypeScript types.
It is out of scope for this RFC to apply `RuleCreator`-style types to `@types/node`, remove `default` from typescript-eslint's schema types, and/or resolve typescript-eslint's factoring default options issue.

## Documentation

We'll want to augment the following docs generators in:

- The core ESLint docs: automating and therefore standardizing how rule docs pages phrase default values
- typescript-eslint's docs: the same as the core ESLint docs
- [`eslint-doc-generator`](https://github.com/bmish/eslint-doc-generator)

We'll also want to mention this in the ESLint core custom rule documentation.

## Drawbacks

- This increases both conceptual and implementation complexity around built-in rule options logic
  - This RFC believes that enough rules were already implemented with ad-hoc logic to make standardizing rule options logic a net reduction of complexity
- This explicitly moves away from the JSON Schema / ajv style description of putting `default` values in `meta.schema`
  - This RFC believes that schema-oriented defaults are too convoluted to justify their usage

## Backwards Compatibility Analysis

If a rule does not specify `meta.defaultOptions`, its behavior will not change.
This change is therefore completely backwards compatible to start.

The removal of Ajv `useDefaults: true` will be a breaking change in a future major version.
This RFC believes the impact of that removal will be minor and not create significant user pain.
Most ESLint core rules did not change behavior when switched to `meta.defaultOptions`.

Popular third-party plugins will be notified of the upcoming change and helped move to `meta.defaultOptions`.
Given the improvements to rule implementations and documentation `eslint-doc-generator`, this RFC expects plugins to be generally receptive.

## Alternatives

This RFC originally proposed including recursive object defaults in the `meta.schema`.
See [[Reference] feat!: factor in schema defaults for rule options](https://github.com/eslint/eslint/pull/17580).

Doing so aligned with existing schema usage; however, its drawbacks were noted as increased user-land complexity for defining options:

<table>
<thead>
<tr>
<th>Approach</th>
<th>Advantages</th>
<th>Disadvantages</th>
</tr>
</thead>
<tbody>
<tr>
<th><code>meta.defaultOptions</code></th>
<td>
<ul>
<li>Easier to change and understand the defaults</li>
<li>Not tied to Ajv implementation / details</li>
</ul>
</td>
<td>
<ul>
<li>More complex deep merging behavior</li>
<li>Two locations for option descriptions</li>
</ul>
</td>
</tr>
<tr>
<th>Recursive Schema Defaults</th>
<td>
<ul>
<li>One idiomatic location for option descriptions</li>
<li>Aligns closer to common Ajv paradigms</li>
</ul>
</td>
<td>
<ul>
<li>More difficult to statically type</li>
<li>Complex schemas can be ambiguous</li>
</ul>
</td>
</tr>
</tbody>
</table>

This RFC believes that the simplicity of defining option defaults in `meta.defaultOptions` is the best outcome for rule authors and end-users.

As a data point, typescript-eslint's `defaultOptions` approach has been in wide use for several years:

1. `eslint-plugin-typescript` originally added its `defaultOptions` in [December 2018](https://github.com/bradzacher/eslint-plugin-typescript/pull/206)
2. `defaultOptions` were made the standard for typescript-eslint's rules in [February 2019](https://github.com/typescript-eslint/typescript-eslint/pull/120)
3. `defaultOptions` became the standard for rules written with typescript-eslint in [May 2019](https://github.com/typescript-eslint/typescript-eslint/pull/425)

The lack of user pushback against typescript-eslint's approach is considered evidence that its noted disadvantages are not severe enough to prevent using it.

### Non-Recursive Object Defaults

Sticking with Ajv's `useDefaults: true` would bring two benefits:

- Aligning ESLint schema defaulting with the standard logic used by Ajv
- Avoiding a custom defaulting implementation in ESLint by using Ajv's instead

However, in addition to the drawbacks mentioned in the alternatives table, [`useDefaults` requires explicit empty object for intermediate object entries to work for nested properties](https://github.com/ajv-validator/ajv/issues/1710).
That means:

- Schemas with optional objects containing properties with default need to provide an explicit `default: {}` for those objects.
- If those properties are themselves optional objects containing properties, both the outer and inner objects need explicit `default` values.

See [this Runkit with simulated Ajv objects](https://runkit.com/joshuakgoldberg/650a826da5839400082514ff) for examples of how rules would need to provide those default values.

### Prior Art

The only standardized options application found in the wild is `@typescript-eslint/eslint-plugin`'s `defaultOptions` property on rules.
It's visible in the plugin's [Custom Rules guide](https://typescript-eslint.io/developers/custom-rules#rulecreator), which suggests users wrap rule creation with a `RuleCreator`'s `createRule`:

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

`createRule` also applies a deep merge rather than a shallow `Object.assign`: https://github.com/typescript-eslint/typescript-eslint/blob/324c4d31e46cbc95e8eb67d71792de9f49e65606/packages/utils/src/eslint-utils/applyDefault.ts.
This has the benefit of allowing user-provided options to "merge" into complex objects, rather than requiring rewriting larger objects.
Most rules with nontrivial options ask for those options to be specified with at least one object.

This RFC proposes roughly the same merging strategy as `RuleCreator`.
This RFC's differences are:

- Moving `defaultOptions` inside of `meta`
- Not calling `JSON.parse(JSON.stringify(...))` on options to remove `undefined` values

Updating `RuleCreator` inside typescript-eslint is out of scope for this RFC.
The typescript-eslint team will take on that work.

## Open Questions

1. Should ESLint eventually write its own schema parsing package - i.e. formalizing its fork from ajv?
2. _"`RuleTester` should validate the default options against the rule options schema."_ was mentioned in the original issue prompting this RFC. `RuleTester` in the PoC validates the results of merging `meta.defaultOptions` with test-provided `options`. Is this enough?

## Help Needed

I expect to implement this change.

I plan to file issues on popular community plugins suggesting using it and offering to implement.

## Frequently Asked Questions

### Does this change how we write rules?

Not significantly.
It should be a little less code to handle optional options.

### How does this impact typescript-eslint's `RuleCreator` and `defaultOptions`?

This RFC includes participation from multiple typescript-eslint maintainers.
The exact rollout plan will be discussed on typescript-eslint's side.
We can roll out this change over the current and next 1-2 major versions of typescript-eslint.

Note that [typescript-eslint's Building Custom Rules documentation](https://typescript-eslint.io/developers/custom-rules) only shows an empty `defaultOptions` in code snippets.
It never explains the presence of `defaultOptions` or suggests using it.

## Related Discussions

- <https://github.com/eslint/eslint/issues/17448>: the issue triggering this RFC
- <https://github.com/typescript-eslint/typescript-eslint/pull/6963>: typescript-eslint forking types for `JSONSchemaV4`
