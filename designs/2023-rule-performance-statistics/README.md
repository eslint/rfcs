- Repo: eslint/eslint
- Start Date: 2023-03-10
- RFC PR: <https://github.com/eslint/rfcs/pull/108>
- Authors: Mara Nikola Kiefer ([@mnkiefer](https://github.com/mnkiefer))

# Performance Statistics

## Summary

<!-- One-paragraph explanation of the feature. -->

This document describes a new ESLint flag `stats`, which enables the user to obtain a series of runtime *statistics* on top of their final lint results (refer to the table below).

These statistics include more fine-grained timing information, such as the parse-, fix-, and lint-times and the number of fix passes.

| Stats          |        |       | Description |
|----------------|--------|-------|-------------|
| **times**      |        |       | Object containing the times spent on (parsing, fixing, linting) a file. |
|                | passes |       | Array containing the times spent on (parsing, fixing, linting) a file for each run- or fix-pass. |
|                |        | fix   | The time that is spent on fixing a file. |
|                |        | parse | The time that is spent on parsing a file. |
|                |        | rules | Object containing the times spent on each rule. |
| **fixPasses**  |        |       | The number of times ESLint has applied at least one fix after linting. |

> Note, that each `times.passes` entry `fix`, `parse` and `rules` provides the time measured in the `total` property to keep the `stats` format flexible and extensible.

Fore more information, please find each of the above properties is addressed individually in the [Detailed Design](#detailed-design) section 2 ("Adding the `stats` option to the Linter").

### Proof of concept (POC)
A proof-of-concept of this feature can be found at:
- [**ESLint**](https://github.com/mnkiefer/eslint/pull/1): **Fork of ESLint** on which the [detailed design](#detailed-design) (as described below) has been implemented.
- [**Sample project**](https://github.com/mnkiefer/eslint-samples): Project on which the above implementation can be demonstrated.

### Example
An example which of the POC on the sample project's file _invalid/file-to-fix.js_ is given below.

```js
/*eslint no-regex-spaces: "error", wrap-regex: "error"*/

function a() {
    return /  foo/.test("bar");
}
```

Running ESLint with the new `stats` option...
```sh
eslint invalid/file-to-fix.js --stats --fix -f json
```

... yields the following `stats` entry as part of the formatted lint results object:

```json
"stats": {
  "fixPasses": 2,
  "times": {
    "passes": [
      {
        "parse": {
          "total": 4.533
        },
        "rules": {
          "no-regex-spaces": {
            "total": 0.000958
          },
          "wrap-regex": {
            "total": 0.003125
          }
        },
        "fix": {
          "total": 0.08
        },
        "total": 18.802292
      },
      {
        "parse": {
          "total": 0.733958
        },
        "rules": {
          "no-regex-spaces": {
            "total": 0.22145700000000001
          },
          "wrap-regex": {
            "total": 0.007709000000000001
          }
        },
        "fix": {
          "total": 0.000292
        },
        "total": 1.43075
      },
      {
        "parse": {
          "total": 0.503
        },
        "rules": {
          "no-regex-spaces": {
            "total": 0.007876
          },
          "wrap-regex": {
            "total": 0.013083000000000001
          }
        },
        "fix": {
          "total": 0
        },
        "total": 1.070708
      }
    ]
  }
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

 - [_lib/eslint/eslint-helpers.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-87d53094b12d82e4c11a0e1167d79cf2f471d2f5e5ebb6fc483e891f9dc87a5a): Here, we add the `stats` option for the *FlatESLint* instance and check that its value is of type Boolean.

- [_lib/eslint/flat-eslint.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-03dd66bfc8332edc2b145936aa2dd607ace1c34a31c222ec4d9617481876c27a): Here, we add the `stats` option to the function `verifyText(Object)` which returns the `LintResult` object. So, if `stats=true`, the `stats` properties (collected by the Linter) must be appended to the lint result (see `getStats()` in the next section).
> Note, that we have also adjusted the `linter.verifyAndFix()` input and output, which will be explained in the next section regarding changes to the Linter.

------

1. **Adding the `stats` option to the Linter:**

    The following paragraphs describe the individual `stats` option properties, how they are collected and stored.<br>
    We will start by the exposure of the `times` object, as most of the information is already collected by the Linter and just needs to be persisted.

  - [_lib/shared/stats.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-03dd66bfc8332edc2b145936aa2dd607ace1c34a31c222ec4d9617481876c27a): The function `getStats()` is used to append the `stats` data onto the lint result in _flat-eslint.js_. We have also introduced 3 helper functions, namely `startTime()`, `endTime(t)`, and `storeTime(time, timeOpts, slots, resetTimes = false)` which are used by the _timing.js_ and _linter.js_ scripts to uniformly store the measured times.
    - `startTime()`: Uses `process.hrtime()` to initiate a time measurement.
    - `endTime(t)`: Uses that time `t` to measure the code execution time since.
    - `storeTime(time, timeOpts, slots, resetTimes = false)`: Uses the measured time `t` and the linter's internal slots map `slots` with `timeOpts` to store the time for a given `type` ('parse', 'rules', or 'fix' time) and `filename` (current file) combination.

  - [_lib/shared/types.js](https://github.com/mnkiefer/eslint/pull/1/files#diff-28a93c0c2a90d26ab4f8007aed5c72473b5650f25dde18060bac60c1081a2767R188): We first document the new feature by adding `Stats` to the `LintResult` type definition. Its definition is as follows:

    ```js
    /**
      * Performance statistics
      * @typedef {Object} Stats
      * @property {number} fixPasses The number of times ESLint has applied at least one fix after linting.
      * @property {Times} times The times spent on (parsing, fixing, linting) a file.
      */

    /**
      * Performance Times for each ESLint pass
      * @typedef {Times[]} Time passes
      */

    /**
      * @typedef {Object} Times
      * @property {number} fix The time for the fix pass.
      * @property {number} parse The time that is spent when parsing a file.
      * @property {Object} rules Object of times for each rule.
      * @property {number} total The total time that is spent on a file.
      */

    ```

  - [_lib/linter/timing.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-126a649c1db33de2cfe67b418435b10d45fc310143547e334f7be9a1a73c0901R142): The Linter collects the TIMING information in the `time(key, fn, filename, slots)` function where we have added two optional parameters to collect more granular information for the `filename` for a given rule in `slots`, which is the linter's internal slots map. The current filename is also stored in `slots.resetTimes` which can be used to reset the times when that file gets re-parsed. Note, that we import three helper functions: `startTime()`, `endTime()`, and `storeTime()` which are also used in the linter below to normalize how to record and track times for each (parse, rules, fix) property.

  - [_lib/linter/linter.js_](https://github.com/mnkiefer/eslint/pull/1/files#diff-a4ade4bc7a7214733687d84fbb54b625e464d13be7181caf54f564e5985db980R1117): When the `stats` option is enabled, the Linter class method `verifyAndFix()` adds the properties `fixPasses` and `times` to the `LintResult` object. The `total` lint times for each pass are collected on this level.
    - To collect the rule performance information, the Linter thus far only called `timing.time(ruleId, ruleListeners)` when `times.enabled = true`. Hence, we add a condition here that the function may also be called when `stats=true`. This, in turn, is called from `runRules()`, so we must add the `stats` option as an extra input parameter as well as `slots` where we store the collected time information (i.e. the linter's internal slots map).
    - For all collected properties, we and extend the `internalSlotsMap` and add helper functions (`getFixPasses()`, `getTimes()`) respectively to store and later collect them to append to the lint result.
    - The *parse* time property (`times.parse`) is wrapped around the `parse()` function. Note, that we also track the filename in `slots.resetTimes`to reset the rules times data (if not in fix mode) as this signals a new parse of a file that has already been linted.
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
