- Repo: eslint/eslint
- Start Date: 2024-02-20
- RFC PR: <https://github.com/eslint/rfcs/pull/116>
- Authors: [bmish](https://github.com/bmish)

# Support additional metadata for rule deprecations

## Summary

<!-- One-paragraph explanation of the feature. -->

This RFC suggests a format for storing additional information in rule metadata about rule deprecations and replacement rules, allowing automated documentation and website tooling to generate more informative deprecation notices.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

There are long-time [rule properties](https://eslint.org/docs/latest/extend/custom-rules#rule-structure) `meta.deprecated` and `meta.replacedBy` that have been intended to document when rules are deprecated and what their replacement rule(s) are. For the most part, usage would look something like this:

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

We propose extended `meta.deprecated` and `meta.replacedBy` rule property schemas to reduce ambiguity and allow additional key details to be represented, described below as a TypeScript type for clarity:

```ts
type RuleMeta = {
  deprecated?:
    | boolean // Existing boolean option, backwards compatible.
    | string // Shorthand property for general deprecation message, such as why the deprecation occurred. Any truthy value implies deprecated.
    | {
        message?: string; // General deprecation message, such as why the deprecation occurred.
        url?: string; // URL to more information about this deprecation in general.
      }
    | undefined;
  replacedBy?:
    | readonly string[] // Existing shorthand property for rule names, backwards compatible. It's recommended to omit the plugin prefix from rule names.
    | readonly {
        /**
         * Plugin containing the replacement.
         * Use "eslint" if the replacement is an ESLint core rule.
         * Omit if the replacement is in the same plugin.
         */
        plugin?:
          | string // Shorthand property for the plugin name i.e. "eslint-plugin-example" that contains the replacement rule.
          | {
              name?: string; // Plugin name i.e. "eslint-plugin-example" that contains the replacement rule.
              url?: string; // URL to plugin documentation.
            };
        rule?:
          | string // Shorthand property for replacement rule name (without plugin prefix).
          | {
              name?: string; // Replacement rule name (without plugin prefix).
              url?: string; // URL to rule documentation.
            };
        meta?: {
          message?: string; // Message about this specific replacement, such as how to use/configure the replacement rule to achieve the same results as the rule being replaced.
          url?: string; // URL to more information about this specific deprecation/replacement.
        };
      }[]
    | undefined;
};
```

Real-world example of how this could be used based on the situation in <https://github.com/eslint/eslint/issues/18053>:

```js
// lib/rules/semi.js
module.exports = {
  meta: {
    deprecated: {
      message: 'Stylistic rules are being moved out of ESLint core.',
      url: 'https://eslint.org/blog/2023/10/deprecating-formatting-rules/',
    },
    replacedBy: [
      {
        plugin: {
          name: '@stylistic/js',
          url: 'https://eslint.style/',
        },
        rule: {
          name: 'semi',
          url: 'https://eslint.style/rules/js/semi',
        },
        meta: {
          message: 'Use the `foo` option on the new rule to achieve the same behavior as before.',
          url: 'https://example.com/how-to-migrate-to-the-new-semi-rule',
        }
      },
    ],
  },
};
```

This data could be used by documentation websites and tooling like [eslint-doc-generator](https://github.com/bmish/eslint-doc-generator) to generate notices and links like:

> semi (deprecated) \
> Replaced by [semi](https://eslint.style/rules/js/semi) from [@stylistic/js](https://eslint.style/). \
> Use the `foo` option on the new rule to achieve the same behavior as before. [Read more](https://example.com/how-to-migrate-to-the-new-semi-rule). \
> Stylistic rules are being moved out of ESLint core. [Read more](https://eslint.org/blog/2023/10/deprecating-formatting-rules/).

We can also support the same `meta.deprecated` and `meta.replacedBy` properties on configurations and processors (the other kinds of objects exported by ESLint plugins), replacing `rule` with `config` or `processor` as needed. This would be part of the effort to standardize documentation properties in <https://github.com/eslint/eslint/issues/17842>.

In terms of actual changes inside ESLint needed for this:

- Mention the new schema in the [custom rule documentation](https://eslint.org/docs/latest/extend/custom-rules#rule-structure)
- Ensure these properties are allowed on configurations and processors
- Add any additional information to these properties in core rules as desired (such as in <https://github.com/eslint/eslint/issues/18053>, <https://github.com/eslint/eslint/pull/13274>)
- Update ESLint's website generator to take into account the additional information for rule doc deprecation notices

External changes:

- Update the [types](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/b77d83e019025017b06953907cb77f35e4231714/types/eslint/index.d.ts#L734) in @types/eslint
- Update the [types](https://github.com/typescript-eslint/typescript-eslint/blob/82cb9dd580f62644ed988fd2bf27f519177a60bd/packages/utils/src/ts-eslint/Rule.ts#L70) in @typescript-eslint/eslint
- Update eslint-doc-generator to handle the new information: <https://github.com/bmish/eslint-doc-generator/issues/512>
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

For the most part, the new `meta.deprecated` format should be backwards-compatible, as code is often written to check simply for a truthy value inside of `meta.deprecated`, e.g. `if (rule.meta.deprecated) { /* ... */ }`, which will continue to work as expected. If code checks specifically for the boolean `true` value in `meta.deprecated`, or retrieves rule names from `meta.replacedBy`, then it would need to be updated.

Overall, a limited number of third-party tools that might be affected, and these should be trivial to fix when impacts are discovered.

We do not need to consider this to be a breaking change in terms of [ESLint's semantic versioning policy](https://github.com/eslint/eslint#semantic-versioning-policy).

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

1. Do nothing. This would leave the current `meta.deprecated` and `meta.replacedBy` properties as they are, which would continue to be ambiguous and limited in the information they can provide.
2. Consolidate `meta.replacedBy` into `meta.deprecated.replacedBy`. This alternative was strongly considered. If we were to design things from scratch, we might choose this approach. However, there's not much tangible benefit to it besides organizational tidiness, and the downsides include:
   - It creates yet another migration burden for the plugin ecosystem (potentially hundreds of plugins) to migrate from the old property to the new property
   - There's overhead for us to go through the process of deprecating the old property and encouraging or helping plugins to move to the new one
   - Tooling would need to support both properties, perhaps indefinitely due to the long tail of rules that may never be updated

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

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I should be able to handle the minimal changes needed in ESLint core, and can kick off some of the changes needed in community projects.

## Frequently Asked Questions

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

The issue triggering this RFC:

- <https://github.com/eslint/eslint/issues/18061>

This proposal is inspired by:

- <https://github.com/jsx-eslint/eslint-plugin-react/pull/3469#discussion_r1002316631>
- <https://github.com/eslint/eslint/issues/5774#issuecomment-220640368>

Related:

- <https://github.com/eslint/eslint/issues/17842>
