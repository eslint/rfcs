- Repo: eslint/eslint
- Start Date: 2022-05-13
- RFC PR:
- Authors: Domantas Petrauskas

# CLI option to suppress ignored file warnings

## Summary

<!-- One-paragraph explanation of the feature. -->

When ignored files are explicitly linted, ESLint shows a warning. When `--max-warnings 0` option is used, and an ignored file is passed to ESLint, the process exits with code 1. In some cases this is unexpected and currently, there is no way to suppress ignored file warnings via CLI.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

While this warning is reasonable in cases where filenames are passed to ESLint manually, it causes issues when the process is automated with tools like [lint-staged](https://github.com/okonet/lint-staged) and [pre-commit](https://github.com/pre-commit/pre-commit). These tools pass staged filenames to any command, not just ESLint. Therefore, they are not aware of the .eslintignore. Linting before a commit is commonly set up with `--max-warnings 0`. In such cases, ESLint will exit with an error and prevent commits, or even cause CI to fail.

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

It is possible to filter out ignored files using ESLint `CLIEngine`. This was the main reason to reject the CLI flag idea during the [TSC meeting in 2018](https://gitter.im/eslint/tsc-meetings/archives/2018/08/02). The committee was only considering the use case of integrations. However, this problem affects end users as well. The lint-staged FAQ includes a section on [how can I ignore files from .eslintignore](https://github.com/okonet/lint-staged#how-can-i-ignore-files-from-eslintignore), which suggests adding an additional config file and using `CLIEngine.isPathIgnored` to filter out ignored files. Having a CLI flag for this would improve the end-user experience.

This RFC proposes a `--warn-ignored/--no-warn-ignored` CLI option, which would allow suppression of the warning. For example, with

```
// .eslintignore
foo.js
```

Running

```
eslint --no-warn-ignored foo.js
```

Would not result in a warning.
To avoid breaking changes, this will only be available via FlatESLint, [as decided in the RFC discussion](https://github.com/eslint/rfcs/pull/90#issuecomment-1386024387).

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

ESLint CLI should have an optional flag to disable the ignored file warning. The flag should be a boolean. It should be called `--warn-ignored/--no-warn-ignored`. If `--no-warn-ignored` is present, ESLint should suppress the `File ignored because of a matching ignore pattern.` warning. The flag should replicate `eslint.lintText`[warnIgnored option](https://eslint.org/docs/developer-guide/nodejs-api#-eslintlinttextcode-options) when it is set to `false`.

Add the flag to `optionator()` in `options.js`:

First, add the property to the `ParsedCLIOptions` typedef:

```
@property {boolean} warnIgnored Show a warning when the file list includes ignored files
```

Define the flag only when FlatESLint is used:

```js
let warnIgnoredFlag;

if (usingFlatConfig) {
  warnIgnoredFlag = {
    option: "warn-ignored",
    type: "Boolean",
    default: "true",
    description: "Show warning when the file list includes ignored files",
  };
}
```

It should then be placed under the `heading: "Handle Warnings"`.

Add `warnIgnored` to `translateOptions()` in `cli.js`:

```js
async function translateOptions({
    ...,
    warnIgnored
}) {
    ...
    if (configType === "flat") {
        ...
        options.warnIgnored = warnIgnored;
    }
}
```

Add `warnIgnored` to `processOptions()` in `eslint-helpers.js` and set it to `true` by default, and add validation:

```js
function processOptions({
    ...,
    warnIgnored = true
}) {
    ...
    if (typeof warnIgnored !== "boolean") {
        errors.push("'warnIgnored' must be a boolean.");
    }
    ...
    return {
        ...,
        warnIgnored
    }
}
```

### lintFiles

Destructure `warnIgnored` from `eslintOptions` in `lintFiles()` function of `flat-eslint.js`:

```js
const {
    ...,
    warnIgnored
} = eslintOptions;
```

When iterating over `filePaths`, add an additional condition for `createIgnoreResult`:

```js
filePaths.map(({ filePath, ignored }) => {
    /*
    * If a filename was entered that matches an ignore
    * pattern, and warnIgnored is true, then notify the user.
    */
    if (ignored) {
        if (warnIgnored) {
            return createIgnoreResult(filePath, cwd);
        }
        return void 0;
    }
    ...
})
```

### lintText

In order to maintain backward compatibility, `lintText` should only be modified in `flat-eslint.js`. Currently, the `lintText` function already has a `warnIgnored` parameter, and it is set as `false` [by default](https://github.com/eslint/eslint/blob/87b247058ed520061fe1a146b7f0e7072a94990d/lib/eslint/flat-eslint.js#L970). To maintain consistency with the rest of ESLint, it should default to the constructor's `warnIgnored` option (`true` by default). It should still be possible to override it by passing `{ warnIgnored: false }` to `lintText`.

Remove the default `false` in `lintText` in `flat-eslint.js`:

```diff
const {
    filePath,
-    warnIgnored = false
+    warnIgnored,
    ...unknownOptions
} = options || {};
```

Destructure `warnIgnored` from `eslintOptions` and rename it to `constructorWarnIgnored` to make it distinct from the options from function parameters:

```js
const {
    ...
    warnIgnored: constructorWarnIgnored
} = eslintOptions;
```

Extend the condition on which ignore result is created. `warnIgnored` from function arguments overrides `constructorWarnIgnored`:

```js
const shouldWarnIgnored = typeof warnIgnored === "boolean" ? warnIgnored : constructorWarnIgnored;
...
if (shouldWarnIgnored) {
    results.push(createIgnoreResult(resolvedFilename, cwd));
}
```

We could use nullish coalescing here (`warnIgnored ?? constructorWarnIgnored`), but it was only added in Node 14, and ESLint needs to support Node 12.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The [CLI Documentation](https://eslint.org/docs/user-guide/command-line-interface) should be updated to include the new flag. It is necessary to mention that it is only available when using FlatESLint.

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

Adding additional CLI flags adds additional mental overhead for the users. It might be confusing for users who don't experience this problem. Since we maintain backward compatibility, ESLint and FlatESLint are going to behave slightly differently.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

Users of FlatESLint linting text via stdin might encounter unexpected `File ignored because of a matching ignore pattern` errors.

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

> No, https://github.com/eslint/rfcs/pull/90#discussion_r907721233

Should the `warnIgnored` option be available via `eslint.config.js` too?

> No, https://github.com/eslint/rfcs/pull/90#discussion_r1137743213

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
