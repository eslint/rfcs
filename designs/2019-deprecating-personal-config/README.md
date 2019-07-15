-   Start Date: 2019-07-16
-   RFC PR: (leave this empty, to be filled in later)
-   Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Deprecating Personal Config

## Summary

This RFC deprecates the personal config that is `.eslintrc` files on home directory.

## Motivation

Since 6.0.0, ESLint doesn't load plugins/configs/parsers from the global installation even if it's not a part of a project such as the personal config. This is inconvenient for the personal config. But, in #28 discussion, we confirm that we don't want to use the global installation.

We don't recommend to use global-installed ESLint. But the personal config has encouraged to use it.

## Detailed Design

Three steps:

1. **Soft deprecation**: Update our documentation to announce to deprecate the personal config on the next minor release.
2. **Hard deprecation**: Add the deprecation warning of the personal config on ESLint 7.0.0.
    - ESLint shows a warning when it loaded the personal config.
    - ESLint shows a warning when it ignored the personal config (Currently, it ignores `.eslintrc` files on home directory if a project directory is in home directory and the project has `.eslintrc` files.)
3. **Removal**: Remove the personal config functionality on ESLint 8.0.0.
    - ESLint no longer loada the personal config even if the project config was not found.
    - ESLint no longer ignores the config files on home directory even if the project config was found.

## Documentation

-   Remove the personal config from [Configuring ESLint](https://eslint.org/docs/user-guide/configuring) page.
-   Announce on the release notes.

## Drawbacks

It will make people inconvenient to lint standalone files.

## Backwards Compatibility Analysis

Yes, this is a breaking change.

To lint standalone files, people have to have a dummy project directory such as `sandbox`.

## Alternatives

-   https://github.com/eslint/rfcs/pull/28

## Related Discussions

-   https://github.com/eslint/rfcs/pull/7
-   https://github.com/eslint/rfcs/pull/28
-   https://github.com/eslint/eslint/issues/11914
