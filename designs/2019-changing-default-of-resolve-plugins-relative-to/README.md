- Start Date: 2019-09-28
- RFC PR: https://github.com/eslint/rfcs/pull/39
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Changing the Default Value of `--resolve-plugins-relative-to`

## Summary

This RFC changes the default value of `--resolve-plugins-relative-to` CLI option to work on more variety environments.

## Motivation

We can use the config file which is in a common directory by `--config` option. In that case, it's intuitive to load plugins from the directory that contains the config file. But currently, we have to use `--resolve-plugins-relative-to` option along with `--config` option, or have to install required plugins into the current working directory. For example:

```
eslint . --config /home/me/.eslintrc.js --resolve-plugins-relative-to /home/me
```

```
eslint projects/foo --config projects/shared/.eslintrc.js --resolve-plugins-relative-to projects/shared
```

This behavior is not convenient.

## Detailed Design

### `--resolve-plugins-relative-to` CLI option

This RFC changes the default value of `--resolve-plugins-relative-to` CLI option.

1. If `--resolve-plugins-relative-to` CLI option is present, adopt it.
1. If `--config` CLI option is present, adopt the directory that contains the config file.
1. Otherwise, adopt the current working directory.

This setting is used in the whole eslint process. Therefore, every rule ID is identified as unique, so the `rulesMeta` of formatters works fine.

### `resolvePluginsRelativeTo` constructor option of CLIEngine

This RFC changes the default value of `resolvePluginsRelativeTo` option.

1. If `resolvePluginsRelativeTo` constructor option is present, adopt it.
1. If `configFile` constructor option is present, adopt the directory that contains the config file.
1. Otherwise, adopt the current working directory.

This setting is used in the whole `CLIEngine` instance. Therefore, every rule ID is identified as unique, so the `CLIEngine#getRules()` method works fine.

### About personal config file (`~/.eslintrc`)

This RFC doesn't change the plugin base path for the personal config file (`~/.eslintrc`) because the functionality has been deprecated in [RFC 32](https://github.com/eslint/rfcs/pull/32). ESLint still loads plugins from the current working directory if there is no project config file, even if there is the personal config file.

If people want to use the personal config file, use `--config` option (e.g., `eslint . --config ~/.eslintrc.json`). It should work intuitively.

### Special handling of `~/`

Bash replaces `~` by the user home directory, but this behavior is not compatible with other platforms. This is similar to that we cannot use glob patterns on every platform.

Therefore, this RFC adds handling of `~`. In the following constructor options, if the path was exactly `~` or started with `~/` (and `~\` as well on Windows), the `CLIEngine` class replaces the `~` by `require("os").homedir()`.

- `cacheLocation`
- `configFile`
- `cwd`
- `ignorePath`
- `resolvePluginsRelativeTo`

Notes:

- It doesn't handle `~username/` notation because Node.js doesn't provide the way that gets the home directory of other users.
- It doesn't handle `~` in config files because this is a fallback of stuff that some shells do.

## Documentation

- It needs an entry in the migration guide because of a breaking change.
- It updates the "[Command Line Interface](https://eslint.org/docs/user-guide/command-line-interface)" page to describe the `--config` option changes the default value of the `--resolve-plugins-relative-to` option.
- It updates the "[Command Line Interface](https://eslint.org/docs/user-guide/command-line-interface)" page to describe the `--config` option changes the default value of the `--resolve-plugins-relative-to` option.
- It updates the "[Node.js API](https://eslint.org/docs/developer-guide/nodejs-api)" page to describe the `configFile` constructor option changes the default value of the `resolvePluginsRelativeTo` constructor option on CLIEngine.

## Drawbacks

- The new logic is complex a bit. It increases maintenance cost.

## Backwards Compatibility Analysis

This RFC is a breaking change but has a small impact.

If people use `--config` option with the path to outside of the current working directory and don't use `--resolve-plugins-relative-to` option, and the referred config depends on the plugins of the current working directory, then this RFC breaks the workflow. (I guess it's a rare case)

## Alternatives

Nothing in particular.

## Related Discussions

- https://github.com/eslint/eslint/pull/11696 - New: add `--resolve-plugins-relative-to` flag
- https://github.com/eslint/eslint/issues/11914 - Personal config (`~/.eslintrc`) doesn't load global-installed configs/parsers/plugins
