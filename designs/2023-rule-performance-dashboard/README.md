- Repo: eslint/eslint
- Start Date: 2023-03-10
- RFC PR:
- Authors: Mara Nikola Kiefer (@mnkiefer)

# Rule Performance Dashboard

## Summary

<!-- One-paragraph explanation of the feature. -->

This document describes how to expose ESLint's [`TIMING`](https://eslint.org/docs/latest/extend/custom-rules#profile-rule-performance) information to the [formatters](https://eslint.org/docs/latest/use/formatters/) and use it in a new formatter [`html-charts`]() to generate a **Rule Performance Dashboard**.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Analysing rule performance currently requires additional scripting to collect/extract more *granular* timing data (lint time per file per rule):

1. Running ESLint per rule, per file and then collecting the file/rule time data output:

  ```bash
  TIMING=1 DEBUG=eslint:cli-engine eslint --no-eslintrc --rule ... file
  ```

2. Doing a single ESLint run on all files and then extracting the file/rule time data output:

  ```bash
  TIMING=all DEBUG=eslint:cli-engine eslint ... files
  ```

In addition, one needs to create an overview for an effective presentation of the results.

It would be more *convenient and shareable* to have:

   1. The **timing data** ESLint already collects exposed to the formatters
   2. A **built-in formatter** for that data

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

The *proof-of-concept* can be found at:
- [**ESLint**](https://github.com/mnkiefer/eslint/pull/1): **Fork of ESLint** with the above mentioned changes.
- [**Sample project**](https://github.com/mnkiefer/eslint-samples): Example on which the dashboard (shown below) was generated.

### ESLint *timing* exposed

Exposure of the timing object requires only a few changes in the Linter and CLI engline.

The timing is already collected in `lib/linter/timing.js`. Note, that the function requires the extra input paramter `filename`. For the function itself, the only change is that we now *persist* that information in the `lintTimesPerRule` object, which stores the detailed *per file per rule* lint times under the **lintTimes** key. An extra function `getLintTime()` is returned to the linter to retrieve this object.

The linter `lib/linter/linter.js` retrieves the timing data only when `TIMING` is enabled (i.e. from the ESLint CLI).

Finally, we append the new [timing data](#timing-data) to the *Lint Report* in the CLI engine `lib/cli-engine/cli-engine.js` such that they are accessible to the formatters:
- `lintTime` (ms): Lint time of file
- `lintOrder` (order number): Lint order of file
- `lintTimePerRule` (ms): Lint time of file per rule

Below is an expert of a sample for the 5th file that was linted in the sample project:

```json
    {
        ...
        "lintOrder": 5,
        "lintTime": 13.642954999999999,
        "lintTimePerRule": {
            "constructor-super": 0.05145500000000001,
            "for-direction": 0.032875,
            "getter-return": 0.003292,
            "no-async-promise-executor": 0,
            "no-case-declarations": 0,
            "no-class-assign": 0,
            "no-compare-neg-zero": 0,
            "no-cond-assign": 0.035042000000000004,
            "no-const-assign": 0.0025,
            "no-constant-condition": 0.563125,
            ...
        },
        ...
        "fileSize": 312
    }
```

### Formatter `html-charts`


<img width="600" alt="rule-performance-dashboard" src="./htmlCharts.png">


The **Rule Performance Dashboard** consists of two parts:

1. On the left hand side (**1**), we have the usual ESLint HTML report. Here, we have embedded the already established `html` formatter as a iframe with some small styling modifications. However, this report could have also been generated independently of another formatter but we have required it here to keep the code slim and focus on the charts.

2. On the right hand side (**2**), we have the charts created by the [Chart.js](https://www.chartjs.org/) library. The first chart (**2a**) is a pie chart of the usual `TIMING` performance results the user is used to seeing from ESLint's stdout. The second chart (**2c**) contains the more detailed *per file per rule* information for each file (x-axis) and lint time (y-axis, left, as line chart) per rule as well as the the respective file size (y-axis, right, as bar chart).The file sizes and the total lint times are shown in the background in gray, while the individual rule lint times are colors lines for better focus. Note, that both charts will update on changes to the rule selection checkbox menu (**2b**, top right corner of the screen) such that one can easily view and compare different rule (times) across all files, which can help to detect more intricate performance issues that may be overlooked otherwise (based on rule reports or average values across entire runs only).

The above dashboard stems from an ESLint run on the [sample project](https://github.com/mnkiefer/eslint-samples) with 28 `*.js` files of valid/invalid [recommended rules](https://eslint.org/docs/latest/rules/) examples (as taken from [ESLint's Rule Documentation](https://eslint.org/docs/latest/rules/) examples).

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

This RFC relates to two sections.

- In the [Profile Rule Performance](https://eslint.org/docs/latest/extend/custom-rules#profile-rule-performance) section, it could be mentioned that the formatter `html-charts` may help to more depict and more closely analyse the results.

- The [Formatters](https://eslint.org/docs/latest/use/formatters/) section should include the formatter `html-charts`.

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

- Current implementation uses third party library [Chart.js](https://www.chartjs.org/)@4.2.1
-  Adding a new built-in formatter for a **Rule Performance Dashboard** may be redundant as the UI and design is heavily dependent on use cases and personal preference.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

- Trying to use the `html-charts` formatter without having `TIMING` on will produce the following error as it is only useful for performance analyses:
```
node ../eslint/bin/eslint.js . --format html-charts -o ESLintDashboard.html

Oops! Something went wrong! :(

ESLint: 8.36.0

Error: The TIMING environment variable needs to be set for this formatter.
    at module.exports ...
```

## Alternatives

For performance analyses prior to this implementation of this dashboard, a series of eslint calls (or a series of processing steps) as well as creation of a custom UI was necessary to derive and then depict the more detailed `TIMING` information to monitor and quickly get an overview of the individual rules lint time evolution.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

- Is the inclusion of Chart.js acceptable or should custom UI/Charts be created for ESLint to use?
- If Chart.js is used, the HTML page's UI should be more repsponsive, which media devices should be catered to?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

Yes, but more detailed requirements and feedback from the ESLint team would be necessary.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

> "Why do we need a new built-in formatter?"

Although everone probably wants to create their own dashboard, it is still time consuming and cumbersome to do so. If `html-charts` was offered, the user still as the option to just use it as is, or copy it and adapt his custom formatter from there.

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- See related issue: https://github.com/eslint/eslint/issues/16690