# Config File Improvements: Array Config

## Summary

This proposal makes `.eslintrc` files allowing an array as top-level config. This is a syntax sugar of `extends` property and `overrides` property to make configuration simpler. Also, this form is closer to ESLint internal structure about configuration.

## Motivation

The current config file has both parent config (`extends`) and child configs (`overrides`) as the same way. The configuration body overwrites the parent config and is overwritten by the child configs.

The array notation makes this relationship simpler: "a later element in the array has precedence over an earlier element in the array."

Also, this is useful for [Save people from the "plugin conflict" error](major-02-plugin-resolution-change.md#save-people-from-the-plugin-conflict-error).

## Detailed Design

> Proof of Concept (implementation): https://github.com/eslint/eslint/tree/proof-of-concept/config-array-in-eslintrc

This proposal enhances [there](README.md#array-config).

Currently, config files don't allow an array at top-level.

This RFC adds the support of an array. Each array element can be:

1. a string that `extends` property is supporting.
1. an object that is same as the current config files.
1. an object that is same as the elements of `overrides` property.
1. an array that is same as top-level's. (nested)

`ConfigArrayFactory` loads strings as `extends` and normalizes others recursively.

> [lib/lookup/config-array-factory.js#L626-L645](https://github.com/eslint/eslint/blob/proof-of-concept/config-array-in-eslintrc/lib/_lookup/config-array-factory.js#L626-L645)

<table><td>
💡 <b>Example</b>:
<pre lang="js">
module.exports = {
    extends: [
        "eslint:recommended",
        "plugin:node/recommended"
    ],
    rules: {
        eqeqeq: "error"
    },
    overrides: [
        {
            files: "*.ts",
            extends: "plugin:@typescript-eslint/recommended"
        }
    ]
}
</pre>
is rewritable to:
<pre lang="yml">
module.exports = [
    "eslint:recommended",
    "plugin:node/recommended",
    {
        rules: {
            eqeqeq: "error"
        }
    },
    {
        files: "*.ts",
        extends: "plugin:@typescript-eslint/recommended"
    }
]
</pre>
</td></table>

## Documentation

This should be described in the "Configuring ESLint" page.

## Drawbacks

This doesn't increase the capability of ESLint.

## Backwards Compatibility Analysis

If people depend on the behavior that ESLint throws an error if they give an array as configuration, this enhancement breaks that. However, I believe that we don't need to worry.

## Alternatives

-

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [#9]

[#9]: https://github.com/eslint/rfcs/pull/9
