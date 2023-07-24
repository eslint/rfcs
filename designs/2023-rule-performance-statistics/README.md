- Repo: eslint/eslint
- Start Date: 2023-03-10
- RFC PR: <https://github.com/eslint/rfcs/pull/108>
- Authors: Mara Nikola Kiefer ([@mnkiefer](https://github.com/mnkiefer))

# Performance Statistics

## Summary

<!-- One-paragraph explanation of the feature. -->

This document describes a new ESLint flag `stats`, which enables the user to obtain a series of runtime *statistics* on top of their final lint results (refer to the table below).

These statistics include more granular timing information, such as the parse-, fix-, and lint-times as well as the number of directives, fix passes, and violations encountered.

| Stats |  | Description |
|--------|-------|---|
| **times*** | | The times/time objects spent on (parsing, fixing, linting) a file. |
|        | fix* | An array of time objects for each of the fix passes. |
|        | parse | The time that is spent when parsing a file. |
|        | rules* | An array of time objects for each file/rule/selector combination. |
| **fixPasses** | | The number of times ESLint has applied at least one fix after linting. |
| **directives** | | The number of disable directives which have been applied to silence rule violations. |
| **violations** | | The number of times a rule has been violated. |

> All object properties marked with a `*` also contain a computed `total` property.

A *proof-of-concept* can be found at:
- [**ESLint**](https://github.com/mnkiefer/eslint/pull/1): **Fork of ESLint** on which the [detailed design](#detailed-design) (as described below) has been implemented.
- [**Sample project**](https://github.com/mnkiefer/eslint-samples): Project on which the above implementation can be demonstrated.

Below is an brief example on what the `stats` properties would look like (based on the sample project file _invalid/no-regex-spaces_).<br>
Each property will be addressed individually in the [Detailed Design](#detailed-design) section 2 ("Adding the `stats` option to the Linter").

```json
"stats": {
    "directives": 1,
    "fixPasses": 1,
    "times": {
        "parse": 0.129959,
        "total": 0.15254299999999998,
        "rules": {
            "total": 0.020625,
            "valid-typeof": {
                "total": 0.020625,
                "nodes": {
                    "UnaryExpression": 0.018208000000000002,
                    "Program": 0.002417
                }
            }
        },
        "fix": {
            "passes": [
                0.001959
            ],
            "total": 0.001959
        }
    },
    "violations": 0
}
```

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

An analysis of rule performance in ESLint can already be carried out by setting the [TIMING](https://eslint.org/docs/latest/extend/custom-rules#profile-rule-performance) environment variable. However, **additional scripting is still required** to collect/extract more *granular* times data (lint time per file per rule) by following one of the two approaches:

1. Running ESLint per rule, per file and then collecting the file/rule time data output:

    ```bash
    TIMING=1 DEBUG=eslint:cli-engine eslint --no-eslintrc --rule ... file
    ```

2. Doing a single ESLint run on all files and then extracting the file/rule time data output:

    ```bash
    TIMING=all DEBUG=eslint:cli-engine eslint ... files
    ```

In addition, one needs to create an overview for an effective presentation of the results.

Since ESLint already collects most of this data internally, it would be more *convenient and shareable* to have:

   1. The **times data** from ESLint exposed to the formatters.
   2. The possibility to also gain more insight into [other runtime statistics](https://github.com/eslint/eslint/issues/14597#issuecomment-1003863524) (not time related).

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

------

1. **Adding the new `stats` option to ESLint:**

 - [_docs/src/use/command-line-interface.md_](https://github.com/mnkiefer/eslint/pull/1/files#diff-80937556f9a8e68352718255c20699fce061fa72760504c6accc3fc6b3b0613aR117): First, we document its *name* and *purpose* for the [Command Line Interface Reference](https://eslint.org/docs/latest/use/command-line-interface) under the *Miscellaneous* category.
     ```md
     --stats     Add additional statistics to lint results
     ```

 - [_lib/options.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-358c9491edc00f0db6f2f3c317df9aa932135803481b86c9289bd56bf8af0622R375-R3809): Now that the option has been documented, we add the `stats` option to ESLint's CLI options of type (`ParsedCLIOptions`):
     ```js
     {
         option: "stats",
         type: "Boolean",
         default: "false",
         description: "Add statistics to the lint report"
     }
     ```

 - [_lib/cli.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-347ff93ed2b00c93c817863e32fbac5b4fac71d7339a48378980e682777689f4R96): The function `translateOptions(ParsedCLIOptions)` is called, so we add the `stats` option as an input parameter here too.

 - [_lib/eslint/eslint.js_](): The function `processOptions(ESLintOptions)` is called, which is responsible for validating and normalizing options for the *ESLint* CLIEngine instance. Here, we add the `stats` option as a parameter to:
    <br>&nbsp;&nbsp;&nbsp;&nbsp;<a name="eslintjs-step1">Step 1</a>: The `processOptions` function and in that function set its default to `false` and make sure that a given value is always a boolean.
    <br>&nbsp;&nbsp;&nbsp;&nbsp;<a name="eslintjs-step2">Step 2</a>: The `ESLintOptions` type definition

 - [_lib/eslint/eslint-helpers.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-87d53094b12d82e4c11a0e1167d79cf2f471d2f5e5ebb6fc483e891f9dc87a5a): Similar to as we have done in [_eslint.js_ step 1](#eslintjs-step1) but for the *FlatESLint* CLIEngine instance.

- [_lib/eslint/flat-eslint.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-03dd66bfc8332edc2b145936aa2dd607ace1c34a31c222ec4d9617481876c27a): Similar to as we have done in [_eslint.js_ step 2](#eslintjs-step2) but for `FlatESLintOptions`. We also add the `stats` option to the function `verifyText(Object)` which returns the `LintResult` object. So, if `stats=true`, the `stats` properties (collected by the Linter) must be appended to the lint result (see `getStats()` in the next section).
> Note, that we have also adjusted the `linter.verifyAndFix()` input and output, which will be explained in the next section regarding changes to the Linter.

------

1. **Adding the `stats` option to the Linter:**

    The following paragraphs describe the individual `stats` option properties, how they are collected and stored.<br>
    We will start by the exposure of the `times` object, as most of the information is already collected by the Linter and just needs to be persisted.

  - [_lib/shared/stats.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-03dd66bfc8332edc2b145936aa2dd607ace1c34a31c222ec4d9617481876c27a): The function `getStats()` is used to append the `stats` data onto the lint result in both _cli-engine.js_ as well as _flat-eslint.js_.

  - [_lib/shared/types.js](https://github.com/mnkiefer/eslint/pull/1/files#diff-28a93c0c2a90d26ab4f8007aed5c72473b5650f25dde18060bac60c1081a2767R188): We first document the new feature by adding `Stats` to the `LintResult` type definition. Its definition is as follows:

    ```js
    /**
        * Performance statistics
        * @typedef {Object} Stats
        * @property {number} directives The number of disable directives which have been applied to silence rule violations.
        * @property {number} fixPasses The number of times ESLint has applied at least one fix after linting.
        * @property {Times} times The times spent on (parsing, fixing, linting) a file.
        * @property {number} violations The number of times a rule has been violated.
        */

    /**
        * Performance statistics times
        * @typedef {Object} Times
        * @property {Object} fix The times spent on each fix pass applied.
        * @property {number} parse The time that is spent when parsing a file.
        * @property {Object} rules An array of time objects for each file/rule/selector combination.
        * @property {number} total The total time that is spent on (parsing, fixing, linting) a file.
        */
    ```

  - [_lib/linter/timing.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-126a649c1db33de2cfe67b418435b10d45fc310143547e334f7be9a1a73c0901R142): The Linter collects the TIMING information in the `time(key, fn, filename, selector, reset)` function where we have added two optional parameters to collect more granular information for the `filename` and `selector` for a given rule. We collect this information in a new Object `times` which is persisted in a new Map `lintTimesPerRule` under the key `"lintTimes"`. Finally, we add new function `timing.getLintTime()`, with which the Linter can collect the respective lint times. The `reset` parameter is used reset the times once a file is re-parsed.

  - [_lib/linter/linter.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-a4ade4bc7a7214733687d84fbb54b625e464d13be7181caf54f564e5985db980R1117): When the `stats` option is enabled, the Linter class method `verifyAndFix()` adds the properties `directives`, `fixPasses`, `times` and `violations` to the `LintResult` object.
    - To collect the times information, the Linter only calls `times.time(ruleId, ruleListeners)` when `times.enabled = true`. Hence, we add a condition here that the function is also called when `stats=true`. This, in turn, is called from `runRules()`, so we must add the `stats` option as an extra input parameter.
    - For all collected properties, we and extend the `internalSlotsMap` and add helper functions (`getDirectives()`, `getFixPasses()`, `getTimes()`, `getViolations()`) respectively to store and later collect them to append to the lint result.
    - The properties *directives* and *violations* are already collected by ESLint via `commentDirectives.directives` and `commentDirectives.problems` and only need to be persisted.
    - The *parse* time property (`times.parse`) is collected in the `parse()` function. Here, we also store the filename in the `resetTimes` variable to reset the times data.
    - The `fixPasses` property is implemented and collected from the Linter method `verifyAndFix()`.


## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

This RFC relates to two sections.

- In the [Command Line Interface Reference](https://eslint.org/docs/latest/use/command-line-interface) section, the new `stats` flag must be listed:
```
--stats     Add additional statistics to lint results
```

- In the [Profile Rule Performance](https://eslint.org/docs/latest/extend/custom-rules#profile-rule-performance) section, it could be mentioned that one can turn on the `stats` flag:

```
For a more detailed performance analysis, the `stats` flag can be employed to append the *per rule per file* lint times onto the final lint results.
```

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
None


## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

- The current ESLint users should not be affected by these changes, as the statistics are only collected when `--stats` is enabled.

## Alternatives

For performance analyses prior to this implementation of this dashboard, a series of eslint calls (or a series of processing steps) as well as creation of a custom UI was necessary to derive and then depict the more detailed `TIMING` information to monitor and quickly get an overview of the individual rule's lint time evolution.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

None

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

More detailed requirements and feedback from the ESLint team would be necessary to implement this feature.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->
 None

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- See related issue: https://github.com/eslint/eslint/issues/16690