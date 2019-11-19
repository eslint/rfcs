- Start Date: 2019-11-07
- RFC PR: https://github.com/eslint/rfcs/pull/47
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Plugin Loading Improvement

## Summary

This RFC changes ESLint to load plugins from the directory of the config files, so it fixes "plugin not found" errors for varied environments.

## Motivation

The current behavior is not useful for some environments.

### When developing multiple projects

When the editor integrations run ESLint, it often uses the opened directory as the current working directory. However, it's problematic for monorepo development or something like.

```
<root>
└ packages
  ├ foo
  ├ bar
  ├ ...
  └ zeta
```

Each package may have plugins, but ESLint cannot find such a plugin from the root directory.

## Detailed Design

This RFC includes three changes:

- It changes where ESLint loads plugins from.
- It replaces the `CLIEngine#getRules()` method.
- It replaces the `metadata.rulesMeta` property of formatters.

On the other hand, this RFC keeps the uniqueness of plugins while ESLint lints a file. This means that this RFC doesn't solve [#3458](https://github.com/eslint/eslint/issues/3458).

### ● It changes where ESLint loads plugins from

In the new loading logic, ESLint loads plugins from the directory that contains the closest config file from the target file. In addition, if the `--resolve-plugins-relative-to` option is present, ESLint loads plugins from there as well.

The concrete steps are:

1. If the `--resolve-plugins-relative-to` option is present, ESLint resolves plugins from there. Otherwise, skipped.
1. If the config files of each lint target file are found, ESLint resolves plugins from the directories that contain each config file. Otherwise, ESLint resolves plugins from the current working directory.
   - This step may have different results for each lint target file.
1. If the set of the resolved path in steps 1 and 2 has exactly one element, ESLint adopts it.
   - If the set is empty, ESLint throws the "plugin not found" error.
   - If the set has two or more elements, it means the shadowing of plugins happened. ESLint throws the "multiple plugins found" error because the shadowing can cause confusion.
1. ESLint loads plugins from the adopted path.

#### For Example

```
<root>
├ packages
│ ├ foo
│ │ ├ .eslintrc.yml  # to use specific plugins
│ │ └ test.js
│ ├ bar
│ │ └ test.js
│ ├ ...
│ └ zeta
└ .eslintrc.yml
```

##### When ESLint verifies `packages/foo/test.js`

The config files of `packages/foo/test.js` are `./.eslintrc.yml` and `packages/foo/.eslintrc.yml`. Therefore,

- If `--resolve-plugins-relative-to pckages/zeta` option is present, ESLint loads plugins from `pckages/zeta`, `packages/foo`, or `./`.
- If any `--resolve-plugins-relative-to` option is not present, ESLint loads plugins from `packages/foo` or `./`.
- If any `--resolve-plugins-relative-to` option is not present and actually `packages/foo/.eslintrc.yml` has `root:true`, ESLint loads plugins from only `packages/foo`.

##### When ESLint verifies `packages/bar/test.js`

The config file of `packages/bar/test.js` is `./.eslintrc.yml`. Therefore,

- If `--resolve-plugins-relative-to pckages/zeta` option is present, ESLint loads plugins from `pckages/zeta` or `./`.
- If any `--resolve-plugins-relative-to` option is not present, ESLint loads plugins from `./`.

#### About personal config file (`~/.eslintrc`)

This RFC doesn't change the plugin base path for the personal config file (`~/.eslintrc`) because the functionality has been deprecated in [RFC32](https://github.com/eslint/rfcs/pull/32). ESLint still loads plugins from the current working directory if there is no project config file, even if there is the personal config file.

If people want to use the personal config file, use `--config` option (e.g., `eslint . --config ~/.eslintrc.json`). It should work intuitively after [RFC39].

#### About identification of rule IDs

In order to lint files, ESLint must identify every rule ID to unique implementation. The new logic loads plugins uniquely for each target file. Therefore, the identification is no problem to lint files.

However, the following APIs require the uniqueness of rule IDs in the whole multiple files.

- `CLIEngine#getRules()`
- `metadata.rulesMeta` of formatters

This proposal deprecates those APIs. The following sections describe about this.

### ● It replaces the `CLIEngine#getRules()` method

This RFC replaces the `CLIEngine#getRules()` method by `CLIEngine#getRulesForFile(filePath)` because the `CLIEngine#getRules()` method requires the uniqueness of rule IDs in spanning multiple files.

The `CLIEngine#getRulesForFile(filePath)` method is similar to `CLIEngine#getConfigForFile(filePath)`. ESLint loads the config files of the given target file then returns the `Map` object that contains the core rules and the plugin rules that the config files used.

Once we removed the `CLIEngine#getRules()` method, `CLIEngine` instances no longer need having the configs that the last call of `executeOnFiles()` used. This will simplify the state management of `CLIEngine`.

#### About `ESLint` class in [RFC40]

If [RFC40] is accepted, this RFC just replaces `ESLint#getRules()` by `ESLint#getRulesForFile(filePath)`. This change is safe because the `ESLint` class has not published yet.

#### Removal plan

This RFC adds the runtime-deprecation of the `CLIEngine#getRules()` method in ESLint `7.0.0`.

But this RFC doesn't decide the timing of removing the `CLIEngine#getRules()` method because likely the `CLIEngine` class will be deprecated in [RFC40].

If there are plugins that have the same name but different implementation, the `CLIEngine#getRules()` contains one of the implementations randomly. The runtime-deprecation message tells this fact.

### ● It replaces the `metadata.rulesMeta` property of formatters

This RFC replaces the `metadata.rulesMeta` property by `metadata.getRuleMeta(ruleId, filePath)` method because the `metadata.rulesMeta` property requires the uniqueness of rule IDs in spanning multiple files.

The `metadata.getRuleMeta(ruleId, filePath)` method returns `cliEngine.getRulesForFile(filePath).get(ruleId)?.meta`.

Existing `metadata.rulesMeta[ruleId]` is replaced to `metadata.getRuleMeta(ruleId, filePath)`.

Once we removed the `metadata.rulesMeta` property, `CLIEngine` instances no longer need having the configs that the last call of `executeOnFiles()` used. This will simplify the state management of `CLIEngine`. Plus, we no longer need creating a large object to get a few metadata.

#### About formatter v2 in [RFC45]

If [RFC45] is accepted, this RFC just replaces `context.getRuleMeta(ruleId)` by `context.getRuleMeta(ruleId, filePath)`. This change is safe because the formatter v2 has not published yet.

#### Removal plan

- It adds the runtime-deprecation of the `metadata.rulesMeta` property in ESLint `7.0.0`.
- It removes the `metadata.rulesMeta` property in the first major version after over a year from `7.0.0`.

If there are plugins that have the same name but different implementation, the `metadata.rulesMeta` property contains one of the implementations randomly. The runtime-deprecation message tells this fact.

## Documentation

- It needs an entry in the migration guide because of a breaking change.
- It updates the "[Configuring Plugins](https://eslint.org/docs/user-guide/configuring#configuring-plugins)" section to describe where ESLint loads plugins from.
- It updates the "[Node.js API](https://eslint.org/docs/developer-guide/nodejs-api)" page to describe the new `CLIEngine#getRulesForFile()` method and the deprecated `CLIEngine#getRules()` method.
- It updates the "[Working with Custom Formatters](https://eslint.org/docs/developer-guide/working-with-custom-formatters)" page to describe the new `metadata.getRuleMeta()` method and the deprecated `metadata.rulesMeta` property.

## Drawbacks

- The new logic is complex a bit. It may increase maintenance cost.
- This RFC deprecates two APIs. This enforces users to migrate to new APIs.

## Backwards Compatibility Analysis

This RFC is a breaking change.

- The change of how ESLint loads plugins has just a small impact because Node.js finds packages from the ancestor directories.
- The replace of the `CLIEngine#getRules()` method and the `metadata.rulesMeta` property of formatters has some impact. Users have to migrate to the new APIs.

## Alternatives

Nothing in particular.

## Related Discussions

- https://github.com/eslint/eslint/pull/11696 - New: add `--resolve-plugins-relative-to` flag
- https://github.com/eslint/eslint/issues/12019 - Allow to specify plugin resolution in config
- https://github.com/microsoft/vscode-eslint/issues/696 - ESLint fails to load plugins when using ESLint 6.x
- https://github.com/microsoft/vscode-eslint/issues/708 - Autodetect and propose working directories in mono repositories
- https://github.com/eslint/rfcs/pull/39 - New: Changing the Default Value of --resolve-plugins-relative-to
- https://github.com/eslint/rfcs/pull/40 - New: `ESLint` Class Replacing `CLIEngine`
- https://github.com/eslint/rfcs/pull/45 - New: Formatter v2

[rfc32]: https://github.com/eslint/rfcs/pull/32
[rfc39]: https://github.com/eslint/rfcs/pull/39
[rfc40]: https://github.com/eslint/rfcs/pull/40
[rfc45]: https://github.com/eslint/rfcs/pull/45
