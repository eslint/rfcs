- Start Date: 2019-06-27
- RFC PR: https://github.com/eslint/rfcs/pull/28
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Personal Config Loads Global-Installed Packages

## Summary

This RFC allows the personal config to load also global-installed packages (shareable configs, parsers, and plugins).

## Motivation

ESLint uses the config file in home directory (i.e. `~/.eslintrc.*`) if there are no config files in the current project directory. It's called personal config.

The personal config is not tied to any projects, but currently, it cannot depend on global-installed packages (shareable configs, parsers, and plugins). Instead, people have to install packages with local installation on the home directory. However, this is not common practice to address shared packages which are not tied to an arbitrary project. We have usually used `npm install --global` for such a package.

This RFC allows the personal config to load also global-installed packages.

## Detailed Design

When ESLint loads the personal config, if it could not resolve dependencies in the usual way, then it finds the missing dependencies from the global-installed packages. In this, the usual way is the way that [RFC 7](https://github.com/eslint/rfcs/tree/master/designs/2018-simplified-package-loading) defined.

This is not the same way as ESLint 5. This RFC handles the personal config in special regardless of where ESLint was installed.

### § With `--resolve-plugins-relative-to` CLI option

If `--resolve-plugins-relative-to` CLI option was given, ESLint doesn't find plugin packages from global. All plugins must be in the specified location.

### § With `--plugins` CLI option

If `--plugins` CLI option was given and ESLint have to load the personal config, it throws a fatal error ("'--plugins' option cannot use with the personal config") because it causes to mix local-installed plugins and global-installed plugins.

If both `--resolve-plugins-relative-to` CLI option and `--plugins` CLI option was given, ESLint doesn't throw the fatal error.

(This limitations came from implementation details. CLI options are loaded in the different phase from config files and it will be reused with every config file. Therefore, it's inconvenient if the result of CLI options is variable for each config file.)

### § With `--config` CLI option

If `--config` CLI option was given, ESLint will never use the personal config. It works as is.

### § With `baseConfig` CLIEngine option

If `baseConfig` option was given and ESLint have to load the personal config, it throws a fatal error as same as with `--plugins` option.

### § Implementation

1. Make the methods of `ConfigArrayFactory` having `findDependenciesInGlobal` option
1. Make `CascadingConfigArrayFactory` loading the personal config with `findDependenciesInGlobal` option

#### 1. Make the methods of `ConfigArrayFactory` having `findDependenciesInGlobal` option

```ts
class ConfigArrayFactory {
    create(configData, { filePath, name, parent, findDependenciesInGlobal })
    loadFile(filePath, { name, parent, findDependenciesInGlobal })
    loadInDirectory(directoryPath, { name, parent, findDependenciesInGlobal })
}
```

- If `findDependenciesInGlobal` was `true`, then `_loadExtendedShareableConfig` method and `_loadParser` method find the package in global if the package was not found in local.
- If `findDependenciesInGlobal` was `true` and the `resolvePluginsRelativeTo` of constructor options was `undefined`, then `_loadPlugin` method finds the package in global if the package was not found in local.
- It uses [resolve-global](https://github.com/sindresorhus/resolve-global) package to resolve global packages.

#### 2. Make `CascadingConfigArrayFactory` loading the personal config with `findDependenciesInGlobal` option

In [lib/cli-engine/cascading-config-array-factory.js#L372-L384](https://github.com/eslint/eslint/blob/e5f1ccc9e2d07ad0acf149027ffc382021d54da1/lib/cli-engine/cascading-config-array-factory.js#L372-L384),

- If CLI option's config array has plugins, it throws an exception.
- If baseConfig's config array has any config elements, it throws an exception.
- It enables the `findDependenciesInGlobal` option of `ConfigArrayFactory`.

## Documentation

- The [Configuring ESLint](https://eslint.org/docs/user-guide/configuring) page should describe the personal config finds dependencies in global as well.

## Drawbacks

- We can use [resolve-global](https://github.com/sindresorhus/resolve-global) package to resolve global packages, but it's not reliable. Probably it will be hard to debug if problems happened.
- We don't recommend using global installation, but this feature may encourage it.

## Backwards Compatibility Analysis

This is a breaking change technically. The `--plugins` CLI option and `baseConfig` CLIEngine option no longer work well with personal configs. But I believe the impact of the change is pretty limited.

## Alternatives

- Deprecating the personal config in favor of making configs in each project. I'm not sure if it's feasible.
- Being as is. I.e.., people have to use `npm install` in the home directory, instead of `npm install --global`. This is simple to us, but we have made an original practice to manage shared npm packages.

## Related Discussions

- https://github.com/eslint/rfcs/pull/7
- https://github.com/eslint/eslint/issues/11914
