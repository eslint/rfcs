- Start Date: 2019-05-12
- RFC PR: https://github.com/eslint/rfcs/pull/22
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Configuring core options in Config Files

## Summary

This RFC adds several properties into config files to configure core options. People can use config files (including shareable configs!) instead of CLI options in order to configure some linter behavior.

## Motivation

We cannot configure some linter's behavior with config files, especially, shareable configs. It's convenient if we can configure those in shareable configs.

## Detailed Design

This RFC adds four properties to config files.

```js
// .eslintrc.js
module.exports = {
    noInlineConfig: false,              // Corresponds to --no-inline-config
    reportUnusedDisableDirectives: false,    // Corresponds to --report-unused-disable-directives
    verifyOnRecoverableParsingErrors: false, // Corresponds to --verify-on-recoverable-parsing-errors
    ignorePatterns: [],                      // Corresponds to --ignore-pattern

    overrides: [
        {
            files: ["*.ts"],
            noInlineConfig: false,
            reportUnusedDisableDirectives: false,
            verifyOnRecoverableParsingErrors: false,
            // ignorePatterns: [], // Forbid this to avoid confusion with 'excludedFiles' property.
        },
    ],
}
```

### § noInlineConfig

That value can be a boolean value. Default is `false`.

If `true` then it disables inline directive comments such as `/*eslint-disable*/`.

If `noInlineConfig` is `true`, `--no-inline-config` was not given, and there are one or more directive comments, then ESLint reports each directive comment as a warning message (`severify=1`). For example, `"'eslint-disable' comment was ignored because your config file has 'noInlineConfig' setting."`. Therefore, end-users can know why directive comments didn't work.

<table><td>
<b>💠Implementation</b>:
<p>In <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/linter.js#L859"><code>Linter#_verifyWithoutProcessors</code> method</a>, the linter checks both <code>providedConfig</code> and <code>filenameOrOptions</code> to determine <code>noInlineConfig</code> option. The <code>filenameOrOptions.allowInlineConfig</code> precedences <code>providedConfig.noInlineConfig</code>.</p>
</td></table>

### § reportUnusedDisableDirectives

That value can be a boolean value. Default is `false`.

If `true` then it reports directive comments like `//eslint-disable-line` when no errors would have been reported on that line anyway.

This option is different a bit from `--report-unused-disable-directives` CLI option. The `--report-unused-disable-directives` CLI option fails the linting with non-zero exit code (i.e., it's the same behavior as `severity=2`), but this `reportUnusedDisableDirectives` setting doesn't fail the linting (i.e., it's the same behavior as `severity=1`). Therefore, we cannot configure ESLint with `reportUnusedDisableDirectives` as failed by patch releases.

<table><td>
<b>💠Implementation</b>:
<ol>
<li><code>Linter</code> and <code>CLIEngine</code> have <code>options.reportUnusedDisableDirectives</code>. This RFC enhances these options to accept <code>"off"</code>, <code>"warn"</code>, and <code>"error"</code>. Existing <code>false</code> is the same as <code>"off"</code> and existing <code>true</code> is the same as <code>"error"</code>.</li>
<li>In <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/linter.js#L859"><code>Linter#_verifyWithoutProcessors</code> method</a>, the linter checks both <code>providedConfig</code> and <code>filenameOrOptions</code> to determine <code>reportUnusedDisableDirectives</code> option. The <code>filenameOrOptions.reportUnusedDisableDirectives</code> precedences <code>providedConfig.reportUnusedDisableDirectives</code>.</li>
</ol>
</td></table>

### § verifyOnRecoverableParsingErrors

That value can be a boolean value. Default is `false`.

If `true` then it runs rules even if recoverable errors existed. Then it shows both results.

<table><td>
<b>💠Implementation</b>:
<p>In <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/linter.js#L859"><code>Linter#_verifyWithoutProcessors</code> method</a>, the linter checks both <code>providedConfig</code> and <code>filenameOrOptions</code> to determine <code>verifyOnRecoverableParsingErrors</code> option. The <code>filenameOrOptions.verifyOnRecoverableParsingErrors</code> precedences <code>providedConfig.verifyOnRecoverableParsingErrors</code>.</p>
</td></table>

### § ignorePatterns

That value can be an array of strings. Default is an empty array.

This is very similar to `.eslintignore` file. Each value is a file pattern as same as each line of `.eslintignore` file. ESLint compares the path to source code files and the file pattern then it ignores the file if it was matched. The path to source code files is addressed as relative to the entry config file, as same as `files`/`excludedFiles` properties.

ESLint concatenates all ignore patterns from all of `.eslintignore`, `--ignore-path`, `--ignore-pattern`, and `ignorePatterns`. If there are multiple `ignorePatterns` in a `ConfigArray`, all of them are concatenated. The order is:

1. The default ignoring. (I.e. `.*`, `node_modules/*`, and `bower_components/*`)
1. `ignorePatterns` in the appearance order in the config array.
1. `--ignore-path` or `.eslintignore`.
1. `--ignore-pattern`

Negative patterns mean unignoring. For example, `!.*.js` makes ESLint checking JavaScript files which start with `.`. Negative patterns are used to override parent settings.
Also, negative patterns is worthful for shareable configs of some platforms. For example, the config of VuePress can provide the configuration that unignores `.vuepress` directory.

It disallows `ignorePatterns` property in `overrides` entries in order to avoid confusion with `excludedFiles`. And if a `ignorePatterns` property came from shareable configs in `overrides` entries, ESLint ignores it. This is the same behavior as `root` property.

The `--no-ignore` CLI option disables `ignorePatterns` as well.

<table><td>
<b>💠Implementation</b>:
<ul>
<li>At <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/cli-engine/file-enumerator.js#L402">lib/cli-engine/file-enumerator.js#L402</a>, the enumerator checks if the current path should be ignored or not.</li>
<li>At <a href="https://github.com/eslint/eslint/blob/af81cb3ecc5e6bf43a6a2d8f326103350513a1b8/lib/cli-engine.js#L897-L906">lib/cli-engine.js#L897-L906</a>, <code>CLIEngine#isPathIgnored(filePath)</code> method uses <code>ignorePatterns</code>.
</ul>
</td></table>

### § Other options?

- `extensions` - This RFC doesn't add `extensions` option that corresponds to `--ext` because [#20 "Configuring Additional Lint Targets with `.eslintrc`"](https://github.com/eslint/rfcs/pull/20) is the good successor of that.
- `rulePaths` - This RFC doesn't add `rulePaths` option that corresponds to `--rulesdir` because [#14 (`localPlugins`)](https://github.com/eslint/rfcs/pull/20) is the good successor of that. Because the `rulePaths` doesn't have namespace, shareable configs should not be able to configure that. (Or but it may be useful for some plugins such as `@typescript-eslint/eslint-plugin` in order to replace core rules. I'd like to discuss the replacement way in another place.)
- `format` - This RFC doesn't add `format` option that corresponds to `--format` because it doesn't fit cascading configs. It needs another mechanism.
- `maxWarnings` - This RFC doesn't add `maxWarnings` option that corresponds to `--max-warnings` because it doesn't fit cascading configs. It needs another mechanism.

## Documentation

- [Configuring ESLint](https://eslint.org/docs/user-guide/configuring) page should describe new top-level properties.
- [`--no-ignore` document](https://eslint.org/docs/user-guide/command-line-interface#--no-ignore) should mention `ignorePatterns` setting.

## Drawbacks

Nothing in particular.

## Backwards Compatibility Analysis

No concerns. Currently, unknown top-level properties are a fatal error.

## Alternatives

-

## Related Discussions

- [eslint/eslint#3529](https://github.com/eslint/eslint/issues/3529) - Set ignore path in .eslintrc
- [eslint/eslint#4261](https://github.com/eslint/eslint/issues/4261) - combine .eslintignore with .eslintrc?
- [eslint/eslint#8824](https://github.com/eslint/eslint/issues/8824) - Allow config to ignore comments that disable rules inline
- [eslint/eslint#9382](https://github.com/eslint/eslint/issues/9382) - Proposal: `reportUnusedDisableDirectives` in config files
- [eslint/eslint#10341](https://github.com/eslint/eslint/issues/10341) - do not ignore files started with `.` by default
- [eslint/eslint#10891](https://github.com/eslint/eslint/issues/10891) - Allow setting ignorePatterns in eslintrc
- [eslint/eslint#11665](https://github.com/eslint/eslint/issues/11665) - Add top-level option for noInlineConfig or allowInlineConfig
