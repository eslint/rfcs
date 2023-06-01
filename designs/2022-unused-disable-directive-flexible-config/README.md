- Repo: eslint/eslint
- Start Date: 2022-12-08
- RFC PR: <https://github.com/eslint/rfcs/pull/100>
- Authors: [bmish](https://github.com/bmish)

# Flexible configuration and stricter default behavior for reporting unused disable directives

## Summary

<!-- One-paragraph explanation of the feature. -->

Provide users with more flexible, intuitive control over how unused disable directives are reported by supporting severity levels instead of just boolean values with CLI and config file options. Change the default behavior from off to warn on unused disable directives.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Outdated/unused ESLint disable directive comments (e.g. `notConsole(); // eslint-disable-line no-console`) are currently a blind spot for most apps/repositories. These leftover comments cause clutter and confusion.

### Existing Solutions

Detecting unused disable directive comments requires going out of one's way by using one of the following existing solutions:

1. Running linting with `eslint --report-unused-disable-directives` (errors, but requires providing the CLI option whenever running ESLint)
2. Adding `reportUnusedDisableDirectives: true` to the `.eslintrc.js` config file (warns, and thus easily ignored)
3. Enabling the third-party rule [eslint-comments/no-unused-disable](https://mysticatea.github.io/eslint-plugin-eslint-comments/rules/no-unused-disable.html) (errors, but requires a third-party plugin)

These existing solutions have various downsides:

1. Lack of discoverability.
2. Requires manual configuration to enable and thus not commonly used.
3. Inability to treat unused disable directives as errors in shareable configs using the first-party options.
4. Inconsistency in behavior between the CLI option and config file option of the same name which is unintuitive and causes confusion.

### New Design

We propose a new "explicit severity" design where the user can set the severity level of unused disable directive reporting using either the CLI option or the config file option. As a result, the user should always be able to achieve the desired effect (`off` / `warn` / `error`), regardless of where they set the option from (CLI option, config file option, or even from a shareable config).

We will also change the default behavior to warn on unused disable directives so that all users (unless they opt-out) will benefit from the detection (and automatic removal with `--fix`) of unused disable directives.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

How the new "explicit severity" design works:

- A new CLI option `--report-unused-disable-directives-severity <severity>` is added, while the existing CLI option `--report-unused-disable-directives` remains in place untouched
- If both CLI options `--report-unused-disable-directives` and `--report-unused-disable-directives-severity <severity>` are provided, an error is thrown
- The existing config file option `reportUnusedDisableDirectives` is updated to, in addition to the current boolean values, also accept standard severity level string values: `off`, `warn`, `error` (or the corresponding number for each level)
- If both the CLI and config file options are present, the CLI option takes precedence over (overrides) the config file option
- If only the CLI option is present, then it is used; if only the config file option is present, then it is used
- If neither the CLI nor config file options are present, the default value (`warn`) is used
- Note: There's only one underlying setting, and the config and CLI options are just two different ways of controlling it

Allowed values for the config file option:

- `reportUnusedDisableDirectives: true` - same behavior as today (`warn`)
- `reportUnusedDisableDirectives: false` - same behavior as today (`off`)
- `reportUnusedDisableDirectives: "error"` - new value (or `2`)
- `reportUnusedDisableDirectives: "warn"` - new value (or `1`)
- `reportUnusedDisableDirectives: "off"` - new value (or `0`)

Allowed values for the CLI options:

- `--report-unused-disable-directives` - existing option, same behavior as today (`error`)
- `--report-unused-disable-directives-severity error` - new option (or `2`)
- `--report-unused-disable-directives-severity warn` - new option (or `1`)
- `--report-unused-disable-directives-severity off` - new option (or `0`)

### Phases

The implementation of this RFC will likely involve two phases:

1. Phase 1: Non-breaking changes to add support for the new severity levels.
2. Phase 2: Any breaking changes for flat config users (`warn` by default, remove redundant `reportUnusedDisableDirectives` option from `processOptions()`), suitable for a minor release prior to ESLint v9 since the flat config is not considered stable yet, or otherwise a major release.

This allows us to get the new functionality out to users as soon as possible as a smaller change with reduced risk, while also keeping the breaking change small and focused.

### Code Changes

See this [Draft PR](https://github.com/eslint/eslint/pull/17212) of phase 1 code changes.

- `conf/default-cli-options.js` - default to `undefined` for new CLI option `--report-unused-disable-directives-severity`
- `lib/cli.js` - convert `--report-unused-disable-directives` and `--report-unused-disable-directives-severity` to `reportUnusedDisableDirectives`
- `lib/config/default-config.js` - (phase 2, breaking change for flag config users) add new default value of `warn` for `reportUnusedDisableDirectives` config option
- `lib/config/flat-config-schema.js` - support boolean or severity value for `linterOptions.reportUnusedDisableDirectives`
- `lib/eslint/eslint-helpers.js` - (phase 2, breaking change for flag config users) remove option `reportUnusedDisableDirectives` from `processOptions()` because `overrideConfig.reportUnusedDisableDirectives` can be used instead now
- `lib/eslint/eslint.js` - same change as `lib/eslint/eslint-helpers.js`
- `lib/linter/linter.js` - normalize severity or boolean for `linterOptions.reportUnusedDisableDirectives` to a severity string
- `lib/options.js` add new CLI option `--report-unused-disable-directives-severity`
- `tests/lib/cli.js` - test the CLI options
- `tests/lib/linter/linter.js` - test the `reportUnusedDisableDirectives` config option

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

These documentation pages will be updated to reflect the new option values and defaults:

- Configuration Files
- Configuring Rules
- Command Line Interface
- Node.js API
- `eslint --help`

Documentation will focus on the new severity level values as opposed to the boolean values. We may choose to leave the boolean values undocumented in some places, or add a note that they are deprecated and remain in place only for backwards compatibility, similar to the note in the [globals configuration section](https://eslint.org/docs/latest/user-guide/configuring/language-options#specifying-globals) that says:

> For historical reasons, the boolean value false and the string value "readable" are equivalent to "readonly". Similarly, the boolean value true and the string value "writeable" are equivalent to "writable". However, the use of these older values is deprecated.

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

- While we attempted to limit the [breaking changes](#backwards-compatibility-analysis) involved, any amount of breaking changes can still be disruptive.
- Some users may not like the new default behavior of warning on unused disable directives and will be burdened by having to opt-out.
- The new warnings can be easily ignored and some may prefer to only use warnings [temporarily](https://github.com/eslint/eslint/discussions/16512#discussioncomment-4089769).
- Keeping around the boolean values for backwards compatibility means increased complexity in supporting both boolean and severity level values. It also means we still have the inconsistency where `reportUnusedDisableDirectives: true` means `warn` but `--report-unused-disable-directives` means `error`.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

Users that already specify `reportUnusedDisableDirectives` or `--report-unused-disable-directives` will not experience any breaking changes, as the current behavior will be maintained.

Users that do not specify any of these options but do specify [`--max-warnings`](https://eslint.org/docs/latest/user-guide/command-line-interface#--max-warnings) will experience a breaking change, as the additional warnings could cause ESLint to exit with a non-zero exit code.

Users that do not specify any of these options are less likely to experience a breaking change, as additional warnings will simply show up as output text. Note that this could still be a breaking change if the user cares about ESLint's exact output text to stdout, or that running with `--fix` will now fix the new warnings.

We don't need tweak our [semantic versioning policy](https://github.com/eslint/eslint#semantic-versioning-policy) (discussed more in [Alternatives](#alternatives)), as the policy will only be violated (i.e. patch releases can break CI) if the user opts-in by setting one of the options to `error`.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

1. Leave as-is. But this means continued lack of configuration flexibility and discoverability of unused disable directives.
2. Remove boolean values for configuration options. We decided against this because it's a breaking change that could be too disruptive for users, and the overhead of supporting both booleans and severity levels is limited.
3. Adopt new option design but use `error` as the default behavior. We decided against this because causing CI to fail due to unused disable directives could be too disruptive to users and violates the current [semantic versioning policy](https://github.com/eslint/eslint#semantic-versioning-policy):
   > Patch release (intended to not break your lint build)
   > > A bug fix in a rule that results in ESLint reporting fewer linting errors.
   > > ...
   >
   > Minor release (might break your lint build)
   > > A bug fix in a rule that results in ESLint reporting more linting errors.
   > > ...

   - <https://github.com/eslint/eslint/issues/12703#issuecomment-568582014>
     > If we allowed for errors to be reported with reportUnusedDisableDirectives, we would limit what kind of changes we could publish in a semver-patch release (since fixing a bug that would result in fewer errors could create additional unused disable directives).
   - <https://github.com/eslint/eslint/pull/14699#discussion_r650863268>
     > ...because of our semver policy, which says that bug fixes that produce fewer errors should not break builds.
4. Adopt new option design but use `off` as the default behavior. This would maintain the existing behavior for the majority of users who aren't reporting unused violations. But most users would continue to miss out on the benefit of this feature then.
5. Turn `reportUnusedDisableDirectives` into a regular ESLint rule as suggested in [#13104](https://github.com/eslint/eslint/issues/13104). While this could enable us to reduce complexity by eliminating a config option, a new rule implementing this feature may require special case handling, and it's not clear it's possible to implement within the current architecture. It's also a larger breaking change.

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

I am open to implementing this.

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

- <https://github.com/eslint/eslint/issues/15466> - the issue triggering this RFC
- <https://github.com/eslint/eslint/issues/9382> - very similar earlier proposal with `reportUnusedDisableDirectives` accepting a severity level
- <https://github.com/eslint/rfcs/pull/22> - previous RFC to allow `reportUnusedDisableDirectives` in config files
