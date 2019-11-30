- Start Date: 2019-11-07
- RFC PR: https://github.com/eslint/rfcs/pull/47
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Plugin Loading Improvement

## Summary

This RFC changes ESLint to load plugins from the directory of each config file, so it fixes "plugin not found" errors for more varied environments. (Note this proposal doesn't aim at solving [eslint/eslint#3458].)

## Motivation

The current behavior is not useful for some environments.

### When developing a common config file

We can use the config file which is in a common directory by `--config` option. In that case, it's intuitive to load plugins from the directory that contains the config file. But currently, we have to use `--resolve-plugins-relative-to` option along with `--config` option, or have to install required plugins into the current working directory. For example:

```
eslint . --config /home/me/.eslintrc.js --resolve-plugins-relative-to /home/me
```

```
eslint projects/foo --config projects/shared/.eslintrc.js --resolve-plugins-relative-to projects/shared
```

This behavior is not convenient.

### When developing multiple projects

When the editor integrations run ESLint, it often uses the opened directory as the current working directory. However, it's problematic for monorepo development or something like.

    <root>
    └ packages
      ├ foo
      ├ bar
      ├ ...
      └ zeta

Each package may have plugins, but ESLint cannot find such a plugin from the root directory.

## Detailed Design

This RFC includes three changes:

- It changes where ESLint loads plugins from.
- It replaces the `CLIEngine#getRules()` method.
- It replaces the `metadata.rulesMeta` property of formatters.

### ● It changes where ESLint loads plugins from

In the new loading logic, ESLint loads plugins from the location of config files. In other words, it's like written `require.resolve(pluginName)` in the config files.

However, ESLint requires to resolve plugins uniquely in order to use the rule ID such as `foo/a-rule`. Otherwise, `foo/a-rule` may be resolved to an unintentional version of the plugin silently then users will see cryptic errors such as "rule not found" or "invalid options".

The following sections describe how ESLint keeps the plugin uniqueness with this proposal. Basically, it throws clear errors in conflicts.

#### The plugin uniqueness with shareable configs

If a config file depends on other config files via the `extends` setting, ESLint loads plugins in the extendees from the location of the extender (the entry file). Therefore, it keeps the plugin uniqueness in whole the config file.

For example:

    <root>
    ├ node_modules/
    │ └ eslint-config-foo/
    ├ index.js
    └ .eslintrc.js

If the `./.eslintrc.js` has `extends:["foo"]` and the `eslint-config-foo` has `plugins:["bar"]`, ESLint loads the plugin `bar` from the location of the `./.eslintrc.js` because it's the entry file.

⚠️ This means that this proposal doesn't solve [eslint/eslint#3458]. ESLint doesn't recognize plugins in indirect dependencies as same as the current behavior. We have to install the required plugins directly. A small improvement is that we no longer have to install the required plugins into the project root. We can install the required plugins into the directory of subprojects.

#### The plugin uniqueness with cascading config files

If both a config file and another config file in subdirectories have the same plugin, and those plugins are resolved as different entities, then ESLint throws a fatal error with an understandable message. Therefore, it keeps the plugin uniqueness in whole the cascading configurations.

For example:

    <root>
    ├ node_modules/
    │ └ eslint-plugin-foo/
    ├ subdir/
    │ ├ node_modules/
    │ │ └ eslint-plugin-foo/
    │ └ .eslintrc.js
    └ .eslintrc.js

In this case, because `./.eslintrc.js` uses `./node_modules/eslint-plugin-foo` and `./subdir/.eslintrc.js` uses `./subdir/node_modules/eslint-plugin-foo`, ESLint throws an error.

    Plugin conflict was found. ESLint requires to determine the plugin uniquely.

    - "subdir/node_modules/eslint-plugin-foo" (loaded in subdir/.eslintrc.js)
    - "node_modules/eslint-plugin-foo" (loaded in .eslintrc.js)

To clarify, for more examples:

    <root>
    ├ node_modules/
    │ └ eslint-plugin-foo/
    ├ subdir/
    │ └ .eslintrc.js
    └ .eslintrc.js

In this case, it's no problem even if both `./.eslintrc.js` and `./subdir/.eslintrc.js` has `plugins:["foo"]`. Both resolve to `./node_modules/eslint-plugin-foo`.

    <root>
    ├ subdir/
    │ ├ node_modules/
    │ │ └ eslint-plugin-foo/
    │ └ .eslintrc.js
    └ .eslintrc.js

In this case, the `./.eslintrc.js` cannot use `eslint-plugin-foo`. If the `./.eslintrc.js` had `plugins:["foo"]`, ESLint throws a "plugin not found" error.

#### The plugin uniqueness with `--config` option

ESLint loads plugins from the location of the config file even if the config file was given by the `--config` option. On the other hand, the `--config` option doesn't change the loading behavior of the regular config files. For each config file, ESLint loads plugins from the location of the config file.

ESLint handles the plugin conflict as same as the case of cascading config files. In other words, if both a regular config file and the config file that was specified by the `--config` option depend on the same plugin, and ESLint resolved them to different entities, then ESLint throws.

For example:

    <home>
    ├ a-project/
    │ ├ node_modules/
    │ │ └ eslint-plugin-foo/
    │ └ .eslintrc.js
    └ common/
      ├ node_modules/
      │ └ eslint-plugin-foo/
      └ config.js

Then if people did `eslint . --config ../common/config.js` on the `a-project/` directory, ESLint throws an error:

    Plugin conflict was found. ESLint requires to determine the plugin uniquely.

    - "../common/node_modules/eslint-plugin-foo" (loaded in ../common/config.js)
    - "node_modules/eslint-plugin-foo" (loaded in .eslintrc.js)

#### The plugin uniqueness with `--plugin` option and `baseConfig` option

ESLint loads the plugins that are not declared in config files from the current working directory. Such plugins are the following:

- `plugins` option of `CLIEngine` (and `--plugin` CLI option)
- `baseConfig.plugins` option of `CLIEngine`

If a config file has the same plugin as those options ware given, and ESLint resolved them to different entities, then ESLint throws.

    Plugin conflict was found. ESLint requires to determine the plugin uniquely.

    - "node_modules/eslint-plugin-foo" (loaded in CLI options)
    - "subdir/node_modules/eslint-plugin-foo" (loaded in subdir/.eslintrc.js)

#### The plugin uniqueness with `--resolve-plugins-relative-to` option

If the `--resolve-plugins-relative-to` option is present, it overrides the loading behavior completely. ESLint loads all plugins of all config files from the location of the `--resolve-plugins-relative-to` option. This is the current behavior as-is.

Therefore, people can use the previous behavior by `--resolve-plugins-relative-to .` if needed.

#### About personal config file (`~/.eslintrc`)

This proposal changes the loading behavior of the personal config file (`~/.eslintrc`) even if it's a deprecated feature. Because the loading logic of config files is common and making the special path to keep the current personal config's behavior will introduce useless complexity.

Therefore, ESLint loads the plugins of the personal config (`~/.eslintrc`) from the HOME directory (`~/`).

If the plugins of the personal config conflict with `--plugin` option or something like, then ESLint throws.

#### Implementation

The implementation will be simple.

- Modify [ConfigArrayFactory](https://github.com/eslint/eslint/blob/62623f9f611a3adb79696304760a2fd14be8afbc/lib/cli-engine/config-array-factory.js) to load plugins relatively `basePath`. If `resolvePluginsRelativeTo` constructor option is present, it ignores the `basePath` and loads from the `resolvePluginsRelativeTo`. The `basePath` is:
  - the directory of `options.filePath` if the `create(data, options)` method is called. It's `cwd` if `options.filePath` is not present.
  - the directory of `filePath` if the `loadFile(filePath, options)` method is called.
  - the `directoryPath` if the `loadInDirectory(directoryPath, options)` method is called.
- Modify [the `mergePlugins` function in ConfigArray](https://github.com/eslint/eslint/blob/62623f9f611a3adb79696304760a2fd14be8afbc/lib/cli-engine/config-array/config-array.js#L163) to throw a clear error if a plugin conflict was found. The conflict is `sourceValue.filePath !== targetValue.filePath`. Both `sourceValue` and `targetValue` are [ConfigDependency](https://github.com/eslint/eslint/blob/62623f9f611a3adb79696304760a2fd14be8afbc/lib/cli-engine/config-array/config-dependency.js) objects, so the error object can have the importer's info as the template data easily.

#### The plugin uniqueness for different lint target files

After this proposal was implemented, ESLint guarantees the plugin uniqueness for each target file individually. Because it's the requirement for we use the rule ID such as `foo/a-rule`. On the other hand, ESLint doesn't guarantee the plugin uniqueness over multiple target files. For example, we can lint `packages/alice/index.js` and `packages/bob/index.js` with each version of the same plugin.

    <root>
    ├ node_modules/
    │ └ eslint/
    ├ packages/
    │ ├ alice/
    │ │ ├ node_modules/
    │ │ │ └ eslint-plugin-foo/  # (A)
    │ │ ├ index.js
    │ │ └ .eslintrc.js
    │ └ bob/
    │   ├ node_modules/
    │   │ └ eslint-plugin-foo/  # (B)
    │   ├ index.js
    │   └ .eslintrc.js
    └ .eslintrc.js

When ESLint lints `packages/alice/index.js`, it uses `packages/alice/node_modules/eslint-plugin-foo`. And when ESLint lints `packages/bob/index.js`, it uses `packages/bob/node_modules/eslint-plugin-foo`. Those are not related to each other.

However, only the following APIs require the plugin uniqueness over multiple target files.

- `CLIEngine#getRules()`
- `metadata.rulesMeta` of formatters

This proposal deprecates those APIs. The following sections describe this.

### ● It replaces the `CLIEngine#getRules()` method

This RFC replaces the `CLIEngine#getRules()` method by `CLIEngine#getRulesForFile(filePath)` because the `CLIEngine#getRules()` method requires the plugin uniqueness over multiple target files.

The `CLIEngine#getRulesForFile(filePath)` method is similar to `CLIEngine#getConfigForFile(filePath)`. ESLint loads the config files of the given target file then returns the `Map` object that contains the core rules and the plugin rules that the config files used.

Once we removed the `CLIEngine#getRules()` method, `CLIEngine` instances no longer need having the configs that the last call of `executeOnFiles()` used. This will simplify the state management of `CLIEngine`. As a side note, for this reason, [RFC40] removed the `getRules()` method from the new API.

#### About `LinterShell` class in [RFC40]

If [RFC40] is accepted, this proposal adds `LinterShell#getRulesForFile(filePath)`.

#### Removal plan

- It adds the runtime-deprecation of the `CLIEngine#getRules()` property in ESLint `7.0.0`.
- It removes the `CLIEngine#getRules()` property in the first major version after over a year from `7.0.0`.

If there are plugins that have the same name but different implementation, the `CLIEngine#getRules()` contains one of the implementations randomly. The runtime-deprecation message tells this fact.

### ● It replaces the `metadata.rulesMeta` property of formatters

This RFC replaces the `metadata.rulesMeta` property by `metadata.getRuleMeta(ruleId, filePath)` method because the `metadata.rulesMeta` property requires the plugin uniqueness over multiple target files.

The `metadata.getRuleMeta(ruleId, filePath)` method returns `cliEngine.getRulesForFile(filePath).get(ruleId)?.meta`.

Existing `metadata.rulesMeta[ruleId]` is replaced to `metadata.getRuleMeta(ruleId, filePath)`.

Once we removed the `metadata.rulesMeta` property, we can remove the deprecated `CLIEngine#getRules()` method (because API users require the deprecated `CLIEngine#getRules()` method to use the `metadata.rulesMeta` property). Then `CLIEngine` instances no longer need having the configs that the last call of `executeOnFiles()` used. This will simplify the state management of `CLIEngine`.

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
- https://github.com/eslint/rfcs/pull/40 - New: `LinterShell` Class Replacing `CLIEngine`
- https://github.com/eslint/rfcs/pull/45 - New: Formatter v2

[rfc32]: https://github.com/eslint/rfcs/pull/32
[rfc39]: https://github.com/eslint/rfcs/pull/39
[rfc40]: https://github.com/eslint/rfcs/pull/40
[rfc45]: https://github.com/eslint/rfcs/pull/45
[eslint/eslint#3458]: https://github.com/eslint/eslint/issues/3458
