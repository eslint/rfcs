- Repo: eslint/rewrite (`@eslint/config-array` and `@eslint/config-helpers`), eslint/eslint
- Start Date: 2025-03-31
- RFC PR: https://github.com/eslint/rfcs/pull/131
- Authors: Milos Djermanovic

# Support `basePath` property in config objects

## Summary

Currently, config array has a single property `basePath` (string) at the top level. When used from `eslint`, this property is set to the location of the config file, or the current working directory (when `--no-config-lookup` or `--config` options are used). `files` and `ignores` patterns in all config objects are treated as relative to the config array's `basePath`.

This RFC proposes allowing config objects to specify their own `basePath` property. When present, config object's `basePath` overrides config array's `basePath`, meaning that `files` and `ignores` patterns in that config object should be treated as relative to the config object's `basePath` instead of the config array's `basePath`. Also, when `basePath` is present in a config object that has neither `files` nor `ignores`, then the config object applies to files in `basePath` and its subdirectories only.

The new `basePath` property of config objects can be an absolute path, which is primarily intended for internal use by `eslint`, or a relative path, which is primarily intended for end users. A relative path is interpreted as relative to the config array's `basePath`.

`defineConfig()` will be updated to apply base config's `basePath` to extended configs, meaning that all `files` and `ignores` patterns in extended configs become relative to the base config's `basePath`. This includes global ignores in extended configs. Specifying `basePath` in extended configs is not allowed. If `defineConfig()` encounters `basePath` property in an extended config, it will throw an error.

## Motivation

Patterns passed through the `--ignore-pattern` CLI option / `ignorePatterns` API option should be treated as relative to the current working directory. When the current working directory must be the same or descendant of the base path directory, it is easy to transform the patterns by prepending the relative path from the base path directory to the current working directory, which is how it is [currently implemented](https://github.com/eslint/eslint/blob/03fb0bca2be41597fcea7c0e84456bbaf2e5acca/lib/config/config-loader.js#L568-L604).

However, with the [new configuration file resolution](https://github.com/eslint/rfcs/tree/main/designs/2024-config-lookup-from-file) (currently available with the `unstable_config_lookup_from_file` flag; it will become the default behavior in ESLint v10), the current working directory no longer has to be a descendant of the base path directory, which makes pattern transformations very difficult, if not theoretically impossible. Instead of transforming patterns, we will introduce a new property that represents a directory to which the patterns are relative.

Additional use cases may include user-specific scenarios, like manually merging configuration files from different directories, in which case specifying a `basePath` for imported patterns would be helpful to avoid transforming them manually.

## Detailed Design

Proof of concept:

- `@eslint/config-array` and `@eslint/config-helpers` changes: https://github.com/mdjermanovic/rewrite/pull/1
- `eslint` changes: https://github.com/mdjermanovic/eslint/pull/6

Most of the changes will be in the `@eslint/config-array` package.

### Changes in `@eslint/config-array`

- `baseSchema` will be updated to allow the new `basePath` property in config objects.
- `filesAndIgnoresSchema` will be updated to validate that the new `basePath` property, if present, has a string value.
- `normalizeConfigPatterns()` will be updated to normalize the `basePath` property value to an equivalent namespace-prefixed path, for consistency with the config array's `basePath` property value.
- `META_FIELDS` will be updated to include `"basePath"`, in order to treat config objects with `basePath` + `ignores` as global ignores.
- `get ignores()` will be updated to return an array of config objects instead of an array of ignore patterns. This is because some of the patterns may be defined in config objects that have the `basePath` property and are therefore relative to different paths, so combining all patterns into one array would no longer be useful for users of this package.
- `getConfigWithStatus()` will be updated to use config objects' `basePath` when present.
- `shouldIgnorePath()` will be updated to take an array of config objects instead of ignore patterns, and use config objects' `basePath` when present.

### Changes in `@eslint/config-helpers`

- `extendConfig()` will be updated to copy `basePath` from the base config to the resulting extended config.
- `processExtends()` will be updated to throw an error if any of the extended configs contains `basePath` property.

### Changes in `eslint`

- In `lib/config/flat-config-array.js`, `META_FIELDS` will be updated to include `"basePath"`, in order to treat config objects with `basePath` + `ignores` as global ignores while preprocessing config array to remove global ignores when the `--no-ignore` option is used.
- In `lib/config/config-loader.js`, the code that handles `ignorePatterns` (`--ignore-pattern`) will be updated to add them to the config array with `basePath` set to `cwd`, without any transformations.
- In `lib/types/index.d.ts`, `Linter.Config` type will be updated with the `basePath` property.

## Documentation

The documentation for this feature will be added to the [Configuration Files](https://eslint.org/docs/latest/use/configure/configuration-files) page.

## Drawbacks

This add more complexity to the `@eslint/config-array` package.

## Backwards Compatibility Analysis

### `eslint`

There will be no breaking changes in the `eslint` package.

### `@eslint/config-array`

- Introducing a new predefined property of config objects is a breaking change by itself for this package, as it can't be used as a custom property anymore.
- `get ignores()` will have a new signature - a different return value.

Neither of these changes are breaking for the `eslint` package as it doesn't use these properties.

## Alternatives

The `--ignore-pattern` / `ignorePatterns` problem could _maybe_ be solved by transforming patterns. However, even if it is theoretically possible, it would be immensely complex to implement correctly.

## Open Questions

Should this RFC include the `--basePath` feature (https://github.com/eslint/eslint/issues/19118)? I didn't see a relation between config-level base paths and ability to override config array's base path, so I didn't include it in this RFC.

## Help Needed

I can implement this RFC.

## Frequently Asked Questions

### Does this allow linting files outside config array's `basePath`?

No. This use case would be covered by the `--basePath` feature.

## Related Discussions

* https://github.com/eslint/eslint/issues/18948
