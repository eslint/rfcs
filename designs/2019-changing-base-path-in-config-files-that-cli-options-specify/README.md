- Start Date: 2019-09-18
- RFC PR: https://github.com/eslint/rfcs/pull/37
- Authors: Toru Nagashima <http://github.com/mysticatea>

# Changing Base Path of `overrides` and `ignorePatterns` that CLI Options Specify

## Summary

This RFC changes the base path of `overrides`, `ignorePatterns`, and `.eslintignore` from the directory which contains the config file to the current working directory if the config file was specified by CLI options `--config` or `--ignore-path`.

## Motivation

Currently, the base path of `overrides`, `ignorePatterns`, and `.eslintignore` is the directory which contains the config file even if the config file was specified by CLI options `--config` or `--ignore-path`.

```bash
# the paths of 'overrides' and 'ignorePatterns' are relative to 'node_modules/my-config/'
eslint lib --config node_modules/@me/my-config/.eslintrc.js
# the paths in `.eslintignore` are relative to 'node_modules/my-config/'
eslint lib --ignore-path node_modules/@me/my-config/.eslintignore
```

This is a barrier to use `--config`/`--ignore-path` option with shared files. If we change the base path to the current working directory, it will resolve this problem.

### .gitignore and core.excludesFile

`.eslintignore` has been designed as similar stuff to `.gitignore`. Therefore, it's better if the feature around `.eslintignore` is similar to `.gitignore`.

Git doesn't have `--ignore-path`-like CLI option, but has `core.excludesFile` setting to specify additional ignore file.

```bash
# Use './config/ignore' file as '.gitignore'.
git config core.excludesFile config/ignore
```

In this case, the base path of the paths in `config/ignore` is the repository root. For example, `/node_modules` in `config/ignore` is `./node_modules` rather than `./config/ignore/node_modules`. This behavior is different from our `--ignore-path` option.

## Detailed Design

It changes the base path of the following patterns to the current working directory:

- In the file which is specified by the `--config` option:
    - `overrides[i].files`
    - `overrides[i].excludedFiles`
    - `ignorePatterns`
- In the file which is specified by the `--ignore-path` option:
    - file patterns.

```bash
# the paths of 'overrides' and 'ignorePatterns' are relative to './'
eslint lib --config node_modules/@me/my-config/.eslintrc.js
# the paths in `.eslintignore` are relative to './'
eslint lib --ignore-path node_modules/@me/my-config/.eslintignore
```

This RFC *doesn't* change the resolving logic of `extends`, `parser`, and `plugins`.

### Implementation

It modifies [`createCLIConfigArray()` function](https://github.com/eslint/eslint/blob/869f96aa87c4f990f54e1eeccb0e3f7dbd66e6c2/lib/cli-engine/cascading-config-array-factory.js#L133-L165). In the function, it modifies the elements of the config arrays which are created from the `ignorePath` (corresponds to `--ignore-path`) and the `specificConfigPath` (corresponds to `--config`). It modifies `element.criteria.basePath` (corresponds to `overrides`) and `element.ignorePattern.basePath` (corresponds to `ignorePatterns` and `.eslintignore`) to `cwd`.

## Documentation

This change needs the migration guide because of a breaking change.

This change needs to update the following documents:

- the "[Configuration Based on Glob Patterns » Relative glob patterns](https://github.com/eslint/eslint/blob/869f96aa87c4f990f54e1eeccb0e3f7dbd66e6c2/docs/user-guide/configuring.md#relative-glob-patterns)" section.
- the "[Ignoring Files and Directories » `ignorePatterns` in config files](https://github.com/eslint/eslint/blob/869f96aa87c4f990f54e1eeccb0e3f7dbd66e6c2/docs/user-guide/configuring.md#ignorepatterns-in-config-files)" section.
- the "[Ignoring Files and Directories » `.eslintignore`](https://github.com/eslint/eslint/blob/869f96aa87c4f990f54e1eeccb0e3f7dbd66e6c2/docs/user-guide/configuring.md#eslintignore)" section.
    - As a side note, this section is outdated a bit. It said "Paths are relative to `.eslintignore` location or the current working directory.", but it has been changed since ESLint 4.0.0. Currently, it's "Paths are relative to `.eslintignore` location."

## Drawbacks

This is a breaking change. It can break the current workflow.

For example, if a user runs ESLint on variety directories with the same config by `--config` and `--ignore-path`, the workflow will be broken. The user has to change the path to target files instead of the working directory.

## Backwards Compatibility Analysis

This is a breaking change.

The behavior of `--config` and `--ignore-path` options will be changed.

## Alternatives

### Prior Arts

- [`.gitignore` and `core.excludesFile`](#gitignore-and-coreexcludesfile) is similar to this RFC's behavior.

## Related Discussions

- https://github.com/eslint/eslint/issues/6759
- https://github.com/eslint/eslint/issues/11558
- https://github.com/eslint/eslint/issues/12278
