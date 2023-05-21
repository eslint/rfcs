- Repo: eslint/eslint
- Start Date: 2022-12-08
- RFC PR: <https://github.com/eslint/rfcs/pull/100>
- Authors: [bmish](https://github.com/bmish)

# Flexible configuration and stricter default behavior for reporting unused disable directives

## Summary

<!-- One-paragraph explanation of the feature. -->

Provide users with more flexible, intuitive control over how unused disable directives are reported by switching the CLI and config file options from using a boolean value to a severity level. Change the default behavior to warn on unused disable directives.

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

- The config and CLI options are both changed to also accept standard severity level string values: `off`, `warn`, `error` (or the corresponding number for each level)
- If both the CLI and config options are present, the CLI option takes precedence over (overrides) the config option
- If only the CLI option is present, then it is used; if only the config option is present, then it is used.
- If neither of the CLI or config options are present, the default value (`warn`) is used
- Note: There's only one underlying setting, and the config and CLI options are just two different ways of controlling it

Boolean values will continue to be allowed with no change in behavior, as summarized below.

Allowed values for the config file option:

- `reportUnusedDisableDirectives: true` - same behavior as today (`warn`)
- `reportUnusedDisableDirectives: false` - same behavior as today (`off`)
- `reportUnusedDisableDirectives: "error"` - new value
- `reportUnusedDisableDirectives: "warn"` - new value
- `reportUnusedDisableDirectives: "off"` - new value

Allowed values for the CLI option:

- `--report-unused-disable-directives` - same behavior as today (`error`) (common CLI shorthand to omit the value)
- `--report-unused-disable-directives=true` - same behavior as today (`error`)
- `--report-unused-disable-directives=false` - same behavior as today (`off`)
- `--report-unused-disable-directives=error` - new value
- `--report-unused-disable-directives=warn` - new value
- `--report-unused-disable-directives=off` - new value

### Phases

The implementation of this RFC will likely involve two phases:

1. Phase 1: A non-breaking change to add support for the new severity levels.
2. Phase 2: A breaking change intended for a major release in which the default behavior is changed to `warn`.

This allows us to get the new functionality out to users as soon as possible as a smaller change with reduced risk, while also keeping the breaking change small and focused.

### Code Changes

- `conf/flat-config-schema.js` - also support string / number type
- `conf/default-cli-options.js` - use new default
- `lib/options.js` - also support string / number type and use new default for the option
  - Something like `type: "Boolean|String|Number"` and `enum: ["true", "false", "error", "warn", "off", "0", "1", "2"]`
- When processing the config or CLI options, convert any boolean value provided to a severity level
- Anywhere `reportUnusedDisableDirectives` is passed around as a boolean needs to change to using a severity level
- In addition to testing `reportUnusedDisableDirectives` as a boolean, we'll also need to test it as a severity level

Luckily, `reportUnusedDisableDirectives` is already stored as a severity level in much of the underlying code.

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
- Keeping around the boolean values for backwards compatibility means increased complexity in supporting both boolean and severity level values. It also means we still have the inconsistency where `reportUnusedDisableDirectives: true` means `warn` but `--report-unused-disable-directives=true` means `error`.

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

1. Should we allow severity levels to be provided as numbers? If we allow these numbers everywhere else and have no plans to scrap them, then we likely want to allow them here as well for consistency.

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
