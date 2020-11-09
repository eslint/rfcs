- Repo: eslint/eslint
- Start Date: 2020-09-11
- RFC PR: <https://github.com/eslint/eslint/pull/13812>
- Authors: [bmish](https://github.com/bmish)

# Configurable List Size For Per-Rule Performance Metrics

## Summary

Running `TIMING=1 eslint` from the command-line outputs a list of [per-rule performance metrics](https://eslint.org/docs/developer-guide/working-with-rules#per-rule-performance-1). The list is currently limited to the top 10 longest-running rules. We would like to enable users to see a longer list of rules if desired.

Sample existing output:

```sh
$ TIMING=1 eslint lib
Rule                    | Time (ms) | Relative
:-----------------------|----------:|--------:
no-multi-spaces         |    52.472 |     6.1%
camelcase               |    48.684 |     5.7%
no-irregular-whitespace |    43.847 |     5.1%
valid-jsdoc             |    40.346 |     4.7%
handle-callback-err     |    39.153 |     4.6%
space-infix-ops         |    35.444 |     4.1%
no-undefined            |    25.693 |     3.0%
no-shadow               |    22.759 |     2.7%
no-empty-class          |    21.976 |     2.6%
semi                    |    19.359 |     2.3%
```

<!-- One-paragraph explanation of the feature. -->

## Motivation

Optimizing linting performance can become important in a large codebase or with particularly expensive rules. In order to determine what rules to focus on for performance optimizations, it's necessary to measure the running times of various rules while running linting on a codebase.

While the current output of performance metrics for the top 10 longest-running rules helps, it doesn't give the complete picture, especially when there may be hundreds of rules enabled (from ESLint core and third-party plugins).

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

## Detailed Design

We propose reusing the existing the `TIMING` environment variable to also specify the desired list size.

- The performance list will continue to show only when the `TIMING` variable is present (no change).
- When the list shows, we will read the integer value of the `TIMING` variable to determine the number of rules to show in the list.
- A minimum list size of 10 will be used to maintain backwards compatability (and because there's not much reason to shorten the list beyond that anyway).
- A special string value of `all` will remove the limit entirely (and make it clear that the user does not want a limit).

| Command | Behavior |
| --- | --- |
| `eslint` | do not show |
| `TIMING=true eslint` | show the first 10 rules (due to minimum of 10) |
| `TIMING=0 eslint` | show the first 10 rules (due to minimum of 10) |
| `TIMING=1 eslint` | show the first 10 rules (due to minimum of 10) |
| `TIMING=5 eslint` | show the first 10 rules (due to minimum of 10) |
| `TIMING=10 eslint` | show the first 10 rules |
| `TIMING=11 eslint` | show the first 11 rules |
| `TIMING=15 eslint` | show the first 15 rules |
| `TIMING=100 eslint` | show the first 100 rules |
| `TIMING=all eslint` | show all the rules |

This is a concise, easy-to-type means of gaining additional configurability without adding additional environment variables.

To implement this, we will add a function to calculate the list size in [TIMING.js](https://github.com/eslint/eslint/blob/master/lib/linter/timing.js):

```js
/**
 * Decide how many rules to show in the output list.
 * @returns {number} the number of rules to show
 */
function getListSize() {
    if (process.env.TIMING === "all") {
        return Number.POSITIVE_INFINITY;
    }

    const MINIMUM_SIZE = 10;
    const TIMING_ENV_VAR_AS_INTEGER = Number.parseInt(process.env.TIMING, 10);

    return TIMING_ENV_VAR_AS_INTEGER > 10 ? TIMING_ENV_VAR_AS_INTEGER : MINIMUM_SIZE;
}

function display() {
    // existing code
    .slice(0, getListSize());
}
```

Test cases will also be added to verify the expected behavior.

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

## Documentation

A sentence should be added to the [Per-rule Performance](https://eslint.org/docs/developer-guide/working-with-rules#per-rule-performance-1) section of the documentation website.

No formal announcement is needed.

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

## Drawbacks

- Potential confusion around `TIMING=15` showing 15 results as expected but `TIMING=5` unexpectedly showing 10 results
- Overloading of the `TIMING` environment variable to serve multiple purposes by being used as both a boolean and number
- Inability to show list sizes from 1 to 9 (due to the minimum of 10)

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think
    about any opposing viewpoints that reviewers might bring up.

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

## Backwards Compatibility Analysis

This change should not reasonably be considered a breaking change, given that the vast majority of users of the existing `TIMING` environment variable specify `TIMING=1` (as mentioned in the [documentation](https://eslint.org/docs/developer-guide/working-with-rules#per-rule-performance-1)) or `TIMING=true`.

However, if there was an existing usage that specified `TIMING=11` or above for some unknown reason, such a usage would be affected, albeit rather harmlessly.

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

## Alternatives

### Alternative A1: Add new `TIMING_LIMIT` environment variable (controls both visibility and limit)

- The performance list will show if either `TIMING` is present or `TIMING_LIMIT` is greater than or equal to `1` (so that `TIMING_LIMIT` could be used by itself).
- When the list shows, we will read the integer value of the `TIMING_LIMIT` variable to determine the number of rules to show in the list, only using the value if it is greater than or equal to `1` (otherwise falling back to the original value of `10`).
- The existing `TIMING` environment variable will be left in place, since most people using this feature do not need to configure a specific limit.

| Command | Behavior |
| --- | --- |
| `eslint` | do not show |
| `TIMING_LIMIT=0 eslint` | do not show |
| `TIMING=1 eslint` | show the first 10 rules |
| `TIMING_LIMIT=1 eslint` | show the first 1 rule |
| `TIMING_LIMIT=5 eslint` | show the first 5 rules |
| `TIMING_LIMIT=50 eslint` | show the first 50 rules |
| `TIMING_LIMIT=all eslint` | show all the rules |

- Pro: provides full configurability
- Pro: does not overload the existing `TIMING` boolean environment variable
- Pro: if we want to add more settings for the performance output, we could add additional environment variables following the same convention (`TIMING_SETTING1`, `TIMING_SETTING2`, ...)
- Pro: not a breaking change
- Con: adds additional complexity in the form of more environment variables
- Con: potential confusion around what the behavior should be with different combinations of the two environment variables

### Alternative A2: Add new `TIMING_LIMIT` environment variable (controls only limit and not visibility)

A slight modification to Alternative A1.

- The performance list will show only when the `TIMING` variable is present. Using `TIMING_LIMIT` alone is not sufficient to see the list.

| Command | Behavior |
| --- | --- |
| `eslint` | do not show |
| `TIMING_LIMIT=50 eslint` | do not show |
| `TIMING=1 eslint` | show the first 10 rules |
| `TIMING=1 TIMING_LIMIT=0 eslint` | show the first 10 rules |
| `TIMING=1 TIMING_LIMIT=1 eslint` | show the first 1 rule |
| `TIMING=1 TIMING_LIMIT=5 eslint` | show the first 5 rules |
| `TIMING=1 TIMING_LIMIT=50 eslint` | show the first 50 rules |
| `TIMING=1 TIMING_LIMIT=all eslint` | show all the rules |

### Alternative B: Eliminate the limit entirely

Remove `.slice(0, 10)`.

- Pro: always provides all possible results
- Pro: no added complexity
- Con: can easily flood the command-line with excess/unwanted output
- Con: technically a breaking change

### Alternative C: Increase the limit substantially to 100

Change to `.slice(0, 100)`).

- Pro: more likely to include all the desired output
- Pro: no added complexity
- Con: can easily flood the command-line with excess/unwanted output
- Con: technically a breaking change

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

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

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

See issue [#13671](https://github.com/eslint/eslint/issues/13671).

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
