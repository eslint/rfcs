# Config File Improvements: `extends` in `overrides`

## Summary

This proposal makes the configuration in `overrides` property supporting `extends` property and `overrides` property (fixes [eslint/eslint#8813]).

## Motivation

Currently, cascading configuration supports `extends`, but globa-based configuration doesn't. This lacking has been preventing people to migrate their configuration to glob-based.

To support `extends` property (and nested `overrides` property for completeness) in `overrides` solve the pain.

## Detailed Design

> Proof of Concept (implementation): https://github.com/eslint/eslint/tree/proof-of-concept/config-array-in-eslintrc

This proposal enhances [there](README.md#extends-in-overrides).

If a configuration in `overrides` property has `extends` property or `overrides` property, it flattens those recursively rather than throws errors. The `files` property and `excludedFiles` property of the configuration is applied to every flattened item. If a flattened item has own `files` property and `excludedFiles` property, it composes those by logical AND.

> [lib/lookup/config-array-factory.js#L601-L624](https://github.com/eslint/eslint/blob/153640180a8944af3a1c488462ed30d0c215f5ed/lib/_lookup/config-array-factory.js#L601-L624) in PoC.

<table><td>
💡 <b>Example</b>:
<pre lang="jsonc">
{
    "extends": ["eslint:recommended"],
    "rules": { ... },
    "overrides": [
        {
            "files": ["*.ts"],
            "extends": ["plugin:@typescript-eslint/recommended"],
            "rules": { ... },
        }
    ]
}
</pre>
is flattend to:
<pre lang="jsonc">
[
    // extends
    {
        "name": ".eslintrc.json » eslint:recommended",
        "filePath": "node_modules/eslint/conf/eslint-recommended.js",
        "rules": { ... }
    },
    // main
    {
        "name": ".eslintrc.json",
        "filePath": ".eslintrc.json",
        "rules": { ... }
    },
    // overrides (because it flattens recursively, extends in overrides is here)
    {
        "name": ".eslintrc.json#overrides[0] » plugin:@typescript-eslint/recommended",
        "filePath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js",
        // `matchFile` is merged from the parent `overrides` entry and itself.
        "matchFile": { "includes": ["*.ts"], "excludes": null },
        "parser": { ... },
        "parserOptions": { ... },
        "plugins": { ... },
        "rules": { ... }
    },
    {
        "name": ".eslintrc.json#overrides[0]",
        "filePath": ".eslintrc.json",
        "matchFile": { "includes": ["*.ts"], "excludes": null },
        "rules": { ... }
    }
]
</pre>
</td></table>

## Documentation

This enhancement should be documented in "Configuring ESLint" page.

## Drawbacks

- I think no problem.

## Backwards Compatibility Analysis

If people depend on the behavior that ESLint throws an error if they give `extends` property or `overrides` property in a config of `overrides` property, this enhancement breaks that. However, I believe that we don't need to worry.

## Alternatives

- We can provide a utility that merges multiple configs for this purpose. However, people have gotten used to `extends` property, and it's easy to use.
- [Array Config](minor-01-array-config.md) may be an alternative, but it cannot apply `files` and `excludedFiles` to the extended config by logical AND.

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [#9]
- [eslint/eslint#8813]

[#9]: https://github.com/eslint/rfcs/pull/9
[eslint/eslint#8813]: https://github.com/eslint/eslint/issues/8813
