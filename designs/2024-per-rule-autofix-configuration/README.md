- Repo: <https://github.com/eslint/eslint>
- Start Date: 2024-10-22
- RFC PR: https://github.com/eslint/rfcs/pull/134
- Authors: 
  - [Samuel Therrien](https://github.com/Samuel-Therrien-Beslogic) (aka [@Avasam](https://github.com/Avasam))
  - [Maxim Morev](https://github.com/MorevM)

# Per-rule autofix configuration

## Summary

<!-- One-paragraph explanation of the feature. -->

This RFC proposes adding support for controlling autofix behavior on a per-rule basis through shareable configuration.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Some rules support autofixing, which is often convenient, but in certain cases 
the fixes may be broken, unsafe, or simply undesirable. \
Ideally, unsafe autofixes should be treated as suggestions, and broken fixes should be reported. 

However, ESLint is a large ecosystem, and some useful plugins are no longer actively maintained. 
Even in actively maintained projects, users may want to disable autofixing for specific rules
due to project-specific or personal preferences.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

### Abstract

We introduce an *extended rule configuration format*:

```ts
type RuleEntry = Partial<{
  /**
   * The rule severity.
   * 
   * @default 2
   */
  severity: 0 | 1 | 2 | 'off' | 'warn' | 'error';

  /**
   * Array of options passed to the rule.
   * 
   * @default []
   */
  options: any[]

  /**
   * Whether to allow a rule to perform autofixes.
   * 
   * @default true
   */
  autofix: boolean;
}>;
```

This format enables explicit control over a rule's autofix behavior,
and opens the door to further extension with other meta-properties
([example](https://github.com/eslint/eslint/issues/19342)).

> [!IMPORTANT]
> This is an additional configuration format; existing configurations remain valid and do not need to be rewritten.

#### Autofix details

* The `autofix` option defaults to `true`, maintaining current behavior 
  (if a rule provides a fixer, it will run when `--fix` is used);
* If autofix is disabled for a rule, its fix becomes a suggestion labeled 
  "Apply disabled autofix" and can be applied manually through editor hints;
* If the rule does not define a fixer, the `autofix` key has no effect.

#### Additional extensions

We extend `linterOptions.reportUnusedDisableDirectives` 
and `linterOptions.reportUnusedInlineConfigs`(?) to support new extended format:

```js
export default [
  {
    linterOptions: {
      reportUnusedDisableDirectives: {
        severity: 'error',
        autofix: false,
      },
    },
  },
];
```

We extend CLI engine to support new format as an additional option to existing ones:

```bash
npx eslint --fix --rule 'no-var: { autofix: false }' 
```

We extend directive parser engine to support new format as well:

```js
/* eslint eqeqeq: "off", curly: { severity: "warn", autofix: false } */
```

### Example

```js
export default [
  {
    rules: {
      // Full rule entry with disabled autofix
      'eqeqeq': {
        severity: 'error',
        options: ['always'],
        autofix: false,
      },
      // Options can be omitted if not needed
      'no-var': {
        severity: 'error',
        autofix: false,
      },
      // Severity can be omitted if not needed
      'camelcase': {
        options: [{ properties: 'never' }],
        autofix: false,
      },
      // Autofix can be omitted if not needed
      'yoda': {
        options: ['never'],
      },
      // Old format remains valid and will be normalized internally
      'no-regex-spaces': 'error',
      'no-unneeded-ternary': ['error', {
        defaultAssignment: false,
      }]
    },
  },
];
```

> [!NOTE]
> In practice, users only need the full form when disabling autofix 
> or when being explicitly descriptive.

<details>
  <summary>Backward compatibility: existing configurations</summary>
  
  ```js
  export default [
    {
      linterOptions: {
        reportUnusedDisableDirectives: 'error',
      },
      rules: {
        'eqeqeq': 'error',
        'no-var': ['warn', 'always'],
        '@scope/some-rule': ['warn', 'always', { option: true }],
      },
    },
  ];
  ```

  Will be normalized as:

  ```js
  export default [
    {
      linterOptions: {
        reportUnusedDisableDirectives: {
          severity: 'error',
          autofix: true,
        },
      },
      rules: {
        'eqeqeq': {
          severity: 'error',
          options: [],
          autofix: true,
        },
        'no-var': {
          severity: 'warn',
          options: ['always'],
          autofix: true,
        },
        'some-plugin/some-rule': {
          severity: 'warn',
          options: ['always', { option: true }],
          autofix: true,
        },
      },
    },
  ];
  ```
</details>

### Implementation plan

#### 1. Normalize rule entries and configuration objects to the extended format

> [!NOTE]
> This is actually not fully related to the problem of this RFC, just a step to implementation and future extensibility.

As a first step, we normalize rule entries and `linterOptions.reportUnusedDisableDirectives` to the new extended format.

Relevant starting points:

* [@eslint/core > types.ts > LinterOptionsConfig](https://github.com/eslint/rewrite/blob/2020d38f9dbaabb9923b8f2116e7bcbafa530c85/packages/core/src/types.ts#L659)
* [@eslint/core > types.ts > RuleConfig](https://github.com/eslint/rewrite/blob/2020d38f9dbaabb9923b8f2116e7bcbafa530c85/packages/core/src/types.ts#L679)
* [types/index.d.ts > RuleEntry](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/types/index.d.ts#L1377)
* [types/index.d.ts > LinterOptions](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/types/index.d.ts#L1876)
* ---
* [@eslint/plugin-kit > isSeverityValid](https://github.com/eslint/rewrite/blob/88525272b717efea7cae1d5ea2429d6343a7e066/packages/plugin-kit/src/config-comment-parser.js#L30-L37)
* [flat-config-schema.js > rulesSchema](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/config/flat-config-schema.js#L450)
* [flat-config-schema.js > disableDirectiveSeveritySchema](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/config/flat-config-schema.js#L298)
* [flat-config-schema.js > normalizeRuleOptions](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/config/flat-config-schema.js#L132)
* [config.js > #normalizeRulesConfig](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/config/config.js#L515)
* [config.js > validateRulesConfig](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/config/config.js#L553)
* [linter/apply-disable-directives.js](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/linter/apply-disable-directives.js#L474) should take a full form of the entry.

Some of static methods / individual functions also need to be considered, 
like [config.js > getRuleNumericSeverity](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/config/config.js#L634)
or [linter.js > getRuleOptions](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/linter/linter.js#L1086).

There are likely additional functions that operate on non-normalized rule entries; tests should help identify them. \
There's also an open question here regarding support for the legacy config format, as the current code relies on `@eslint/eslintrc` in some places.

Next, the CLI engine and directive parser should be verified to support object-form rule entries.

---

Next, all consumers of normalized rule entries and `reportUnusedDisableDirectives` must be updated accordingly.

For directives, updating [apply-disable-directives.js](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/linter/apply-disable-directives.js#L466) may be sufficient.

For rules usage is more widespread, there are a lot of usage like [this](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/linter/linter.js#L699), [here](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/linter/linter.js#L2140) and [there](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/linter/linter.js#L1086). \
I'm not sure it's possible to identify them all and that it's necessary to list them here, hopefully the tests will help.

Once normalization is in place and consistent, we can safely introduce autofix disabling logic.

#### 2. Disabling autofixes

##### Disabling for rules

I think the earliest place where we can account for disabled autofix is [here](https://github.com/eslint/eslint/blob/52f5b7a0af48a2f143f0bccfd4e036025b08280d/lib/linter/linter.js#L1249), 
when creating the `ReportTranslator`. \
We have the context of the rule and its extended configuration, 
and we can pass that into `createReportTranslator` as a separate boolean property
(since the current implementation turns off suggestions when `disableFixes: false`, 
but we need them if the fixer is disabled). 

Inside `ReportTranslator` we can also convert the fix into a suggestion.

##### Disabling for directives

I think all the work in this regard can be done in the 
[linter/apply-disabled-directives.js](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/linter/apply-disable-directives.js#L466) 
assuming the full entry of `reportUnusedDisableDirectives` configuration is available.

It's unclear whether suggestions are currently supported for directives - please advise if they are, and where that logic resides.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

### Autofix on rules

I think it is necessary to demonstrate the use of the full form of the record 
in the [Configuration Files > Configuring rules](https://eslint.org/docs/latest/use/configure/configuration-files#configuring-rules) section,
that way we show all the available options literally on one screen.

I think there should be an explanatory section in the [Configure rules](https://eslint.org/docs/latest/use/configure/rules) section, 
and maybe it should be the first one on the page to show all the available options at once so that the user is not intimidated 
when they see multiple options on the same screen at the same time like:

```js
/* eslint eqeqeq: "off", curly: { severity: "error", autofix: false } */
```

I also believe that there should be a separate "Disabling autofix" section,
probably located before the [Disabling rules](https://eslint.org/docs/latest/use/configure/rules#disabling-rules) 
section, so that it can be directly referenced.

### `linterOptions.reportUnusedDisableDirectives`

A note should be added in the
[Configuration Files > Reporting Unused Disable Directives](https://eslint.org/docs/latest/use/configure/configuration-files#reporting-unused-disable-directives)
section that autofix behavior can be configured using the extended format.

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

The primary drawback is the complexity of normalizing configuration formats 
across the codebase, which requires careful implementation across multiple packages (?).

It's important to ensure nothing breaks during this transition.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This feature is fully backwards compatible for end users:

* `autofix` defaults to `true`, matching current behavior; 
* Internal normalization does not affect user configurations or existing workflows.

However, this can be a breaking change for authors of solutions on top of ESLint
(e.g. a wrapper function for configuration(s), since the new config format is just JS),
or those that rely on ESLint's public API (like [`eslint.calculateConfigForFile()`](https://eslint.org/docs/latest/integrate/nodejs-api#-eslintcalculateconfigforfilefilepath))

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

### Use of a 3rd-party plugin

<https://www.npmjs.com/package/eslint-plugin-no-autofix> is a tool that exists to currently work around this limitation of ESLint, but it is not perfect.

1. It is an extra third-party dependency, with its own potential maintenance issues (having to keep up with ESLint, separate dependencies that can fall out of date, obsolete, unsecure, etc.)
2. It may not work in all environments. For example, pre-commit.ci: <https://github.com/aladdin-add/eslint-plugin/issues/98>
3. It may not work correctly with all third-party rules: <https://github.com/eslint-community/eslint-plugin-eslint-comments/issues/234>

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

- **Should there be legacy format support?** \
  I'd say no, but there's a call to [getRuleSeverity](https://github.com/eslint/eslint/blob/0f49329b4a7f91714f2cd1e9ce532d32202c47f4/lib/eslint/eslint.js#L20) 
  coming [from @eslint/eslintc](https://github.com/eslint/eslintrc/blob/556e80029f01d07758ab1f5801bc9421bca4b072/lib/shared/config-ops.js#L30) repeatedly in the code, 
  and it would obviously stop working if a new record form was passed in.

- **Should `linterOptions.reportUnusedInlineConfigs` support configurable autofix?**

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

While I am confident in implementing this RFC, I would greatly appreciate help 
reviewing the structure and phrasing of this document 
to ensure clarity and correctness in English.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

Currently, most key concepts are covered above. \
Additional FAQs can be added as needed during review.

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
- Original issue: <https://github.com/eslint/eslint/issues/18696>
- Previous RFC: <https://github.com/eslint/rfcs/pull/125>
- <https://github.com/aladdin-add/eslint-plugin/issues/98>
- <https://github.com/eslint-community/eslint-plugin-eslint-comments/issues/234>
- <https://github.com/eslint-community/eslint-plugin-eslint-comments/issues/249#issuecomment-2605557271>
