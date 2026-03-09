- Repo: `eslint/eslint`, possibly `eslint/rewrite`
- Start Date: 2026-02-03
- RFC PR: eslint/rfcs#145
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
- the current working directory, if one of `--config` or `--no-config-lookup` is passed
- otherwise, the directory where the config file is located. If a project contains different config files, each config file has its own base path.

Internally, this value is stored as `basePath` on `ConfigArray`. It serves two purposes:
- it limits which files can be linted
- it provides the default `basePath` for config objects that don't have one explicitly set. The config object's base path is then used to resolve that object's `files` and `ignores`

### Proposed Change

This proposal adds a way to set a custom base path that overrides the `basePath` of each config array.

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

Relative paths in the config file will now be resolved relative to `..`. This includes `files` and `ignores` settings in config objects that don't include a `basePath`, or the config object's `basePath` itself. So, a config object that specifies different settings for the current directory and its sibling would look like this:

<a name="example-1"></a>

```js
export default defineConfig([
    {
        name: "app-config",
        files: ["app/**/*.{js,cjs,mjs}"],
        ...myAppConfig
    },
    {
        name: "tmp-config",
        files: ["tmp/**/*.{js,cjs,mjs}"],
        ...myTmpConfig
    },
]);
```

or similarly, using a `basePath`:

<a name="example-2"></a>

```js
export default defineConfig([
    {
        name: "app-config",
        basePath: "app",
        files: ["**/*.{js,cjs,mjs}"],
        ...myAppConfig
    },
    {
        name: "tmp-config",
        basePath: "tmp",
        files: ["**/*.{js,cjs,mjs}"],
        ...myTmpConfig
    },
]);
```

### Alternative: Maintain Reference Location

The above solution is fully backward-compatible but it's hard to extend to existing configurations because changing the base path also means that all relative paths and patterns in existing configs (in `files`, `ignores` and `basePath`) must be modified to be relative to the new base path.

For example, in the example in the previous section, a project may already have a config that lints files in the current directory (`app`) like this:

```js
export default defineConfig([
    {
        name: "app-config",
        files: ["**/*.{js,cjs,mjs}"],
        ...myAppConfig
    },
]);
```

In order to extend this config to also lint files in `../tmp`, besides passing `--base-path=..`, the existing config object(s) in the config must be changed if they should still apply to the current directory only, either by modifying the `files` patterns as in [the first example](#example-1), or by adding an explicit `basePath` like in [the second example](#example-2). Note that in both cases, the name of the current working directory (`app`) must be explicitly added, because there is no longer an implicit way to reference it from the config.

To avoid this inconvenience, we could add a new property to config arrays, tentatively named `referenceLocation`. This property will hold the value of what is currently the base path (CWD or config file location), and it will be used to resolve relative `basePath` values in config objects. This will ensure that existing config arrays can be extended to lint additional files with a new common directory root without changing existing config objects. For example, with this alternative, the example above can be extended to lint `../tmp` without changing the other config object(s), like this:

```js
export default defineConfig([
    {
        name: "app-config",
        files: ["**/*.{js,cjs,mjs}"],
        ...myAppConfig
    },
    {
        name: "tmp-config",
        files: ["../tmp/**/*.{js,cjs,mjs}"],
        ...myTmpConfig
    },
]);
```

With this alternative, `--base-path` would define which files are in scope for file enumeration and linting, but resolution of `basePath` values in config objects would remain unchanged.

While still maintaining the changes non-breaking and backward-compatible, this alternative solution requires changing `@eslint/config-array` to introduce a new property on `ConfigArray` instances. It will also add complexity to the implementation.

### Testing Impact

The implementation should include coverage for:
- CLI `--base-path` validation (empty string, invalid values, relative and absolute paths)
- file enumeration behavior when `--base-path` is provided and no file patterns are passed on the command line
- interaction with `--config` and `--no-config-lookup`
- interaction with the ESLint constructor `cwd` option when resolving a relative `basePath`
- behavior when config lookup is enabled (depending on the decision for `--base-path` support in that mode)

### Preview

Prototype branches for the proposed implementations are available for testing and previewing:
- [issue-19118](https://github.com/eslint/eslint/tree/issue-19118): prototype implementation for the main proposal
- [issue-19118-alt](https://github.com/eslint/eslint/tree/issue-19118-alt): prototype implementation for the [alternative proposal](#alternative-maintain-reference-location)

## Documentation

- [CLI options documentation](https://eslint.org/docs/latest/use/command-line-interface#options) needs to include the `--base-path` option
- [`ESLint` constructor documentation](https://eslint.org/docs/latest/integrate/nodejs-api#-new-eslintoptions) needs to include `options.basePath`
- [Configuration Migration Guide](https://eslint.org/docs/latest/use/configure/migration-guide) should mention the use case of linting files outside the current directory
- if a new `ConfigArray` option is added, [`@eslint/config-array`](https://github.com/eslint/rewrite/tree/main/packages/config-array#readme) documentation should be updated

## Drawbacks

Besides the added maintenance burden, this change will add complexity to the configuration, in terms of logic, documentation, and testing. Users and maintainers will need to reason about an additional parameter that plays a role in the behavior of a config.

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

### Path-resolution model when `--base-path` is provided

- Main proposal: relative config paths (`files`, `ignores`, and config-object `basePath`) resolve from `--base-path`.
- Alternative proposal: `--base-path` defines linting scope, while relative config-object `basePath` values resolve from `referenceLocation` (CWD or config file location).

Which model should ESLint adopt?

### `--base-path` with config lookup?

Should `--base-path` be allowed when config lookup is enabled (i.e. neither `--config` nor `--no-config-lookup` is passed)? In that case, ESLint searches for a config file in a file's directory and its ancestors. This means that config files outside that line of search are not considered, so that each file is required to have its associated config in an expected location. In this setup, I'm not sure how `--base-path` would be useful.

## Help Needed

I can implement this RFC.

## Related Discussions

- eslint/eslint#19118
- eslint/eslint#18806
