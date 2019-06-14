- Start Date: 2019-06-14
- RFC PR: https://github.com/eslint/rfcs/pull/27
- Authors: Toru Nagashima &lt;https://github.com/mysticatea&gt;

# Config Tester

## Summary

Providing `ConfigTester` API that tests shareable configs and plugin's preset configs.

## Motivation

Currently, we don't provide stuff to test shareable configs. This means the community doesn't have an easy way to check if their config is good or not.

- Is the config structure valid?
- Does every rule exist?
- Does every rule have valid options?
- Aren't there any deprecated rules?
- Are there plugin settings of configured rules?
- Will this config work after publish?
    - Does `package.json` have parsers, plugins, and extended configs?

## Detailed Design

This RFC adds `ConfigTester` API to check configs.

```js
const { ConfigTester } = require("eslint")

// Instantiate the tester.
const tester = new ConfigTester({
    projectRoot: process.cwd(),
    ignoreMissingRules: false,
    ignoreMissingDependencies: false
})

// Verify a shareable config.
tester.run("eslint-config-xxxx", require("../lib/index.js"))

// Or verify plugin's preset configs.
tester.run("eslint-plugin-xxxx/recommended", require("../lib/configs/recommended.js"))
tester.run("eslint-plugin-xxxx/opinionated", require("../lib/configs/opinionated.js"))
```

### § `ConfigTester(options)` constructor

#### Parameters

The constructor has an optional parameter.

Name | Description
:----|:-----------
`options.projectRoot` | Default is `process.cwd()`. The path to the project root. The root should contain `package.json`.
`options.ignoreMissingRules` | Default is `false`. If `true` then the tester ignores missing rules. The missing rules mean the rules that ESLint or a plugin defined but not configured.
`options.ignoreMissingDependencies` | Default is `false`. If `true` then the tester ignores wrong dependency definition (`dependencies`/`peerDependencies`).

### § `tester.run(displayName, config)` method

Validates a config data.

#### Parameters

Name | Description
:----|:-----------
`displayName` | Required. The display name of this check.
`config` | Required. The config object to check.

#### Behavior

Similarly to `RuleTester`, `ConfigTester` defines tests by `describe` and `it` global variables. Then it does:

1. Validate the config object has the valid scheme ([validateConfigScheme(config, source)](https://github.com/eslint/eslint/blob/aef8ea1a44b9f13d468f48536c4c93f79f201d9b/lib/shared/config-validator.js#L282)).
1. Parse the config to `ConfigArray` with [ConfigArrayFactory#create(config, options)](https://github.com/eslint/eslint/blob/aef8ea1a44b9f13d468f48536c4c93f79f201d9b/lib/cli-engine/config-array-factory.js#L360).
    - `cwd` and `resolvePluginsRelativeTo` options of the factory are `options.projectRoot`.
    - if the `name` field of `${options.projectRoot}/package.json` is an ESLint plugin name, it loads the `main` field's file then adds it to `additionalPluginPool`.
1. Validate the content ([validateConfigArray(configArray)](https://github.com/eslint/eslint/blob/aef8ea1a44b9f13d468f48536c4c93f79f201d9b/lib/shared/config-validator.js#L322)).
1. Report non-existence rules.
    - Because `validateConfigArray(configArray)` ignores non-existence rules.
    - Configured plugin's rules are in `configArray.pluginRules`.
1. Report deprecated rules if it's `warn` or `error`.
    - Check `meta.deprecated` in core rules and `configArray.pluginRules`.
1. If `options.ignoreMissingRules` was not `true`, check whether the config contains the settings of all rules.
1. If `options.ignoreMissingDependencies` was not `true`, check whether `${options.projectRoot}/package.json` contains the configured parser, plugins, and shareable configs.
    - If `parser` or `extends` were a file path except `node_modules/**`, the file should be published; check `.npmignore` and `package.json`'s `lib` field.
    - If `parser` or `extends` were a package or a file path to `node_modules/**`, the package should be in `dependencies` or `peerDependencies`.
    - `plugins` should be in `peerDependencies` or `name`.

## Documentation

- [Node.js API](https://eslint.org/docs/developer-guide/nodejs-api) page should describe the new `ConfigTester` API.
- [Creating a Shareable Config](https://eslint.org/docs/developer-guide/shareable-configs#creating-a-shareable-config) section should note the tester.
- [Configs in Plugins](https://eslint.org/docs/developer-guide/working-with-plugins#configs-in-plugins) section should note the tester.

## Drawbacks

- If people can write the config with no mistakes, this tester may not be needed.

## Backwards Compatibility Analysis

- This is not a breaking change. It just adds a new API.

## Alternatives

- https://www.npmjs.com/package/eslint-find-rules - we can check missing rules and deprecated rules with this package.

## Related Discussions

- https://github.com/eslint/eslint/issues/10289
