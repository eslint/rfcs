- Repo: eslint/eslint
- Start Date: 2022-05-13
- RFC PR:
- Authors: Domantas Petrauskas

# CLI option to supress ignored file warnings

## Summary

<!-- One-paragraph explanation of the feature. -->

When ignored files are explicitly linted, ESLint shows a warning. This causes problems when using tools which pass the file list to ESLint automatically together with `--max-warnings 0` option.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

While this is warning is reasonable in cases where filenames are passed to ESLint manually, it causes issues when the process is automated with tools like [lint-staged](https://github.com/okonet/lint-staged) and [pre-commit](https://github.com/pre-commit/pre-commit). These tools pass staged filenames to any command, not just ESLint. Therefore, they are not aware of the .eslintignore. Linting before commit is commonly set up with `--max-warnings 0`, therefore ESLint will exit with an error in such case.

For example, with

```
// .eslintignore
foo.js
```

Running

```
eslint foo.js
```

Will result in

```
warning  File ignored because of a matching ignore pattern. Use "--no-ignore" to override
```

It is possible to filter out ignored files using ESLint `CLIEngine`. This was the main reasoning to reject the CLI flag idea during [TSC meeting in 2018](https://gitter.im/eslint/tsc-meetings/archives/2018/08/02). The commitee was only considering the use case of integrations. However, this problem affects end users as well. Lint-staged FAQ includes a section [how can I ignore files from .eslintignore](https://github.com/okonet/lint-staged#how-can-i-ignore-files-from-eslintignore), which suggests adding additional config file and using `CLIEngine.isPathIgnored` to filter out ignored files. Having a CLI flag for this would improve the end-user experience.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

ESLint CLI should have an optional flag to disable the ignored file warning. The flag should be boolean. In order to stay consistent with current CLI flags, it should be called `--no-warning-on-ignored-files`. If it is present, ESLint should supress the `File ignored because of a matching ignore pattern.` warning. The flag should replicate `eslint.lintText`[warnIgnored option](https://eslint.org/docs/developer-guide/nodejs-api#-eslintlinttextcode-options) when it is set to `false`.

The [3 warnings](https://github.com/eslint/eslint/blob/f31216a90a6204ed1fd56547772376a10f5d3ebb/lib/cli-engine/cli-engine.js#L299-L305) related to ignored files should be appended with `Use "--no-warning-on-ignored-files" to suppress this warning.`

Add the flag to `optionator()` in `options.js`:

```js
{
    option: "warning-on-ignored-files",
    type: "Boolean",
    default: "true",
    description: "Show warning when the file list includes ignored files"
},
```

It should be placed under `heading: "Miscellaneous"`.

Add `warningOnIgnoredFiles` to `translateOptions()` in `cli.js`. It should be returned as `warnIgnored`. Forced `warnIgnored: true` should be removed [here](https://github.com/eslint/eslint/blob/f31216a90a6204ed1fd56547772376a10f5d3ebb/lib/cli.js#L298) in order for CLI to respect the `--no-warning-on-ignored-files` flag when `useStdin` is true.

Destructure `warnIgnored` from `options` from `internalSlotsMap.get(this)` in `executeOnFiles()` function in `cli-engine.js`.

In `iterateFiles()` of `executeOnFiles()` function in `cli-engine.js`, wrap the `results.push(createIgnoreResult(filePath, cwd));` with `if (warnIgnored)` to suppress the warning for ignored files. Keep in mind that the ignored file should not be linted.

Destructure `warnIgnored` from `options` from `internalSlotsMap.get(this)` in `executeOnText()` function in `cli-engine.js` and rename it to `engineWarnIgnored` in order to distinguish it from the `warnIgnored` parameter. Update the `if (warnIgnored)` condition to fallback to `engineWarnIgnored` in case `warnIgnored` parameter is not provided.

Remove the code which overrides the default value of `warnIgnored` [here](https://github.com/eslint/eslint/blob/f31216a90a6204ed1fd56547772376a10f5d3ebb/lib/cli.js#L298) and [here](https://github.com/eslint/eslint/blob/main/lib/eslint/eslint.js#L571)

The default `warnIgnored` value should only be set in `default-cli-options.js` and `eslint.js` `processOptions`. It should be set to `true`. The CLIEngine and ESLint classes will inherit this and use it in `lintText`, `lintFiles`, `executeOnFiles`, and `executeOnText`. This is a breaking change, as previously `lintText` and `executeOnText` functions were defaulting to `warnIgnored: false`. This change will improve consistency.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The [CLI Documentation](https://eslint.org/docs/user-guide/command-line-interface) should be updated to include the new flag. Additionally, [Node.js API documentation](https://eslint.org/docs/developer-guide/nodejs-api#-new-eslintoptions) will need to add additional option under `new ESLint(options)`.

[Ignoring Code](https://eslint.org/docs/user-guide/configuring/ignoring-code) page will need to be updated to reflect the changes to the error messages.

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

Adding additional CLI flags adds additional mental overhead for the users. It might be confusing for users who don't experience this problem.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

It should not affect existing users in any way.

The `CLIEngine.isPathIgnored` solution will continue working, it is `lint-staged` and similar tools responsibility to communicate that it is possible to use the flag instead.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

It was considered to output the warning to `stderr` instead, but that would cause breaking changes.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

ESLint [collects suppressed warnings and errors](https://github.com/eslint/eslint/pull/15459). In case `--no-warning-on-ignored-files` or `warnIgnored: false` is used, should the warning be included in the suppressed messages array?

<!-- ## Help Needed -->

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

<!-- ## Frequently Asked Questions -->

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

Main issue:

https://github.com/eslint/eslint/issues/15010

Related issues:

https://github.com/eslint/eslint/issues/9977

https://github.com/eslint/eslint/issues/12206

https://github.com/eslint/eslint/issues/12249
