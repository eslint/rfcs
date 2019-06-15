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
const tester = new ConfigTester()
const options = {
    ignoreDeprecatedRules: false,
    ignoreDisabledUnknownRules: false,
    ignoreMissingDependencies: false,
    ignoreMissingRules: false,
}

// Verify a shareable config (a path to the target file).
tester.run("index.js", options)
tester.run("es5.js", options)
tester.run("es2015.js", options)

// Or verify plugin's preset configs (a config name in the plugin).
tester.run("base", options)
tester.run("recommended", options)
tester.run("opinionated", options)
```

### § `ConfigTester(projectRoot)` constructor

Instantiate a new `ConfigTester` instance.

#### Parameters

The constructor has an optional parameter.

Name | Description
:----|:-----------
`projectRoot` | Default is `process.cwd()`. The path to the project root. The root should contain `package.json`.

#### Behavior

The tester reads `` `${projectRoot}/package.json` `` to use in `run()` method.

<table><tr><td>
<b>⏩ PoC</b>: <a href="https://github.com/eslint/eslint/blob/2fb21b5dd52c81fe3c93cce0eb5fda3bf7789da0/lib/config-tester/config-tester.js#L60-L66">lib/config-tester/config-tester.js#L60-L66</a>
</td></tr></table>

### § `tester.run(targetName, options)` method

Validates a config data.

#### Parameters

Name | Description
:----|:-----------
`targetName` | Required. If this package was a plugin, this is a config name of the plugin. Otherwise, this is a path to a file (relative from `projectRoot`).
`options.ignoreDeprecatedRules` | Default is `false`. If `true` then the tester ignores deprecated rules.
`options.ignoreDisabledUnknownRules` | Default is `false`. If `true` then the tester ignores unknown rules if the rule was configured as `0` (`"off"`).
`options.ignoreMissingRules` | Default is `false`. If `true` then the tester ignores missing rules. The missing rules mean the rules that ESLint or a plugin defined but not configured.
`options.ignoreMissingDependencies` | Default is `false`. If `true` then the tester ignores wrong dependency definition (`dependencies`/`peerDependencies`).

#### Behavior

Similarly to `RuleTester`, `ConfigTester` defines tests by `describe` and `it` global variables. Then it does:

1. Validate the config object has the valid scheme with `validateConfigSchema()`.
    <table><tr><td>
    <b>🔗PoC</b>: <a href="https://github.com/eslint/eslint/blob/2fb21b5dd52c81fe3c93cce0eb5fda3bf7789da0/lib/config-tester/config-tester.js#L244-L248">lib/config-tester/config-tester.js#L244-L248</a>
    </td></tr></table>
1. Validate the config content with `validateConfigArray()`.
    <table><tr><td>
    <b>🔗PoC</b>: <a href="https://github.com/eslint/eslint/blob/2fb21b5dd52c81fe3c93cce0eb5fda3bf7789da0/lib/config-tester/config-tester.js#L250-L265">lib/config-tester/config-tester.js#L250-L265</a>
    </td></tr></table>
1. Report non-existence rules.
    - Because `validateConfigArray(configArray)` ignores non-existence rules.
    - Configured plugin's rules are in `configArray.pluginRules`.
    - If `ignoreDisabledUnknownRules` option was `true` and non-existence rule's severity was `"off"`, the tester ignores it.
    <table><tr><td>
    <b>🔗PoC</b>: <a href="https://github.com/eslint/eslint/blob/2fb21b5dd52c81fe3c93cce0eb5fda3bf7789da0/lib/config-tester/config-tester.js#L267-L299">lib/config-tester/config-tester.js#L267-L299</a>
    </td></tr></table>
1. Report deprecated rules.
    - If `ignoreDeprecatedRules` option was `true`, the tester skips this step.
    - If the rule severity was `"off"`, the tester ignores it.
    - Check `meta.deprecated` in both core rules and `configArray.pluginRules`.
    <table><tr><td>
    <b>🔗PoC</b>: <a href="https://github.com/eslint/eslint/blob/2fb21b5dd52c81fe3c93cce0eb5fda3bf7789da0/lib/config-tester/config-tester.js#L301-L338">lib/config-tester/config-tester.js#L301-L338</a>
    </td></tr></table>
1. Check whether the config congiures all rules.
    - If `ignoreMissingRules` option was `true`, the tester skips this step.
    - This step lets people know about new rules.
    <table><tr><td>
    <b>🔗PoC</b>: <a href="https://github.com/eslint/eslint/blob/2fb21b5dd52c81fe3c93cce0eb5fda3bf7789da0/lib/config-tester/config-tester.js#L340-L363">lib/config-tester/config-tester.js#L340-L363</a>
    </td></tr></table>
1. Check whether `${options.projectRoot}/package.json` contains the configured parser, plugins, and shareable configs.
    - If `ignoreMissingDependencies` option was `true`, the tester skips this step.
    - If `parser` or `extends` were a file path except `node_modules/**`, the file should be published; check `.npmignore` and `package.json`'s `lib` field.
    - If `parser` or `extends` were a package or a file path to `node_modules/**`, the package should be in `dependencies` or `peerDependencies`.
    - `plugins` should be in `peerDependencies` or `name`.
    <table><tr><td>
    <b>🔗PoC</b>: <a href="https://github.com/eslint/eslint/blob/2fb21b5dd52c81fe3c93cce0eb5fda3bf7789da0/lib/config-tester/config-tester.js#L365-L420">lib/config-tester/config-tester.js#L365-L420</a>
    </td></tr></table>

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
