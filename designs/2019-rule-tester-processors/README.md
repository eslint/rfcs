-   Start Date: 2019-07-16
-   RFC PR: (leave this empty, to be filled in later)
-   Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea)), Brandon Mills ([@btmills](https://github.com/btmills))

# RuleTester supports processor

## Summary

This RFC makes `RuleTester` class supporting `processor` option.

## Motivation

Currently, we cannot test rules with processors. This is inconvenient for plugin rules which depend on processors. For example, [vue/comment-directive](https://github.com/vuejs/eslint-plugin-vue/blob/6751ff47b4ecd722bc2e2436ce6b34a510f92b07/tests/lib/rules/comment-directive.js) rule could not use `RuleTester` to test. Rules that distinguish between physical and virtual filenames [cannot be tested without processors](https://github.com/eslint/eslint/issues/14800).

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

If a processor is given, the tester passes its `preprocess`, `postprocess`, and optional `supportsAutofix` properties as part of `Linter#verify()`'s `filenameOrOptions` options object.

The tester only applies fixes if the processor also has `supportsAutofix: true`.

### § `processorOptions` option

RFC #29 defines the `processorOptions`, though it has not yet been implemented.

If this option was given along with `processor` option, it will be given to the processor.

## Documentation

The [RuleTester](https://eslint.org/docs/developer-guide/nodejs-api#ruletester) section should describe the new `processor` and, if implemented, `processorOptions` properties.

## Drawbacks

This expands `RuleTester`'s purpose to include testing processor-aware rules.
Some could see this as out of scope for `RuleTester`.

It adds additional complexity to `RuleTester`, though it is minimal.

This design only supports a single processor.

## Backwards Compatibility Analysis

`RuleTester` currently accepts the `processor` key as a string in valid and invalid test case objects, but it does nothing with that string.
It throws an error if `processor` is set to any non-string type.
If someone has set `processor` to a string value in any of their test cases, those tests would throw with this change.
Because the `processor` key does nothing prior to this change, having it is unlikely, and the fix is easy: deleting the `processor` key leaves the test logically unchanged.

## Alternatives

- To support testing rules that distinguish between physical and virtual filenames, we could instead add a `physicalFilename` option to test cases and modify `Linter` to use that option instead of computing filenames.

## Open Questions

- Does this design need to accept multiple nested processors?

## Help Needed

I can implement this myself.

## Frequently Asked Questions

None yet.
## Related Discussions

- https://github.com/eslint/rfcs/pull/25#issuecomment-499877621
- https://github.com/eslint/eslint/issues/14800
