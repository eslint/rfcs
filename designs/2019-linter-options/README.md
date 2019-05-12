- Start Date: 2019-05-12
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# The `linterOptions` property in Config Files

## Summary

This RFC adds `linterOptions` property to config files. People can use config files (including shareable configs!) instead of CLI options in order to configure some linter behavior.

## Motivation

We cannot configure some linter's behavior with config files, especially, shareable configs. It's convenient if we can configure those in shareable configs.

## Detailed Design

This adds `linterOptions` property to config files with three properties.

```js
// .eslintrc.js
module.exports = {
    linterOptions: {
        allowInlineConfig: true,              // Corresponds to --no-inline-config / `options.allowInlineConfig`
        reportUnusedDisableDirectives: "off", // Corresponds to --report-unused-disable-directives / `options.reportUnusedDisableDirectives`
        ignorePatterns: [],                   // Corresponds to --ignore-pattern / `options.ignorePattern`
    },

    overrides: [
        {
            files: ["*.ts"],
            linterOptions: {
                allowInlineConfig: true,
                reportUnusedDisableDirectives: "off",
                ignorePatterns: [],
            },
        },
    ],
}
```

Ajv should verify it by JSON Scheme and reject unknown properties in `linterOptions`.

### allowInlineConfig

That value can be a boolean value. Default is `true`.

If `false` then it disables inline directive comments such as `/*eslint-disable*/`.

<table><td>
🚀 <b>Implementation</b>:
<p>In <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/linter.js#L859"><code>Linter#_verifyWithoutProcessors</code> method</a>, the linter checks both <code>providedConfig</code> and <code>filenameOrOptions</code> to determine <code>allowInlineConfig</code> option. The <code>filenameOrOptions.allowInlineConfig</code> precedences <code>providedConfig.linterOptions.allowInlineConfig</code>.</p>
</td></table>

### reportUnusedDisableDirectives

That value can be one of `"off"`, `"warn"`, and `"error"`. Default is `"off"`.

It reports directive comments like `// eslint-disable-line` when no errors would have been reported on that line anyway.

- If `"warn"` then it doesn't cause to change the exit code of ESLint.
- If `"error"` then it causes to change the exit code of ESLint.

<table><td>
🚀 <b>Implementation</b>:
<ol>
<li><code>Linter</code> and <code>CLIEngine</code> have <code>options.reportUnusedDisableDirectives</code>. This RFC enhances these options to accept <code>"off"</code>, <code>"warn"</code>, and <code>"error"</code>. Existing <code>false</code> is the same as <code>"off"</code> and existing <code>true</code> is the same as <code>"error"</code>.</li>
<li>In <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/linter.js#L859"><code>Linter#_verifyWithoutProcessors</code> method</a>, the linter checks both <code>providedConfig</code> and <code>filenameOrOptions</code> to determine <code>reportUnusedDisableDirectives</code> option. The <code>filenameOrOptions.reportUnusedDisableDirectives</code> precedences <code>providedConfig.linterOptions.reportUnusedDisableDirectives</code>.</li>
</ol>
</td></table>

### ignorePatterns

That value can be an array of strings. Default is an empty array.

This is very similar to `.eslintignore` file. Each value is a file pattern as same as the line of `.eslintignore` file. ESLint compares the path to source code files and the file pattern then it ignores the file if it was matched. The path to source code files is addressed as relative to the entry config file, as same as `files`/`excludedFiles` properties.

ESLint concatenates all ignore patterns from all of `.eslintignore`, `--ignore-path`, `--ignore-pattern`, and `linterOptions.ignorePatterns`. If there are multiple `linterOptions.ignorePatterns`, all of them are concatenated. The order is:

1. `--ignore-path` or `.eslintignore`.
1. `linterOptions.ignorePatterns` in the appearance order in the config array.
1. `--ignore-pattern`

Negative patterns mean unignoring. For example, `!.*.js` makes ESLint checking JavaScript files which start with `.`. Negative patterns are used to override parent settings.
Also, negative patterns is worthful for shareable configs of some platforms. For example, the config of VuePress can provide the configuration that unignores `.vuepress` directory.

If this property is in `overrides` entries, ESLint uses the `ignorePatterns` property only if `files`/`excludedFiles` criteria were matched.

The `--no-ignore` CLI option disables `linterOptions.ignorePatterns` as well.

<table><td>
🚀 <b>Implementation</b>:
<p>At <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/cli-engine/file-enumerator.js#L402">lib/cli-engine/file-enumerator.js#L402</a>, the enumerator checks if the current path should be ignored or not.</p>
</td></table>

### Other options?

- `extensions` - This RFC doesn't add `extensions` option that corresponds to `--ext` because [#20 "Configuring Additional Lint Targets with `.eslintrc`"](https://github.com/eslint/rfcs/pull/20) is the good successor of that.
- `rulePaths` - This RFC doesn't add `rulePaths` option that corresponds to `--rulesdir` because [#14 (`localPlugins`)](https://github.com/eslint/rfcs/pull/20) is the good successor of that. Because the `rulePaths` doesn't have namespace, shareable configs should not be able to configure that. (Or but it may be useful for some plugins such as `@typescript-eslint/eslint-plugin` in order to replace core rules. I'd like to discuss the replacement way in another place.)
- `format` - This RFC doesn't add `format` option that corresponds to `--format` because it doesn't fit cascading configs. It needs another mechanism.
- `maxWarnings` - This RFC doesn't add `maxWarnings` option that corresponds to `--max-warnings` because it doesn't fit cascading configs. It needs another mechanism.

## Documentation

- [Configuring ESLint](https://eslint.org/docs/user-guide/configuring) page should describe about `linterOptions` setting.
- [`--no-ignore` document](https://eslint.org/docs/user-guide/command-line-interface#--no-ignore) should mention `linterOptions.ignorePatterns` setting.

## Drawbacks

- Nothing in particular.

## Backwards Compatibility Analysis

- No concerns. Currently, the `linterOptions` top-level property is a fatal error.

## Alternatives

-

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [eslint/eslint#3529](https://github.com/eslint/eslint/issues/3529) - Set ignore path in .eslintrc
- [eslint/eslint#4261](https://github.com/eslint/eslint/issues/4261) - combine .eslintignore with .eslintrc?
- [eslint/eslint#8824](https://github.com/eslint/eslint/issues/8824) - Allow config to ignore comments that disable rules inline
- [eslint/eslint#9382](https://github.com/eslint/eslint/issues/9382) - Proposal: `reportUnusedDisableDirectives` in config files
- [eslint/eslint#10341](https://github.com/eslint/eslint/issues/10341) - do not ignore files started with `.` by default
- [eslint/eslint#10891](https://github.com/eslint/eslint/issues/10891) - Allow setting ignorePatterns in eslintrc
- [eslint/eslint#11665](https://github.com/eslint/eslint/issues/11665) - Add top-level option for noInlineConfig or allowInlineConfig
