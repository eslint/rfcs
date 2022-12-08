- Repo: eslint/eslint
- Start Date: 2022-12-08
- RFC PR: <https://github.com/eslint/rfcs/pull/100>
- Authors: [bmish](https://github.com/bmish)

# Flexible configuration and stricter default behavior for reporting unused disable directives

## Summary

<!-- One-paragraph explanation of the feature. -->

Provide users with more flexible, intuitive control over how unused disable directives are reported by switching the CLI and config file options from using a boolean value to a severity level. Change the default behavior to error on unused disable directives.

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

We propose a new "explicit severity" design where the user can set the severity level of unused disable directive reporting using either the CLI option or the config file option. This design has the following benefits:

- The user should always be able to achieve the desired effect (`off` / `warn` / `error`), regardless of where they set the option from (CLI option, config file option, or even from a shareable config)
- No confusing inconsistency where one of the options triggers errors but the other triggers warnings

We will also change the default behavior to error on unused disable directives so that all users (unless they opt-out) will benefit from the detection (and automatic removal with `--fix`) of unused disable directives.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

How the new "explicit severity" design works:

- The config and CLI options are both changed from booleans to instead accept standard severity level string values: `off`, `warn`, `error` (or a the corresponding number for each level)
- If both options are present, the CLI option takes precedence over (overrides) the config option
- If only one of the options is present, the value of the present option is used
- If neither option is present, the default value (`error`) is used
- Note: There's only one underlying setting, and the config and CLI options are just two different ways of controlling it

To reduce complexity and avoid confusion, we will not allow boolean values to be provided anymore, except during a possible intermediary [phase](#phases).

In addition to allowing an explicit severity level to be passed to the CLI option `--report-unused-disable-directives` (e.g. `--report-unused-disable-directives warn`), we will also allow the option to be provided without a value (e.g. `--report-unused-disable-directives`) (a common CLI shorthand) which will default to `error` severity, the same as today.

### Phases

The implementation of this RFC will likely involve two phases:

1. Phase 1: A non-breaking change to add support for the new severity levels, in addition to the existing boolean values. This intermediary, dual-support phase will give shareable configs using `reportUnusedDisableDirectives: true` a chance to switch over to `reportUnusedDisableDirectives: "warn"` while supporting both the ESLint minor version (e.g. `v8.x`) in which phase 1 gets implemented as well as the ESLint major version in which phase 2 gets implemented (e.g. `v9.0`).
2. Phase 2: A breaking change intended for a major release in which support for the boolean values is removed, and the default behavior is changed to `error`.

Pros of the phased approach:

1. Without this phased approach, the only chance the user would have to cut-over to the new severity levels would be during the ESLint major version bump in which they were implemented. A shareable config would only be able to support the pre-implementation major version or the post-implementation major version.
2. Separating breaking changes from non-breaking changes keeps the changes focused.
3. Users can use and benefit from the new severity levels sooner, without having to wait for them in the next major version.

Cons of the phased approach:

1. Lengthier / more drawn-out implementation.
2. Complexity/overhead of supporting both booleans and severity levels during the transition period.
3. Based on searching the top 1,000 ESLint plugins with [eslint-ecosystem-analyzer](https://github.com/bmish/eslint-ecosystem-analyzer), I found only a handful of shared configs setting `reportUnusedDisableDirectives` who would be impacted by this.

### Code Changes

- `conf/config-schema.js` - switch to string / number
- `conf/default-cli-options.js` - use new default
- `lib/options.js` - switch to string / number and use new default
- Anywhere `reportUnusedDisableDirectives` is passed around as a boolean needs to change to using a severity level (and no more need to convert between boolean and severity level)
- A few test files that set `reportUnusedDisableDirectives` as a boolean will be updated to use a severity level

Luckily, `reportUnusedDisableDirectives` is already stored as a severity level in much of the underlying code.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

These documentation pages will be updated to reflect the new behavior of these options:

- Configuration Files
- Configuring Rules
- Command Line Interface
- Node.js API
- `eslint --help`

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

- [Breaking changes](#backwards-compatibility-analysis) cause churn and disruption.
- Potential [semver policy implications](#semver-policy) as described below.

### Semver Policy

Erroring by default on unused disable directives will require slight tweaks to our [semantic versioning policy](https://github.com/eslint/eslint#semantic-versioning-policy). It currently reads:

> Patch release (intended to not break your lint build)
> > A bug fix in a rule that results in ESLint reporting fewer linting errors.
> > ...
>
> Minor release (might break your lint build)
> > A bug fix in a rule that results in ESLint reporting more linting errors.
> > ...

These lines will need to be adjusted because a bug fix that causes a rule to produce fewer errors will now break builds if disable directives become unused as a result. See this context:

- <https://github.com/eslint/eslint/issues/12703#issuecomment-568582014>
  > If we allowed for errors to be reported with reportUnusedDisableDirectives, we would limit what kind of changes we could publish in a semver-patch release (since fixing a bug that would result in fewer errors could create additional unused disable directives).
- <https://github.com/eslint/eslint/pull/14699#discussion_r650863268>
  > ...because of our semver policy, which says that bug fixes that produce fewer errors should not break builds.

#### Semver Policy Alternative 1

One possibility for updating the semver policy is that we could consider any bug fix to a rule, regardless of whether it results in more or less violations, as simply a bug fix and therefore suitable for release as a patch release.

- This is my preference.
- It's how I believe most of the ecosystem (i.e. ESLint plugins) treats bug fixes in real-world usage.
- It's also similar to [Prettier's policy](https://github.com/prettier/prettier/issues/2201) that allows formatting tweaks/fixes in any type of release:
  > If prettier produces a different output on two different versions, that's not a breaking change. Ideally prettier will never have any breaking changes, but if there are, it would be to the the API/CLI itself, not the output.
- Some bug fixes cause a rule to produce a combination of more violations and less violations. The distinction between the two is not always clear-cut and not necessarily deserving of different treatment in the semver policy.
- From what I can tell, attempts to work around the existing semver policy are why we have ended up with such a convoluted and unsatisfying design for reporting unused disable directives in the first place, so it could be time to rethink this constraint.
- In addition, the current restriction of lint-build-breaking changes to minor releases already seems arbitrary, as technically, any breaking changes should be limited to major releases according to semver. This obviously wouldn't be practical for us, as it would prevent us from making any bug fixes to rules except in annual major releases. So since ESLint already does not strictly follow semver in terms of breaking lint builds, why impose this arbitrary limitation on what types of bug fixes can be made in patch releases?
- Even today, it's somewhat aspirational to proclaim that a patch release will never break your lint build.

#### Semver Policy Alternative 2

If, however, we want to maintain the categorization of changes based on whether they can break your lint build, then any change to the number of violations a rule produces would have to change from being patch release compatible to only minor release compatible.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

Users specifying the following config file options will experience a breaking change:

- `reportUnusedDisableDirectives: true`. No longer allowed, must be changed to `reportUnusedDisableDirectives: "warn"` for the same behavior.
- `reportUnusedDisableDirectives: false`. No longer allowed, must be changed to `reportUnusedDisableDirectives: "off"` for the same behavior.

Shareable configs specifying `reportUnusedDisableDirectives: true` will need to be updated to avoid breaking consumers.

Based on searching the top 1,000 ESLint plugins, only a handful of usages of `reportUnusedDisableDirectives` were found, so this breaking change may have limited impact. Also see the earlier discussion of [phases](#phases) that should help ease migration.

Users specifying `--report-unused-disable-directives` on the CLI will not experience a breaking change, as this will continue to report unused disable directives as errors. However, since erroring on unused disable directives is the new default behavior, specifying this option is now redundant.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

1. Leave as-is. Reduces churn, but has downsides as mentioned above.
2. Allow both existing boolean values and also new severity levels indefinitely. This has the potential to cause confusion, increase complexity, and continue to be unintuitive as to how the boolean values map to severities. We might as well bite the bullet and fully cleanup and simplify the allowed values instead of allowing the legacy values indefinitely.
3. Adopt new option design but use `warn` as the default behavior. But `warn` is easily ignored and best used only [temporarily](https://github.com/eslint/eslint/discussions/16512#discussioncomment-4089769).
4. Adopt new option design but use `off` as the default behavior. This would maintain the existing behavior for the majority of users who aren't reporting unused violations. But most users would continue to miss out on the benefit of this feature then.
5. Turn `reportUnusedDisableDirectives` into a regular ESLint rule as suggested in [#13104](https://github.com/eslint/eslint/issues/13104). While this could enable us to reduce complexity by eliminating a config option, a new rule implementing this feature may require special case handling, and it's not clear it's possible to implement within the current architecture.

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
2. Is the [phased approach](#phases) worth the overhead?
3. Which [semver policy](#semver-policy) alternative do we prefer?

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
