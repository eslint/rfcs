# Config File Improvements: Plugin Renaming

## Summary

This proposal adds the feature that renames plugins in config files. This enhancement will be the official successor of `--rulesdir` option. Also, this enhancement is required to [save people from the "plugin conflict" error](major-02-plugin-resolution-change.md#save-people-from-the-plugin-conflict-error).

## Motivation

People cannot define local rules with config files (rather than `--rules` CLI option). The ecosystem has hacked this problem by several plugins to solve such as `eslint-plugin-rulesdir`, `eslint-plugin-local`, etc. However, the official way is great.

Also, to solve "plugin conflict" error, a utility that renames the plugins in shareable configs would be useful. This enhancement is required to implement such a utility.

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
        "foo": "foo",
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
            "foo": {
                "definition": { ... },
                "id": "foo",
                "filePath": "node_modules/eslint-plugin-foo/index.js",
                "importerPath": ".eslintrc.json"
            },
            "abc": {
                "definition": { ... },
                "id": "xyz",
                "filePath": "node_modules/eslint-plugin-xyz/index.js",
                "importerPath": ".eslintrc.json"
            },
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

## Documentation

This enhancement should be documented in "Configuring ESLint" page.

## Drawbacks

I think no problem.

## Backwards Compatibility Analysis

If people depend on the behavior that ESLint throws an error if they give an object to `plugins` property, this enhancement breaks that. However, I believe that we don't need to worry.

## Alternatives

- [#9] is the alternative. But double duplicate features cause confusion for the ecosystem. For newcomers, a mix of articles about two config systems makes hard to understand ESLint. For non-English users, the official document is far.
- [#14] is a different idea to handle local plugins. I think people have gotten used to key-value form via JavaScript object literals.

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [#14]
- [#9]

[#14]: https://github.com/eslint/rfcs/pull/14
[#9]: https://github.com/eslint/rfcs/pull/9
