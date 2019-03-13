# Config File Improvements: Plugin Renaming

## Summary

This proposal adds the feature that renames plugins in config files. This enhancement will be the official successor of `--rulesdir` option. And, this solves the important pain of the ecosystem as making shareable configs being able to have plugins.

## Motivation

People cannot define local rules with config files (rather than `--rules` CLI option). The ecosystem has hacked this problem by several plugins to solve such as `eslint-plugin-rulesdir`, `eslint-plugin-local`, etc. However, the official way is great.

And if a shareable config depends on plugins, people have to install the plugins manually. The combination of this enhancement and `require.resolve()` function makes shareable configs being able to have plugins, so solves the pain!

## Detailed Design

> Proof of Concept (implementation): https://github.com/eslint/eslint/tree/proof-of-concept/config-array-in-eslintrc

This proposal enhances [there](README.md#plugin-renaming).

If `plugins` property of a configuration was an object, `ConfigArrayFactory` loads the value of each property as a plugin then store the loaded plugin as the key of the property. Here, `["foo", "bar"]` is equivalent to `{ "foo": "foo", "bar": "bar" }`. The object form supports file paths.

> [lib/lookup/config-array-factory.js#L807-L828](https://github.com/eslint/eslint/blob/fedb0293ef8fb3e2e17d88bdfeb5e5cfb725a282/lib/_lookup/config-array-factory.js#L807-L828) in PoC.

<table><td>
💡 <b>Example</b>:
<pre lang="jsonc">
{
    "plugins": {
        "foo": "foo", // Or long version "eslint-plugin-foo" is OK as well.
        "abc": "xyz",
        "local": "./tools/eslint-rules/index.js"
    },
    "rules": {
        "foo/a-rule": "error",
        "abc/a-rule": "error",
        "local/a-rule": "error"
    }
}
</pre>
is loaded as:
<pre lang="jsonc">
[
    {
        "name": ".eslintrc.json",
        "filePath": ".eslintrc.json",
        "plugins": {
            // This `foo` is not renamed because property key and `id` in value is same.
            "foo": {
                "definition": { ... },
                "id": "foo",
                "filePath": "node_modules/eslint-plugin-foo/index.js",
                "importerPath": ".eslintrc.json"
            },
            // This `foo` is renamed because property key and `id` in value is different.
            "abc": {
                "definition": { ... },
                "id": "xyz",
                "filePath": "node_modules/eslint-plugin-xyz/index.js",
                "importerPath": ".eslintrc.json"
            },
            // This `foo` is renamed because property key and `id` in value is different.
            "local": {
                "definition": { ... },
                "id": "./tools/eslint-rules/index.js",
                "filePath": "tools/eslint-rules/index.js",
                "importerPath": ".eslintrc.json"
            }
        },
        "rules": {
            "foo/a-rule": "error",
            "abc/a-rule": "error",
            "local/a-rule": "error"
        }
    }
]
</pre>
</td></table>

### Conflict handling

If a plugin was renamed, it can be different content to other declarations even if it has the same plugin ID.
Therefore, if a config file had a renamed plugin and another config file included the same plugin ID regardless of renamed or not, ESLint throws "plugin conflict" error.

ESLint loads plugins which are not renamed relative to CWD (by [#7]), it doesn't conflict because of unique if there are no renamed plugins.

### Using two shareable configs which have the same plugin ID with renaming

This section depends on two enhancements [Array Config](minor-01-array-config.md).

We can provide a utility to use conflicted two shareable configs.
The utility does load a given shareable config with renaming conflicted plugins.

I assume that the utility is used only in the pretty rare case.

```js
// .eslintrc.js
const config = require("@eslint/config")

module.exports = [
    // Rename conflicted plugins while loading.
    // It renames the prefixes of rules, environments, and processors at the same time.
    // It resolves the relative paths in `parser` and `plugins` to work file on this file.
    config.withConvert("eslint-config-foo", {
        relativeTo: __filename,
        mapPluginName: id => `foo::${id}`
    }),

    "eslint-config-bar",

    {
        root: true,
        rules: { ... }
    }
]
```

<table><td>
💡 <b>Example</b>:
<pre lang="js">
// eslint-config-foo
module.exports = {
    plugins: {
        "a-plugin": require.resolve("eslint-plugin-a-plugin")
    },
    parser: "./lib/parser",
    env: {
        "a-plugin/env": true
    },
    rules: {
        eqeqeq: "error",
        "a-plugin/x": "error",
        "a-plugin/y": "error"
    }
}
</pre>
<code>config.withConvert(...)</code> method with the above config returns as:
<pre lang="jsonc">
[
    {
        "plugins": {
            // Renaming with absolute path.
            "foo::a-plugin": "/path/to/node_modules/eslint-config-foo/node_modules/eslint-plugin-a-plugin/index.js"
        },
        // absolute path.
        "parser": "/path/to/node_modules/eslint-plugin-foo/lib/parser.js",
        "env": {
            "foo::a-plugin/env": true
        },
        "rules": {
            "eqeqeq": "error",
            "foo::a-plugin/x": "error",
            "foo::a-plugin/y": "error"
        }
    }
]
</pre>
</td></table>

The `config.withConvert(request, options)` method loads `extends` property and flattens `extends` property and `overrides` property recursively. Then it converts all plugin names in the configs.

With [`extends` in `overries`](minor-02-extends-in-overrides.md) enhancement, it uses nested `overrides` properties to express logical AND conditions.

### Deprecating `--rulesdir` CLI option

This feature allows us to define rules in config files in local.
Therefore, now we don't have to add `--rulesdir` CLI option for each `eslint` command call, we can deprecate `--rulesdir` CLI option.

## Documentation

This enhancement should be documented in "Configuring ESLint" page.

## Drawbacks

I think no problem.

## Backwards Compatibility Analysis

If people depend on the behavior that ESLint throws an error if they give an object to `plugins` property, this enhancement breaks that. However, I believe that we don't need to worry.

## Alternatives

- [#9] is the alternative. But double duplicate features cause confusion for the ecosystem. For newcomers, a mix of articles about two config systems makes hard to understand ESLint. For non-English users, the official document is far. Also, this proposal has information that a plugin name was renamed or not, so it helps to address [Robustness Guarantee].
- [#14] is a different idea to handle local plugins. I think people have gotten used to key-value form via JavaScript object literals.

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [#14]
- [#9]
- [#7]

[#14]: https://github.com/eslint/rfcs/pull/14
[#9]: https://github.com/eslint/rfcs/pull/9
[#7]: https://github.com/eslint/rfcs/pull/7
[Guarantee Robustness]: https://gist.github.com/not-an-aardvark/169bede8072c31a500e018ed7d6a8915
