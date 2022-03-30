- Repo: eslint/eslint and @eslint/eslintrc
- Start Date: 2021-12-12
- RFC PR: <https://github.com/eslint/rfcs/pull/85>
- Authors: [bmish](https://github.com/bmish)

# Require schemas and object-style rules

## Summary

<!-- One-paragraph explanation of the feature. -->

We propose two changes to rules:

- Requiring rules with options to specify schemas (removing support for rules with options that are missing schemas)
- Requiring rule implementations to use the object-style format (removing support for the deprecated function-style format)

Note that while these two proposals are included in the same RFC so that they can be considered holistically, they do not have to be tied together, and can be split into separate RFCs if desired.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

### Motivation for requiring schemas

Today, rule schemas ([meta.schema](https://eslint.org/docs/developer-guide/working-with-rules#rule-basics)) are optional. This has the following effects:

- If a rule does not specify a schema, nothing is enforced.
- Rules with options may not have a schema specified, despite it being best practice to specify a schema for rule options. This increases the chances of developers passing invalid rule options and encountering unexpected behavior / bugs in rules.
- If a rule author wants to enforce that no options are passed to their rule, they have to manually specify `schema: []`, otherwise it's possible that developers could accidentally provide useless rule options to their rule without knowing it.

So we will change the default rule schema to `schema: []` to enforce that no rule options are passed to rules that do not specify a schema.

This has the desirable consequence of requiring rules with options to begin specifying their schema if they did not already.

Making ESLint's default behavior more strict like this goes along with many recent changes to tighten validation (i.e. more [RuleTester class validation in ESLint 7](https://eslint.org/docs/user-guide/migrating-to-7.0.0#rule-tester-strict), [increased rule configuration validation in ESLint 6](https://eslint.org/docs/user-guide/migrating-to-6.0.0#rule-config-validating), etc) to improve rule quality and usability and reduce the chance of mistakes.

And in addition to the obvious benefits of increased rule schema usage such as reduced chance of silent mistakes/bugs, being able to rely on having rule schemas available could unlock new features/improvements:

- Improved/auto-generated documentation regarding rule options
- TypeScript-style auto-complete/code-completion when configuring a rule

There is an existing, third-party lint rule [eslint-plugin/require-meta-schema](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin/blob/master/docs/rules/require-meta-schema.md) to enforce that rules have schemas.

### Motivation for requiring object-style rules

Supporting two rule formats requires increased complexity and maintenance burden inside ESLint, as well as in third-party tools like [eslint-plugin-eslint-plugin](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin) that need to be able to handle both rule formats.

Object-style rules were introduced in [ESLint v2](https://eslint.org/blog/2016/02/eslint-v2.0.0-released) and function-style rules were deprecated in early 2016. Over six years has elapsed since then, and we don't want to support both formats forever.

Object-style rules are clearly superior in terms of configuration thanks to the `meta` object they include. Function-style rules are also blocked from accessing valuable features like [autofixing](https://eslint.org/docs/8.0.0/user-guide/migrating-to-8.0.0#rules-require-metafixable-to-provide-fixes) and suggestions.

There is an existing, third-party lint rule [eslint-plugin/prefer-object-rule](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin/blob/master/docs/rules/prefer-object-rule.md) to enforce and autofix that rules are written with the object-style.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

The two proposals in this RFC should be implemented separately.

### Detailed design for requiring schemas

In `getRuleOptionsSchema()` in `eslint/lib/shared/config-validator.js` and in `@eslint/eslintrc/lib/shared/config-validator.js`, we will change the default schema from `null` to:

```json
{
    "type": "array",
    "minItems": "0",
    "maxItems": "0"
}
```

This will resulting in the following error message when running a test case that includes `options` for a rule without a schema:

```pt
Error: rule-tester:
Configuration for rule "my-rule" is invalid:
Value [{"foo-option":"foo-value"}] should NOT have more than 0 items.
```

or when passing options to a rule without a schema and executing linting with the rule:

```pt
Error: .eslintrc.js:
Configuration for rule "my-rule" is invalid:
Value [{"foo-option":"foo-value"}] should NOT have more than 0 items.
```

We'll then add tests in `eslint/tests/lib/shared/config-validator.js` and `@eslint/eslintrc/tests/lib/cascading-config-array-factory.js` that this error is correctly thrown.

#### Opt-out

In (hopefully) rare situations (e.g. bare-bones internal/private rules), we will allow users to opt-out from specifying a schema using `schema: false`. This clearly indicates that the user has chosen to forgo a schema. And a lint rule called `eslint-plugin/no-schema-opt-out` or `eslint-plugin/no-schema-false` could be created to discourage overuse/abuse of this opt-out.

We believe this explicit opt-out is preferred over users resorting to hacky workarounds such as providing no-op/fake/minimal/incomplete schemas like `schema: {}` or `schema: { type: "array" }`. No-op schemas like these can confuse both humans (who might mistake it with `schema: []` which is not a no-op) and automated analysis/tooling built around schemas.

#### No-op schemas

In the rule tester, we will throw an exception when encountering `schema: {}` (and possibly other no-op schemas in the future):

```pt
`schema: {}` is a no-op. For rules with options, please fill in a complete schema. For rules without options, please omit `schema` or use `schema: []`.
```

Optionally, we can begin warning about this issue in the rule tester even sooner as a non-breaking change.

### Detailed design for requiring object-style rules

Code changes:

- Remove `normalizeRule()` from `lib/linter/rules.js` which normalized function-style rules to object-style
- Remove handling of the function-style rule properties `rule.schema` and `rule.deprecated` in `createCoreRuleConfigs()` in `lib/init/config-rule.js`
- Remove handling of the function-style rule property `rule.schema` in `getRuleOptionsSchema()` in `lib/config/rule-validator.js` and `lib/shared/config-validator.js`

Test changes:

- We'll change the test in `tests/lib/linter/rules.js` for function-style rules to ensure that function-style rules are NOT detected

Some of these changes need to be replicated in `@eslint/eslintrc` as well.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

### Documentation for requiring schemas

The [Rule Basics](https://eslint.org/docs/developer-guide/working-with-rules#rule-basics) section of the `Working with Rules` documentation page will be updated to mention when the `schema` property is required. We will also update any other documentation showcasing rules with options to include the `schema` property.

### Documentation for requiring object-style rules

We can finally delete `docs/developer-guide/working-with-rules-deprecated.md`, along with updating/removing any other documentation or test cases that use function-style rules.

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

### Drawbacks of requiring schemas

Plugin authors who left out schemas will need to take the time to add them (typically a quick, easy task).

Users of old/unmaintained plugins/rules which are missing schemas will not be able to use them until they are updated.

### Drawbacks of requiring object-style rules

Plugin authors who still have function-style rules will need to trivially convert them to object-style rules.

Users of old/unmaintained plugins/rules which still have function-style rules will not be able to use them until they are updated.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This RFC describes a set of breaking changes that can be released in the next major version of ESLint:

- Both ESLint plugin authors and ESLint users will be affected and will be unable to use offending rules until they are updated. This will mostly affect a small number of older ESLint plugins that haven't been updated nor maintained in years.
- ESLint users who have rules configured incorrectly will need to fix their configurations when these rules add schemas.

I did an analysis of the ESLint plugin ecosystem by writing a tool called [eslint-ecosystem-analyzer](https://github.com/bmish/eslint-ecosystem-analyzer) to find out how much of the ecosystem will be affected by these changes.

For the top 100 ESLint plugins, 1% of total rules have options but are missing a schema (or said another way, 5% of rules with options are missing a schema). The percentages are only slightly higher (worse) when the top 1,000 ESLint plugins are considered (the top 1,000 results include many repositories that haven't been maintained nor used in years).

For the top 100 ESLint plugins, 94% of rules are object rules. The percentage is only slightly lower (worse) when the top 1,000 ESLint plugins are considered.

These results indicate that the vast majority of the ecosystem is already ready for these changes and will not be affected.

While the current numbers are already supportive of making these changes, we have a number of means available to further prepare plugins for these changes, which we could optionally employ:

- Communicating about these upcoming changes in blog posts leading up to the major release
- Adding deprecation notices inside ESLint in a minor version release, showing a warning when running tests for an offending rule (these will change to fatal assertions in pre-release versions of ESLint v9):
  > DEPRECATION WARNING: This test case specifies `options` but the rule is missing `meta.schema` and will stop working in ESLint v9.
  > Please add `meta.schema`: <https://eslint.org/docs/developer-guide/working-with-rules#options-schemas>
  > This lint rule can assist with the conversion: <https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin/blob/main/docs/rules/require-meta-schema.md>
  >
  > DEPRECATION WARNING: This rule is using the deprecated function-style format and will stop working in ESLint v9.
  > Please convert to object-style rules: <https://eslint.org/docs/developer-guide/working-with-rules>
  > This lint rule can assist with the conversion: <https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin/blob/main/docs/rules/prefer-object-rule.md>
- Directly notifying or helping out any top plugins that still need to adopt these changes ([eslint-ecosystem-analyzer](https://github.com/bmish/eslint-ecosystem-analyzer) can help identify the top candidates to reach out to)
- Encouraging further adoption of [eslint-plugin-eslint-plugin](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin) which has two rules that can detect and even autofix offending rules

Full results of the analysis below:

```pt
╔══════════════════════════╤═════════════════════════╤═══════════════════════════════════════════════╤════════════════════════════════════════════════╤══════════════════════════╗
║ Metric                   │ Value (Top 100 Plugins) │ Value (Top 1000 Plugins, Updated Last 1 Year) │ Value (Top 1000 Plugins, Updated Last 2 Years) │ Value (Top 1000 Plugins) ║
╟──────────────────────────┼─────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────╢
║ Plugins Found            │ 100                     │ 639                                           │ 777                                            │ 1000                     ║
╟──────────────────────────┼─────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────╢
║ Plugins With Rules Found │ 79                      │ 446                                           │ 533                                            │ 677                      ║
╟──────────────────────────┼─────────────────────────┼───────────────────────────────────────────────┼────────────────────────────────────────────────┼──────────────────────────╢
║ Average Rules Per Plugin │ 22.09                   │ 7.96                                          │ 7.23                                           │ 6.27                     ║
╚══════════════════════════╧═════════════════════════╧═══════════════════════════════════════════════╧════════════════════════════════════════════════╧══════════════════════════╝

╔══════════════════════════════════════════════════════════════════╤═════════════════════╤═══════════════════════════════════════════╤════════════════════════════════════════════╤══════════════════════╗
║ Metric                                                           │ % (Top 100 Plugins) │ % (Top 1000 Plugins, Updated Last 1 Year) │ % (Top 1000 Plugins, Updated Last 2 Years) │ % (Top 1000 Plugins) ║
╟──────────────────────────────────────────────────────────────────┼─────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────╢
║ Rules With Options                                               │ 27.16               │ 27.84                                     │ 26.77                                      │ 26.46                ║
╟──────────────────────────────────────────────────────────────────┼─────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────╢
║ Rules With Options But Missing Schema, Out of Total Rules        │ 1.49                │ 1.91                                      │ 1.95                                       │ 2.26                 ║
╟──────────────────────────────────────────────────────────────────┼─────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────╢
║ Rules With Options But Missing Schema, Out of Rules With Options │ 5.49                │ 6.88                                      │ 7.27                                       │ 8.55                 ║
╚══════════════════════════════════════════════════════════════════╧═════════════════════╧═══════════════════════════════════════════╧════════════════════════════════════════════╧══════════════════════╝

╔═══════════════╤═════════════════════╤═══════════════════════════════════════════╤════════════════════════════════════════════╤══════════════════════╗
║ Rule Type     │ % (Top 100 Plugins) │ % (Top 1000 Plugins, Updated Last 1 Year) │ % (Top 1000 Plugins, Updated Last 2 Years) │ % (Top 1000 Plugins) ║
╟───────────────┼─────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────╢
║ Object Rule   │ 93.81               │ 91.41                                     │ 89.74                                      │ 86.80                ║
╟───────────────┼─────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────╢
║ Function Rule │ 4.01                │ 6.14                                      │ 7.84                                       │ 10.91                ║
╟───────────────┼─────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────╢
║ Unknown       │ 2.18                │ 2.45                                      │ 2.41                                       │ 2.29                 ║
╚═══════════════╧═════════════════════╧═══════════════════════════════════════════╧════════════════════════════════════════════╧══════════════════════╝
```

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

### Alternatives to requiring schemas

We could continue to allow rules that are missing schemas. But this would prevent us from raising rule quality / testing standards and could slow down improvements or features that depend on schemas being present.

### Alternatives to requiring object-style rules

We could continue to allow function-style rules. But it's undesirable to have to support two rule formats forever.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I will implement this change.

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

- [Original issue proposing that schemas should be required](https://github.com/eslint/eslint/issues/14709)
