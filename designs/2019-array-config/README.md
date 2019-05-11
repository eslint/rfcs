- Start Date: 2019-05-12
- RFC PR:
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Array Config

## Summary

This proposal makes `.eslintrc` files allowing an array as the top-level config. This is a syntax sugar of `extends` property and `overrides` property to make configuration simpler. Also, this form is closer to ESLint internal structure about configuration (`ConfigArray`).

## Motivation

The current config file mixes all of parent configs, child configs, and config content. A property (`extends`) is the parent config. Another property (`overrides`) is child configs. And the other properties is the config content. This means that we can write the parent configs after the child configs as opposing to intuitive.

The array notation makes this relationship simpler: "a later element in the array has precedence over an earlier element in the array."

Also, if people used multiple kinds of files, it's more intuitive than `overrides` property.

## Detailed Design

Currently, config files don't allow an array at top-level.

This RFC adds the support of the array. Each array element can be:

1. a string that `extends` property supports.
1. an object that is the same as the current config files.
1. an object that is the same as the elements of `overrides` property.
1. an array that is the same as top-level's. (nested)

The config array factory loads strings as `extends` and normalizes the others recursively.

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
<pre lang="js">
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
<code>extends</code> property will be still useful to apply shareable configs to limited files. (<code>extends</code> in <code>overrides</code> was implemented in <a href="https://github.com/eslint/eslint/pull/11554">eslint/eslint#11554</a>.)
</td></table>

## Documentation

This should be described in the "Configuring ESLint" page.

## Drawbacks

This is mostly syntax sugar, so the worth of this enhancement is smaller.

## Backwards Compatibility Analysis

If people depend on the behavior that ESLint throws an error if they give an array as configuration, this enhancement breaks that. However, I believe that we don't need to worry.

## Alternatives

-

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [#9]: https://github.com/eslint/rfcs/pull/9
- [#13]: https://github.com/eslint/rfcs/pull/13
