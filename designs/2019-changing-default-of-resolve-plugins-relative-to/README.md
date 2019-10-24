- Start Date: 2019-09-28
- RFC PR: https://github.com/eslint/rfcs/pull/39
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Changing the Default Value of `--resolve-plugins-relative-to`

## Summary

This RFC changes the default value of `--resolve-plugins-relative-to` CLI option to work on more variety environments.

## Motivation

The current behavior is not useful for some environments.

### When using a config file from outside of the project

People can use the config file which is in a common directory by `--config` option. In that case, it's intuitive to load plugins from the directory that contains the config file. But currently, people have to use `--resolve-plugins-relative-to` option or install required plugins into the current working directory.

```
eslint . --config /home/me/.eslintrc.js --resolve-plugins-relative-to /home/me
```

```
eslint projects/foo --config projects/shared/.eslintrc.js --resolve-plugins-relative-to projects/shared
```

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

This RFC changes the behavior when `--resolve-plugins-relative-to` CLI option was not presented.

The logic to decide `--resolve-plugins-relative-to` value will be the following steps:

1. If `--resolve-plugins-relative-to` CLI option is present, adopt it.
1. If `--config` CLI option is present, adopt the directory that contains the config file.
1. Otherwise, it's variable for each lint target file.<br>
   Adopt the directory which contains the closest config file from the lint target file.<br>
   This step is skipped if `--no-eslintrc` CLI option is present.
1. Adopt the current working directory if all of the above didn't match.

For example:

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

- With `eslint . --resolve-plugins-relative-to packages/zeta`<br>
  It's `pckages/zeta` always.
- With `eslint . --config /home/me/my-config.js`<br>
  It's `/home/me` always.
- With `eslint .`
  - When ESLint verifies `packages/foo/test.js`, it's `packages/foo` because the closest config file is `packages/foo/.eslintrc.yml`.
  - When ESLint verifies `packages/bar/test.js`, it's `.` because the closest config file is `./.eslintrc.yml`.

### About personal config file (`~/.eslintrc`)

This RFC doesn't change the plugin base path for the personal config file (`~/.eslintrc`) because the functionality has been deprecated in [RFC 32](https://github.com/eslint/rfcs/pull/32). ESLint still loads plugins from the current working directory if there is no project config file, even if there is the personal config file.

If people want to use the personal config file, use `--config` option (e.g., `eslint . --config ~/.eslintrc.json`). It should work intuitively.

### Implementation

The implementation appears in `CascadingConfigArrayFactory` class and `ConfigArrayFactory` class.

**PoC:** [eslint/eslint#poc-improve-resolve-plugins-relative-to](https://github.com/eslint/eslint/compare/master...poc-improve-resolve-plugins-relative-to)

- It adds the `pluginBasePath` option into the public methods of `ConfigArrayFactory`. Those methods load plugins from the location of the option. If the option is not present, load plugins from the location of constructor options.
- It updates the `CascadingConfigArrayFactory` behavior. The factory determines the plugin base path while finding config files. The factory traverses directories in the order from the leaf to the root. When the first config file was found, it determines the current traversing directory as the plugin base path.<br>
  There is the cache of the plugin base path for each directory and all cache keys of configs contain the plugin base path to avoid mix.

## Documentation

- It needs an entry in the migration guide because of a breaking change.
- It updates the "[Configuring Plugins](https://eslint.org/docs/user-guide/configuring#configuring-plugins)" section to describe where ESLint loads plugins from.

## Drawbacks

- The new logic is complex a bit. It increases maintenance cost.

## Backwards Compatibility Analysis

This RFC is a breaking change but has a small impact.

If people use `--config` option with the path to outside of the current working directory and don't use `--resolve-plugins-relative-to` option, and the referred config depends on the plugins of the current working directory, then this RFC breaks the workflow. (I guess it's a rare case)

Otherwise, this RFC moves the plugin base path to subdirectories, but Node.js can find packages from ancestor directories.

Therefore, this RFC has a small impact.

## Alternatives

Nothing in particular.

## Related Discussions

- https://github.com/eslint/eslint/pull/11696 - New: add `--resolve-plugins-relative-to` flag
- https://github.com/eslint/eslint/issues/11914 - Personal config (`~/.eslintrc`) doesn't load global-installed configs/parsers/plugins
- https://github.com/eslint/eslint/issues/12019 - Allow to specify plugin resolution in config
- https://github.com/microsoft/vscode-eslint/issues/696 - ESLint fails to load plugins when using ESLint 6.x
- https://github.com/microsoft/vscode-eslint/issues/708 - Autodetect and propose working directories in mono repositories
