- Repo: eslint/eslint
- Start Date: 2024-06-14
- RFC PR: https://github.com/eslint/rfcs/pull/120
- Authors: Nicholas C. Zakas

# Look Up Config Files From Linted File

## Summary

This proposal changes the lookup behavior for flat configuration files to match the behavior of eslintrc configuration lookup, in that the search begins from the file being linted rather than from the current working directory.

## Motivation

One of the significant changes from the eslintrc configuration system to the flat configuration system was the logic of looking up configuration files. In an effort to simplify configuration lookup, the decision was made to start the search from the current working directory, with the idea that most projects would contain only one configuration file. The intent was to reduce disk access, and in turn, speed up ESLint. While this approach worked well for most projects, shortly after ESLint v9.0.0 was released, we started hearing complaints from monorepo projects that they could not easily configure their repos using this approach.

We first offered that they should put a configuration file in the root of the repo and list out overrides for specific projects, but the feedback was that for large monorepos, this created a burden and prevented them from having everything related to a project inside of the project subdirectory.

Looking up configuration files from the current working directory also proved to be problematic whenever users were using ESLint without the CLI. Notably, IDEs don't necessarily have a directory that would logically map to the current working directory, and that meant needing to calculate something that would make ESLint work.

As a result, it became obvious that the modified configuration lookup strategy in flat config would not work in the long term, and we need to go back to looking up configuration files from the file being linted.

## Detailed Design

This proposal consists of the following changes:

1. Create a new `ConfigLoader` class that manages the lookup and caching of configurations
1. Create a new `LegacyConfigLoader` class with the same interface as `ConfigLoader` to encapsulate the current configuration lookup strategy (look up from the cwd)
1. Use a feature flag to switch between the two modes in the `ESLint` class

**Compatibility:** This proposal requires the flat config system and cannot be used with the eslintrc config system.

### The `ConfigLoader` interface

The following interface will be implemented twice, once as `ConfigLoader` to encapsulate the new behavior and once as `LegacyConfigLoader` to encapsulate the current behavior. This will allow the `ESLint` class to switch between these two modes easily and remove all configuration lookup-related functionality from the `ESLint` class.

```ts
interface ConfigLoader {

    /**
     * Searches the file system for the right config file to use based on the
     * absolute file or directory path.
     */
    findConfigFileForPath(fileOrDirPath): Promise<string|undefined>;

    /**
     * An asynchronous call that searches for a config file from the given
     * absolute file path and returns the config array for that path.
     */
    loadConfigArrayForFile(filePath): Promise<FlatConfigArray>;

    /**
     * An asynchronous call that searches for a config file from the given
     * absolute directory path and returns the config array for that path.
     */
    loadConfigArrayForDirectory(dirPath): Promise<FlatConfigArray>;

    /**
     * A synchronous call to retrieve already-cached configuration information.
     * Necessary for areas that must be synchronous and still need access to
     * config data.
     */
    getCachedConfigArrayForPath(fileOrDirPath): FlatConfigArray|undefined;
}
```

#### The `findConfigFileForPath()` Method

This methods returns the path to the config file for a given file or directory path. When used in `LegacyConfigLoader`, this method would search from the cwd and ignore the argument that was passed in. This replaces the [`findFlatConfigFile()` method](https://github.com/eslint/eslint/blob/455f7fd1662069e9e0f4dc912ecda72962679fbe/lib/eslint/eslint.js#L266-L271) that is currently in `lib/eslint/eslint.js`.

#### The `loadConfigArrayForFile()`,`loadConfigArrayForDirectory()` Methods

These methods behave similarly as the current [`ESLint#calculateConfigForFile()` method](https://github.com/eslint/eslint/blob/455f7fd1662069e9e0f4dc912ecda72962679fbe/lib/eslint/eslint.js#L1165-L1174) except that they return the `FlatConfigArray` instead of the config for the given file. All of the logic in `ESLint#calculateConfigForFile()` will be moved to the `ConfigLoader` class and the `ESLint` class will call the `ConfigLoader` method to provide this functionality. This requires moving the [`calculateConfigArray()` function](https://github.com/eslint/eslint/blob/455f7fd1662069e9e0f4dc912ecda72962679fbe/lib/eslint/eslint.js#L369) into `ConfigLoader` (as a private method).

In most of the `ESLint` class logic, `FlatConfigArray#getConfig()` will need to be replaced by `ConfigLoader#loadConfigArrayForFile()` to ensure that the file system is always searched to find the correct configuration.

It's necessary to return a `FlatConfigArray` because we [pass a `FlatConfigArray` to `Linter`](https://github.com/eslint/eslint/blob/455f7fd1662069e9e0f4dc912ecda72962679fbe/lib/eslint/eslint.js#L498) and also use it when we [filter out code blocks](https://github.com/eslint/eslint/blob/455f7fd1662069e9e0f4dc912ecda72962679fbe/lib/eslint/eslint.js#L511-L513). Because `Linter` is a synchronous API, we need to maintain the synchronous calls, and the easiest way to do that is to access the `FlatConfigArray` directly.

To maintain current behavior, `loadConfigArrayForFile()` throws an error if no config file is found; `loadConfigArrayForDirectory()` does not throw an error if a config file isn't found and instead returns just the default configuration, which is necessary to ensure we can traverse directories.

#### The `getCachedConfigArrayForPath()` Methods

This method checks the cache of already-read configuration information produced by `loadConfigArrayForFile()`/`loadConfigArrayForDirectory()` to return a `FlatConfigArray`. This is necessary for areas of the codebase that must be synchronous and don't need updated configuration information from disk, such as [`getOrFindUsedDeprecatedRules()`](https://github.com/eslint/eslint/blob/7c78ad9d9f896354d557f24e2d37710cf79a27bf/lib/eslint/eslint.js#L185).


### Core Changes

In order to make all of this work, we'll need to make the following changes in the core:

1. Create a new `lib/config/config-loader.js` file to contain both `ConfigLoader` and `LegacyConfigLoader`.
1. Move `findFlatConfigFile()`, `loadFlatConfigFile()`, `locateConfigFileToUse()`, and `calculateConfigArray()` from `lib/eslint/eslint.js` to `lib/config/config-loader.js`.
1. Create a new feature flag called `unstable_config_lookup_from_file` (once https://github.com/eslint/eslint/pull/18516 is merged).
1. Update `lib/eslint/eslint.js`:
    1. Create a new `ConfigLoader` based on the presence or absence of the flag. This should be used in the `ESLint#lintText()` and `ESLint#lintFiles()` methods.
    1. Update `ESLint#calculateConfigForFile()` to use the config loader.
    1. Update `ESLint#findConfigFile()` to use the config loader. To preserve backwards compatibility, when called without an argument, this method will start the config file search from the current working directory; it will also accept an argument, which is the path to search from.
    1. Update `getOrFindUsedDeprecatedRules()` to use the config loader.
1. Update `findFiles()` in `lib/eslint/eslint-helpers.js` to use the config loader. This also requires a change to the file system walking logic because `fswalk` filter functions are synchronous, but we'll need them to be asynchronous to use with the config loader. I plan on using [`humanfs`](https://github.com/humanwhocodes/humanfs/blob/main/docs/directory-operations.md#reading-directory-entries-recursively) (which I wrote).

### Rollout Plan

#### v9.x

* Add the `unstable_config_lookup_from_file` flag while the feature is in development.
* Once stabilized, retire the `unstable_config_lookup_from_file` flag and create the `v10_config_lookup_from_file` flag, which will let people opt-in to the breaking behavior.

#### v10.x

* Remove the `v10_config_lookup_from_file` flag and make this behavior the only available config lookup scheme.
* Remove `LegacyConfigLoader`.

## Documentation

The primary place to update is the section on [configuration file resolution](https://eslint.org/docs/latest/use/configure/configuration-files#configuration-file-resolution).

## Drawbacks

1. **Performance.** There may be a performance impact to this change, as we'll be hitting the file system more frequently and switching from sync to async functions in certain places. This is unavoidable to implement this proposal.
1. **More flat config changes.** With some of the criticisms around flat config, we may get some blowback for making another change while most of the ecosystem hasn't yet converted. Hopefully, though, this change will have minimal impact on most users.

## Backwards Compatibility Analysis

While a breaking change, this proposal will have a minimal impact on most users of flat config. Because linted files typically exist under the current working directory, and the configuration file is either in the current working directory or an ancestor directory, projects that currently use a flat config file will likely not see any difference in the way ESLint works. The same config file will be found during the resolution process.

This is breaking primarily because users might have other config files deeper in the project directory hierarchy that might be found instead of the one found by searching from the cwd.

## Alternatives

* **Lookup strategy CLI/`ESLint` Option.** Instead of migrating towards a new config lookup strategy, we could also let people opt in via a command line argument and associated option to the `ESLint` constructor. Something like `--config-lookup-from file` with the default being `--config-lookup-from cwd`. The downsides of this approach are that it requires people to opt-in (at least until we can switch the default) and we'll be stuck maintaining two different config lookup strategies.
* **Configurable lookup strategy.** We could also allow people to specify a lookup strategy, or even a custom implementation of a lookup strategy, using a configuration file (either in `eslint.config.js` or as a distinct file). This would be more complicated to implement and maintain.

## Open Questions

N/A

## Help Needed

N/A

## Frequently Asked Questions

### What will the behavior be of `-c` when `*_config_lookup_from_file` is enabled?

The same as the current implementation: The `-c` option completely overrides which config file to use and there will be no config lookup performed.

### If I'm linting `./subdir/foo.js` where `./subdir/eslint.config.js` exists, why is `./eslint.config.js` loaded?

Here's the structure in question:

```
/usr/tmp/
├── eslint.config.js
└── subdir/
    ├── foo.js
    └── eslint.config.js
```

Because ignores are handled in `eslint.config.js`, we need to look at `./eslint.config.js` to ensure that `subdir` isn't being ignored.

### What happens if there's a `eslint.config.js` file with an ignore pattern of `subdir` in the parent directory `subdir`?

Here's the structure in question:

```
/usr/tmp/
├── eslint.config.js    <-- ignores: ["subdir"]
└── subdir/
    ├── foo.js
    └── eslint.config.js
```

In this scenario, it matters how ESLint is called:

1. `eslint .` - ESLint first reads `./eslint.config.js`, which says to ignore `subdir`. Because ESLint must recursively traverse `.`, it skips `subdir`. Only `./eslint.config.js` is linted.
1. `eslint subdir` - ESLint first reads `./subdir/eslint.config.js` and lints `./subdir/foo.js` and `./subdir/eslint.config.js`. The `./eslint.config.js` file is never read because config lookup starts and stops in `subdir` due to the presence of `./subdir/eslint.config.js`.
1. `eslint subdir/foo.js` - ESLint first reads `./subdir/eslint.config.js` and lints `./subdir/foo.js`.

A more complicated example:

```
/usr/tmp/
├── eslint.config.js       <-- ignores: ['**/subsubdir1']
└── subdir/
    ├── eslint.config.js   <-- ignores: ['**/subsubdir2'] 
    ├── subsubdir1/
    │   └── file.js
    └── subsubdir2/
        └── file.js
```

In this scenario:

1. `eslint .` - ESLint first reads `./eslint.config.js`, which has an `ignores` pattern that doesn't match any of the child directories, so ESLint traverses into each child directory. Once in `subdir`, ESLint reads `./subdir/eslint.config.js`, which says to ignore `subsubdir2`, so ESLint skips traversing into `subsubdir2` but still traverses into `subsubdir1`. ESLint lints `./eslint.config.js`, `./subdir/eslint.config.js`, and `./subdir/subsubdir1/file.js`.
1. `eslint subdir` - ESLint first reads `./subdir/eslint.config.js` which says to ignore `subsubdir2`, so ESLint skips traversing into `subsubdir2` but still traverses into `subsubdir1`. ESLint lints `./subdir/eslint.config.js`, and `./subdir/subsubdir1/file.js`.

### What happens when I run `eslint *`?

Consider this example:

```
/usr/tmp/
└── subdir/
    ├── eslint.config.js
    ├── subsubdir1/
    │   └── file.js
    └── subsubdir2/
        └── file.js
```

When you run `eslint *`, it's the same as if you ran `eslint subdir`, as it matches all immediate children of `.`. If the immediate children include `node_modules` and `.git`, those will still be ignored because they are default ignores in ESLint. All other subdirectories will be traversed but will only be linted if there's an `eslint.config.js` file present, as is the case with `./subdir` in this example.

## Related Discussions

* https://github.com/eslint/eslint/discussions/18574
* https://github.com/eslint/eslint/discussions/16960
* https://github.com/eslint/eslint/discussions/18353
* https://github.com/eslint/eslint/discussions/18101
* https://github.com/eslint/eslint/discussions/18456
* https://github.com/eslint/eslint/discussions/17853
* https://github.com/eslint/eslint/discussions/16202
