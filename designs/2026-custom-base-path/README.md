- Repo: `eslint/eslint`, possibly `eslint/rewrite`
- Start Date: 2026-02-03
- RFC PR: (leave this empty, to be filled in later)
- Authors: Francesco Trotta

# Support `--base-path` Option

## Summary

This document proposes a mechanism for configuring and linting files relative to a specified directory, regardless of the current working directory (CWD) or where the configuration file is located. This is achieved by introducing a new CLI option `--base-path`, and a corresponding `basePath` option on the `ESLint` constructor.

## Motivation

In the legacy eslintrc configuration system, users could lint files in arbitrary locations by explicitly specifying a config file (for example, via `--config`).
ESLint would resolve the target file paths relative to the current working directory and lint them, even though those files couldn’t be configured by that config.

Flat config, on the other hand, enforces that only files under the current base path can be linted. When a config file is explicitly provided, the base path is the current working directory. This has caused problems for users migrating from legacy config to flat config, where linting files outside the current working directory previously worked. See [related discussions](#related-discussions) below for some examples.

To support the legacy use case while allowing users to configure files independently, we propose introducing a new option that will allow users to specify a custom base path.

## Detailed Design

### Current Behavior

The base path is the directory that defines which files a `ConfigArray` can lint.
Files outside that base path cannot be linted, even if they are matched by a config object.

Today, ESLint determines the base path as follows:
- the current working directory, if either `--config` or `--no-config-lookup` is passed
- otherwise, the directory where the config file is located. If a project contains different config files, each config file has its own base path.

Internally, this value is stored as `basePath` on `ConfigArray`. It serves two purposes:
- it limits which files can be linted
- it provides the default `basePath` for config objects that don't have one explicitly set. The config object's base path is then used to resolve that object's `files` and `ignores`

### Proposed Change

This proposal adds a way to set a custom base path that overrides the `basePath` of each config object.

For users and integration developers, this introduces:
- a new CLI option, `--base-path`
- a corresponding `basePath` option on the `ESLint` constructor

The base path must be a non-empty string and a valid absolute or relative path. If relative, it will be resolved against the current working directory, or against the `cwd` option when provided.

When `basePath` is specified and no file patterns are passed on the command line, ESLint will enumerate files in that base path. For example:

```shell
npx eslint --config=eslint.config.js --base-path=../src
```

will behave the same as 

```shell
npx eslint --config=eslint.config.js --base-path=../src ../src
```

### Alternative: Maintain Reference Location

TODO

### Example Usage

The use case proposed in eslint/eslint#19118 involves linting files in a sibling directory of the current directory.

```text
(root)
├─ app (cwd)
│  ├─ eslint.config.mjs
│  └─ ... (files to lint)
└─ tmp
   └─ ... (more files to lint)
```

For this case, `--base-path` should point to a parent of both directories, i.e. `..`:

```shell
npx eslint --config=eslint.config.mjs --base-path=.. . ../tmp
```

Relative paths in the config file will now be resolved relative to `..`. This includes `files` and `ignores` settings in config objects that don't include a `basePath`, or the config object's `basePath` itself. So, a config object that specifies different settings for the current directory and its sibling would look for example like this:

```js
export default defineConfig([
    {
        name: "app-config",
        files: ["app/**/*.{,c,m}js"],
        ...myAppConfig
    },
    {
        name: "tmp-config",
        files: ["tmp/**/*.{c,m}js"],
        ...myTmpConfig
    },
]);
```

or similarly, using a `basePath`:

```js
export default defineConfig([
    {
        name: "app-config",
        basePath: "app",
        files: ["**/*.{,c,m}js"],
        ...myAppConfig
    },
    {
        name: "tmp-config",
        basePath: "tmp",
        files: ["**/*.{c,m}js"],
        ...myTmpConfig
    },
]);
```

## Documentation

- [CLI options](https://eslint.org/docs/latest/use/command-line-interface)
- [`ESLint` constructor options](https://eslint.org/docs/latest/integrate/nodejs-api#-new-eslintoptions)
- [`@eslint/config-array`](https://github.com/eslint/rewrite/tree/main/packages/config-array#readme) (if a new `ConfigArray` option is added)

## Drawbacks

Besides the added maintenance burden, this change will add complexity to the configuration.

## Backwards Compatibility Analysis

This proposal is backward-compatible and does not introduce any breaking changes.

## Alternatives

### Change `**/` semantics to match outside the base path

One proposal was to make patterns prefixed with `**/` (such as `**/*.js`) match files regardless of base path.

This approach has some drawbacks:

- it creates edge cases for `ignores` (for example, `**/foo/` might unexpectedly match ancestor path segments)
- it does not provide a clear way to target only specific paths outside the base path
- it would require matching against absolute paths in some cases, which is harder to reason about
- it changes established glob semantics and is potentially breaking

To address the last point and avoid a breaking change, additional solutions have been proposed:

- adding a feature flag `v10_file_matching` for changed glob semantics
- adding a dedicated config key `matchAbsolute` for absolute matching

### Rely on config lookup improvements only

The `unstable_config_lookup_from_file` / `v10_config_lookup_from_file` flag was considered as a possible way to cover these scenarios.

User feedback pointed out that this does not solve the specific problem of linting files outside the effective base path when using a centralized config/tooling setup.

### Add only `basePath` on config objects

Adding `basePath` to config objects (discussed in related design work, see [RFC \#131](https://github.com/eslint/rfcs/tree/main/designs/2025-base-path-in-config-objects)) was also considered as a standalone solution. However, this change alone was not sufficient for this issue because files external to the base path were still being ignored as external.

## Open Questions

When `--base-path` is provided, should relative `basePath` values in config objects continue to resolve exactly as they do today (relative to config location / CWD), or should CLI `--base-path` influence that resolution?

Should `--base-path` be allowed when config lookup is enabled (i.e. neither `--config` nor `--no-config-lookup` is passed)? In that case, ESLint will search for a config file in a file's directory and its ancestors. This means that config files outside that line of search will not be considered, so that each file is required to have its associated config in an expected location. In this setup, I'm not sure how `--base-path` would be useful.

## Help Needed

I can implement this RFC.

## Related Discussions

- eslint/eslint#19118
- eslint/eslint#18806
