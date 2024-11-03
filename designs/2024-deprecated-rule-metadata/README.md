- Repo: eslint/eslint
- Start Date: 2024-02-20
- RFC PR: <https://github.com/eslint/rfcs/pull/124>
- Authors: [bmish](https://github.com/bmish), [DMartens](https://github.com/DMartens)

# Support additional metadata for rule deprecations

## Summary

<!-- One-paragraph explanation of the feature. -->

This RFC suggests a format for storing additional information in rule metadata about rule deprecations and replacement rules, allowing tooling (e.g. documentation generators) to generate more informative deprecation notices.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

There are long-time [rule meta properties](https://eslint.org/docs/latest/extend/custom-rules#rule-structure) `meta.deprecated` and `meta.replacedBy` that have been intended to document when rules are deprecated and what their replacement rule(s) are. For the most part, usage would look something like this:

```js
module.exports = { meta: { deprecated: true, replacedBy: ['replacement-rule-name'] } };
```

These properties are often used for generating plugin/rule documentation websites and in documentation tooling like [eslint-doc-generator](https://github.com/bmish/eslint-doc-generator).

But there are some limitations to this current format:

- Simply providing the replacement rule name as a string doesn't yield much context/explanation of the replacement/deprecation. That means documentation tooling / websites and code editors can only generate limited information to present about the situation.
- Some rules provide the replacement rule name with the plugin prefix as in `prefix/rule-name` while others just provide it as `rule-name`, which can result in ambiguity about whether the replacement rule is in the same plugin, a different third-party plugin, or ESLint core. And for third-party plugins, there's no easy way to automatically determine where their documentation is located or how to link to them.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

We propose to extend `meta.deprecated` rule property schemas to reduce ambiguity and allow additional key details to be represented.
Plugin developers are free to add additional properties (e.g. a link to the original GitHub PR).

```ts
type RuleMeta = {
  deprecated?:
    | boolean // Existing boolean option, backwards compatible.
    | DeprecatedInfo // Proposed extension

  /** @deprecated */
  replacedBy?: string[] // Deprecate the top-level property and "move" into the "deprecated" object
};

/* At least one property is required */
type DeprecatedInfo = {
  message?: string // General message presented to the user, e.g. for the key rule why the rule is deprecated or for info how to replace the rule
  url?: string     // URL to more information about this deprecation in general
  replacedBy?: ReplacedByInfo[] // An empty array explicitly states that there is no replacement
  deprecatedSince?: Version // Helps users gauge when to migrate and useful for documentation
  availableUntil?: Version | null // The estimated version when the rule is removed (probably the next major version). null means the rule is "frozen" (will be available but will not be changed)
}

/* At least one property is required */
type ReplacedByInfo = {
  message?: string // General message presented to the user, e.g. how to replace the rule
  url?: string     // URL to more information about this replacement in general
  plugin?: Specifier // name should be "eslint" if the replacemenet is an ESLint core rule. Omit the property if the replacement is in the same plugin
  rule?: Specifier
}

type Specifier = {
  name?: string // Name of the plugin / rule / ...
  url?: string  // URL to documentation for the plugin / rule
}

/* Version string of the package containing the rule (without a leading v and using the full semver, e.g. 8.53.0) */
type Version = string
```

The `meta.replacedBy` property is moved into the `meta.deprecated` property as `meta.replacedBy` requires `meta.deprecated` to be set.
The reason for this is that a rule logically must be marked as deprecated to be replaced by another rule which it currently can be.

### Example
Real-world example of how this could be used based on the situation in <https://github.com/eslint/eslint/issues/18053>:

```js
// lib/rules/semi.js
module.exports = {
  meta: {
    deprecated: {
      message: 'Stylistic rules are being moved out of ESLint core.',
      url: 'https://eslint.org/blog/2023/10/deprecating-formatting-rules/',
      replacedBy: [
        {
          plugin: {
            name: '@stylistic/eslint-plugin-js',
            url: 'https://eslint.style/packages/js',
          },
          rule: {
            name: 'semi',
            url: 'https://eslint.style/rules/js/semi'
          }
        }
      ],
      deprecatedSince: "8.53.0",
    },
  },
};
```


This data could be used by documentation websites and tooling like [eslint-doc-generator](https://github.com/bmish/eslint-doc-generator) to generate notices and links like:

> semi (deprecated) \
> Replaced by [semi](https://eslint.style/rules/js/semi) from [@stylistic/js](https://eslint.style/). \
> Use the `foo` option on the new rule to achieve the same behavior as before. [Read more](https://example.com/how-to-migrate-to-the-new-semi-rule). \
> Stylistic rules are being moved out of ESLint core. [Read more](https://eslint.org/blog/2023/10/deprecating-formatting-rules/).

We can also support the same `meta.deprecated` and `meta.replacedBy` properties on configurations and processors (the other kinds of objects exported by ESLint plugins), replacing `rule` with `config` or `processor` as needed. This would be part of the effort to standardize documentation properties in <https://github.com/eslint/eslint/issues/17842>.

### Changes
In terms of actual changes inside ESLint needed for this:

- Mention the new schema in the [custom rule documentation](https://eslint.org/docs/latest/extend/custom-rules#rule-structure)
- Add any additional information to these properties in core rules as desired (such as in <https://github.com/eslint/eslint/issues/18053>)
- Update ESLint's website generator to take into account the additional information for rule doc deprecation notices
- Update [LintResult.usedDeprecatedRules](https://github.com/eslint/eslint/blob/0f5df509a4bc00cff2c62b90fab184bdf0231322/lib/eslint/eslint.js#L197-L211) by normalizing the old and new format for the existing `replacedBy` property and adding a new property with the name `info` for rules using the new deprecated format
- Update the [types](https://github.com/eslint/eslint/blob/35a8858d62cb050fa0b56702e55c94ffaaf6956d/lib/types/index.d.ts#L745) in eslint

External changes:

- Update the [types](https://github.com/typescript-eslint/typescript-eslint/blob/82cb9dd580f62644ed988fd2bf27f519177a60bd/packages/utils/src/ts-eslint/Rule.ts#L70) in @typescript-eslint/eslint
- Update eslint-doc-generator to handle the new information: <https://github.com/bmish/eslint-doc-generator/issues/512>
- Update the metadata for the most common plugins
- Consider implementing an [eslint-plugin-eslint-plugin](https://github.com/eslint-community/eslint-plugin-eslint-plugin) rule to encourage more complete deprecation information to be stored in these properties

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

We don't necessarily need a formal announcement for this. The aforementioned changes to the rule documentation page and types should be sufficient.

However, this update could be covered in a blog post about general rule documentation best practices, if anyone ever has an interest in writing something like that.

## Drawbacks

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think
    about any opposing viewpoints that reviewers might bring up.

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

There are some limited [backwards compatibility](#backwards-compatibility-analysis) concerns for third-party tooling.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

Existing rules will continue to be backwards-compatible with the new format.

Changing the format of these properties mainly affects third-party documentation tooling and websites that use this information, and not ESLint users nor ESLint plugins directly.

For the most part, the new `meta.deprecated` format should be backwards-compatible, as code is often written to check simply for a truthy value inside of `meta.deprecated`, e.g. `if (rule.meta.deprecated) { /* ... */ }`, which will continue to work as expected. The code needs to be updated if:
- it checks specifically for the boolean `true` value in `meta.deprecated`
- it checks for whether the rule is deprecated by checking for a non-empty `meta.replacedBy`
- retrieves rule names from `meta.replacedBy`

Overall, a limited number of third-party tools that might be affected, and these should be trivial to fix when impacts are discovered.

We do not need to consider this to be a breaking change in terms of [ESLint's semantic versioning policy](https://github.com/eslint/eslint#semantic-versioning-policy).

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

### Do nothing

This would leave the current `meta.deprecated` and `meta.replacedBy` properties as they are, which would continue to be ambiguous and limited in the information they can provide.

### Create a new property

Create a new property, e.g. `meta.deprecation`,

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

1. Is there additional deprecation information we'd like to represent? Note that additional information can always be added later, but it's good to consider any possible needs now.
2. Should `meta.deprecated.plugin.name` accommodate different package registries (e.g. [jsr](https://jsr.io/) with `jsr:eslint-plugin-example`)
  - If it is a concern, plugin developers can use a direct URL
3. Which "extension points" (rules, processors, configurations, parsers, formatters) shold be supported?
  - Focus on rules as other extension points have no well-defined use cases

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I should be able to handle the minimal changes needed in ESLint core, and can kick off some of the changes needed in community projects.

## Frequently Asked Questions

- Why not provide a property to describe how to migrate to the replacement rule which requires an option to be set?
  - The options of the replacement rule could change and it is unlikely that a deprecated rules gets updated to accommodate the change

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- [The original RFC](https://github.com/eslint/rfcs/pull/116)
- [The issue triggering this RFC](https://github.com/eslint/eslint/issues/18061)
- Inspirations
  - <https://github.com/jsx-eslint/eslint-plugin-react/pull/3469#discussion_r1002316631>
  - <https://github.com/eslint/eslint/issues/5774#issuecomment-220640368>
- Related
  - <https://github.com/eslint/eslint/issues/17842>
  - <https://github.com/eslint/eslint/issues/18053>
