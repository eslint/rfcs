# Config File Improvements: Plugin Resolution Change

## Summary

This proposal changes the plugin resolution strategy to resolve plugins relative to the config files. This change will solve the important pain for shareable config maintainers.

## Motivation

Currently, if a shareable config depends on plugins, people have to install the plugins manually. This is an important pain for the ecosystem.

## Detailed Design

> Proof of Concept (implementation): https://github.com/eslint/eslint/tree/proof-of-concept/config-array-in-eslintrc

This proposal enhances [there](README.md#plugin-resolution-change).

Currently, ESLint loads three stuff from config files.

1. other config files (`extends` property)
1. parsers (`parser` property)
1. plugins (`plugins` property)

In those, only plugins are special. ESLint resolves plugins relative to the location ESLint was installed, but ESLint resolves other stuff relative to the config file.

This RFC changes this strategy to that ESLint resolves all of the three consistently relative to the config file.

This change introduces a new problem that a plugin can be loaded from two different locations for some situations. Therefore, when `ConfigArray#extractConfig(filePath)` method determines the final config, if a plugin had been loaded from different locations, it throws "plugin conflict" error.

> [lib/lookup/config-array.js#L76-L88](https://github.com/eslint/eslint/blob/153640180a8944af3a1c488462ed30d0c215f5ed/lib/_lookup/config-array.js#L76-L88) in PoC.

### Save people from the "plugin conflict" error

This section depends on two enhancements [Array Config](minor-01-array-config.md) and [Plugin Renaming](minor-03-plugin-renaming.md).

We provide a utility to load shareable configs with renaming problematic plugins.
I assume that the `@eslint/config` package is used only in the rare case.

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
    plugins: ["a-plugin"],
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

## Documentation

This change needs migration guide because of a breaking change.

- If a shareable config and your package, or two shareable configs, depend on the incompatible versions of the same plugin, a package manager fails to dedupe the plugin, then ESLint throws "plugin conflict" error. If you encouhtered the "plugin conflict" error, please try to solve the incompatible version problem. If you cannot solve that, use `@eslint/config` package to tweak your configuration.

## Drawbacks

- "conflict error" may cause new confusion.
- It can be broken without major version of shareable configs because of lacking [Robustness Guarantee].

## Backwards Compatibility Analysis

Because this is after [#7], the effect of the plugin resolution change is limited.

If people are using multiple shareable configs which depend on plugins via `dependencies` field of `package.json`, a package manager may fail to dedupe the plugins. However, most shareable configs are using `peerDependencies` to depend on plugins currently.

## Alternatives

- [#9] is the alternative. But double duplicate features cause confusion for the ecosystem. For newcomers, a mix of articles about two config systems makes hard to understand ESLint. For non-English users, the official document is far.
- [#5] and [#14] are alternatives about the plugin resolution change. Those are more strict, but those need some complex logic.

## Open Questions

- <a href="#q-robustness-guarantee" id="robustness-guarantee">❓</a> Should we keep [Robustness Guarantee] for the plugin conflict error?<br>
  As @not-an-aardvark's gist described, if a shareable config pinned a plugin version in a patch version, it may disable dedupe of a plugin manager, then it may cause breaking user's builds.<br>
  Should we address this problem?

## Frequently Asked Questions

-

## Related Discussions

- [#14]
- [#9]
- [#7]
- [#5]
- [eslint/eslint#3458]
- [eslint/eslint#9505]
- [eslint/eslint#9897]
- [eslint/eslint#10643]

[#14]: https://github.com/eslint/rfcs/pull/14
[#9]: https://github.com/eslint/rfcs/pull/9
[#7]: https://github.com/eslint/rfcs/pull/7
[#5]: https://github.com/eslint/rfcs/pull/5
[eslint/eslint#3458]: https://github.com/eslint/eslint/issues/3458
[eslint/eslint#9505]: https://github.com/eslint/eslint/issues/9505
[eslint/eslint#9897]: https://github.com/eslint/eslint/issues/9897
[eslint/eslint#10643]: https://github.com/eslint/eslint/issues/10643
[Robustness Guarantee]: https://gist.github.com/not-an-aardvark/169bede8072c31a500e018ed7d6a8915
