-   Start Date: 2019-07-16
-   RFC PR: (leave this empty, to be filled in later)
-   Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# RuleTester supports processor

## Summary

This RFC makes `RuleTester` class supporting `processor` option.

## Motivation

Currently, we cannot test rules with processors. This is inconvenient for plugin rules which depend on processors. For example, [vue/comment-directive](https://github.com/vuejs/eslint-plugin-vue/blob/6751ff47b4ecd722bc2e2436ce6b34a510f92b07/tests/lib/rules/comment-directive.js) rule could not use `RuleTester` to test.

## Detailed Design

To add `processor` option and `processorOptions` (see #29) to test cases.

```js
const { RuleTester } = require("eslint")
const rule = require("../../../lib/rules/example-rule")
const exampleProcessor = require("../../../lib/processors/example-processor")

const tester = new RuleTester()

tester.run("example-rule", rule, {
    valid: [
        {
            code: `
                <script>
                    console.log("Hello")
                </script>
            `,
            processor: exampleProcessor,
            processorOptions: {},
        },
    ],
    invalid: [],
})
```

### § `processor` option

This is a definition object of a processor that the "[Processors in Plugins](https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins)" section describes.

If this option was given, the tester applies the given processor to test code.

If the given processor didn't has `supportsAutofix:true`, the tester doesn't do autofix. Then if the test case had `output` option (except `null`) then the test case will fail.

### § `processorOptions` option

RFC #29 defines the `processorOptions`.

If this option was given along with `processor` option, it will be given to the processor.

## Documentation

The [RuleTester](https://eslint.org/docs/developer-guide/nodejs-api#ruletester) section should describe the new `processor` and `processorOptions` properties.

## Drawbacks

It's a pretty rare case of a rule depends on the processor's behavior. I'm not sure if this feature is needed.

## Backwards Compatibility Analysis

There are no concerns about breaking changes.

## Related Discussions

-   https://github.com/eslint/rfcs/pull/25#issuecomment-499877621
