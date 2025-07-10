- Repo: https://github.com/eslint/eslint
- Start Date: 2025-07-10
- RFC PR: (leave this empty, to be filled in later)
- Authors: ST-DDT

# Rule Tester Assertions Options

## Summary

Add options that control which assertions are required for each `RuleTester` test case.

## Motivation

In most eslint(-plugin)s' rules the error assertions are different from each other.
Adding options that could be set/shared when configuring the rule tester would ensure a common base level of assertions is met throughout the (plugin's) project.
The options could be defined on two levels. On `RuleTester`'s `constructor` effectively impacting all tests, or on the test `run` method itself, so only that set is affected.

## Detailed Design

### Variant 1 - Constructor based options

````ts
new RuleTester(testerConfig: {...}, assertionOptions: {
    /**
     * Require message assertion for each invalid test case.
     * 
     * @default false
     */
    requireMessage: boolean;
    /**
     * Require full location assertions for each invalid test case.
     * 
     * @default false
     */
    requireLocation: boolean;
} = {});
````

### Variant 2 - Test method based options

````ts
ruleTester.run("rule-name", rule, tests, assertionOptions: {
    /**
     * Require message assertion for each invalid test case.
     * 
     * @default false
     */
    requireMessage: boolean;
    /**
     * Require full location assertions for each invalid test case.
     * 
     * @default false
     */
    requireLocation: boolean;
    /**
     * Require and expect only the given test scenarios.
     * This allows omitting certain scenarios from this run with the current options.
     * 
     * @default ["valid","invalid"]
     */
    requiredScenarios: ReadonlyArray<'valid' | 'invalid'>;
});
````

### Shared Logic

If `requireMessage` is set to `true`, the invalid test case cannot consist of an error count assertion only, but must also include a message assertion.
This can be done either by providing a only message, or by using the `message` property of the error object in the assertion (Same as the current behavior).
We could enable this property by default, but it would be a breaking change.
If we add a `requireMessageId` option, it would be mutually exclusive with `requireMessage`, and the invalid test case cannot consist of an error count or message assertion only, but must also include a messageId assertion.
Alternatively, we could alter the `requireMessage` option to `false | true | "message" | "messageId"` (`true` => `"message"`).

````ts
ruleTester.run("rule-name", rule, {
    invalid: [
        {
            code: "const a = 1;",
            errors: 1, // ❌
        },
        {
            code: "const a = 2;",
            errors: [
                "Error message here.", // ✅
            ]
        },
        {
            code: "const a = 3;",
            errors: [
                {
                    message: "Error message here.", // ✅
                }
            ]
        }
    ]
}, {
    requireMessage: true
});
````

If `requireLocation` is set to `true`, the invalid test case cannot consist of an error count or errorMessage assertion only, but must also include a full location assertion.
We could enable this property by default, but it would be a breaking change.

````ts
ruleTester.run("rule-name", rule, {
    invalid: [
        {
            code: "const a = 1;",
            errors: 1, // ❌
        },
        {
            code: "const a = 2;",
            errors: [
                "Error message here.", // ❌
            ]
        },
        {
            code: "const a = 3;",
            errors: [
                {
                    line: 1, // ❌
                    column: 1, 
                    
                }
            ]
        },
        {
            code: "const a = 4;",
            errors: [
                {
                    line: 1, // ✅
                    column: 1,
                    endLine: 1,
                    endColumn: 12,
                }
            ]
        }
    ]
}, {
    requireLocation: true
});
````

If `requiredScenarios` is set, the `run` will only require and expect the given scenarios.
This can only be used for the `run` method, not the constructor, because there should always be at least one valid and one invalid test case.
The `requiredScenarios` option can be used to omit certain scenarios from the run, e.g. if the user wants to import a set of tests from a different source, that may have other assertion requirements or haven't achieved the quality needed to require the same assertion strictness.

````ts
ruleTester.run("rule-name", rule, {
    valid: [...], // ✅
    invalid: [...],
}, {
    // Default is ["valid", "invalid"]
});
ruleTester.run("rule-name", rule, {
    valid: [...], // ❌
}, {
    // Default is ["valid", "invalid"]
});
ruleTester.run("rule-name", rule, {
    valid: [...], // ✅
    invalid: [...],
}, {
    requiredScenarios: ["valid", "invalid"]
});
ruleTester.run("rule-name", rule, {
    valid: [...], // ✅
}, {
    requiredScenarios: ["valid"]
});
ruleTester.run("rule-name", rule, {
    valid: [...], // ❌
    invalid: [...],
}, {
    requiredScenarios: ["invalid"]
});
````

## Documentation

This RFC will be documented in the RuleTester documentation, explaining the new options and how to use them.
So mainly here: https://eslint.org/docs/latest/integrate/nodejs-api#ruletester
Additionally, we should write a short blog post to announce for plugin maintainers, to raise awareness of the new options and encourage them to use them in their tests.

## Drawbacks

This proposal adds slightly more complexity to the RuleTester logic, as it needs to handle the new options and enforce the assertions based on them.
Currently, the RuleTester logic is already deeply nested, so adding more options may make it harder to read and maintain.

Additionally, since we add the options as a second parameter it might interfere with future additions to the parameters.
This could by eleviated by renaming the parameter from `assertionOptions` to `options` (either from the start or when the need for different type of options arises).

If we enable the `requireMessage` and `requireLocation` options by default, it would be a breaking change for existing tests that do not follow these assertion requirements yet.

## Backwards Compatibility Analysis

This change should not affect existing ESLint users or plugin developers, as it only adds new options to the RuleTester and does not change any existing behavior.
If we enable the `requireMessage` and `requireLocation` options by default, it would be a breaking change for existing tests that do not follow these assertion requirements yet.
If we want to enable them by default, we should do so in a major release and communicate the upcoming change to the users early via blog post.

## Alternatives

As an alternative to this proposal, we could add a eslint rule that applies the same assertions, but uses the central eslint config.
While this would apply the same assertions for all rule testers, it would be a lot more complex to implement and maintain, 
it requires identifying the RuleTester calls in the codebase and might run into issues if the assertions aren't specified inline but via a variable or transformation.

## Open Questions

1. Is there a need for disabling scenarios like `valid` or `invalid`?
2. Should we use constructor-based options or test method-based options? Do we support both? Or global options so it applies to all test files?
3. Should we enable the `requireMessage` and `requireLocation` options by default? (Breaking change)
4. Do we add a `requireMessageId` option or should we alter the `requireMessage` option to support both message and messageId assertions?
5. Should we add a `strict` option that enables all assertion options by default?

## Help Needed

English is not my first language, so I would appreciate help with the wording and grammar of this RFC.
I'm able to implement this RFC, if we decide to go with options instead of a new rule.

## Frequently Asked Questions

### Why

Because it is easy to miss a missing assertion in RuleTester test cases, especially when many new invalid test cases are added.

## Related Discussions

The idea was initially sparked by this comment: vuejs/eslint-plugin-vue#2773 (comment)

- https://github.com/vuejs/eslint-plugin-vue/pull/2773#discussion_r2176359714

> It might be helpful to include more detailed error information, such as line, column, endLine, and endColumn...
> [...]
> Let’s make the new cases more detailed first. 😊

The first steps have been taken in: eslint/eslint#19904 - feat: output full actual location in rule tester if different

- https://github.com/eslint/eslint/pull/19904

This lead to the issue that this RFC is based on: eslint/eslint#19921 - Change Request: Add options to rule-tester requiring certain assertions

- https://github.com/eslint/eslint/issues/19921
