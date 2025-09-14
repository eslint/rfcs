- Repo: https://github.com/eslint/eslint
- Start Date: 2025-07-10
- RFC PR: https://github.com/eslint/rfcs/pull/137
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

```ts
new RuleTester(testerConfig: {...}, assertionOptions: {
  /**
   * Require message assertion for each invalid test case.
   * If `true`, errors must extend `string | RegExp | Array<{ message: string | RegExp } | { messageId: string }>`.
   * If `'message'`, errors must extend `string | RegExp | Array<{ message: string | RegExp }>`.
   * If `'messageId'`, errors must extend `Array<{ messageId: string }>`.
   *
   * @default false
   */
  requireMessage: boolean | 'message' | 'messageId';
  /**
   * Require full location assertions for each invalid test case.
   * If `true`, errors must extend `Array<{ line: number, column: number, endLine?: number | undefined, endColumn?: number | undefined }>`.
   * `endLine` or `endColumn` may be absent, if the observed error does not contain these properties.
   *
   * @default false
   */
  requireLocation: boolean;
} = {});
```

### Variant 2 - Test method based options

```ts
ruleTester.run("rule-name", rule, {
  assertionOptions?: {
    /**
     * Require message assertion for each invalid test case.
     * If `true`, errors must extend `string | RegExp | Array<{ message: string | RegExp } | { messageId: string }>`.
     * If `'message'`, errors must extend `string | RegExp | Array<{ message: string | RegExp }>`.
     * If `'messageId'`, errors must extend `Array<{ messageId: string }>`.
     *
     * @default false
     */
    requireMessage?: boolean | 'message' | 'messageId';
    /**
     * Require full location assertions for each invalid test case.
     * If `true`, errors must extend `Array<{ line: number, column: number, endLine?: number | undefined, endColumn?: number | undefined }>`.
     * `endLine` or `endColumn` may be absent or undefined, if the observed error does not contain these properties.
     *
     * @default false
     */
    requireLocation?: boolean;
  },
  valid: [ /* ... */ ],
  invalid: [ /* ... */ ],
});
```

### Shared Logic

#### requireMessage

If `requireMessage` is set to `true`, the invalid test case cannot consist of an error count assertion only, but must also include a message assertion. (See below)
This can be done either by providing only a `string`/`RegExp` message, or by using the `message`/`messageId` property of the error object in the `TestCaseError` (Same as the current behavior).
If `true`, errors must extend `string | RegExp | Array<{ message: string | RegExp } | { messageId: string }>`.
If `'message'`, errors must extend `string | RegExp | Array<{ message: string | RegExp }>`.
If `'messageId'`, errors must extend `Array<{ messageId: string }>`.

```ts
ruleTester.run("rule-name", rule, {
  assertionOptions: {
    requireMessage: true,
  },
  invalid: [
    {
      code: "const a = 1;",
      errors: 1, // ❌ Message isn't being checked here
    },
    {
      code: "const a = 2;",
      errors: [
        "Error message here.", // ✅ we only check the error message
      ],
    },
    {
      code: "const a = 3;",
      errors: [
        {
          message: "Error message here.", // ✅ we check the error message and potentially other properties
        },
      ],
    },
  ],
});
```

#### requireLocation

If `requireLocation` is set to `true`, the invalid test case's error validation must include all location assertions such as `line`, `column`, `endLine`, and `endColumn`.
`endLine` or `endColumn` may be absent or `undefined`, if the observed error does not contain these properties.

```ts
ruleTester.run("rule-name", rule, {
  assertionOptions: {
    requireLocation: true,
  },
  invalid: [
    {
      code: "const a = 1;",
      errors: 1, // ❌ Location isn't checked here
    },
    {
      code: "const a = 2;",
      errors: [
        "Error message here.", // ❌ Location isn't checked here
      ],
    },
    {
      code: "const a = 3;",
      errors: [
        {
          line: 1, // ❌ Location isn't fully checked here
          column: 1,
        },
      ],
    },
    {
      code: "const a = 4;",
      errors: [
        {
          line: 1, // ✅ All location properties have been checked
          column: 1,
          endLine: 1,
          endColumn: 12,
        },
      ],
    },
  ],
});
```

## Implementation Hint

```patch
diff --git a/lib/rule-tester/rule-tester.js b/lib/rule-tester/rule-tester.js
index dbd8c274f..b39631faa 100644
--- a/lib/rule-tester/rule-tester.js
+++ b/lib/rule-tester/rule-tester.js
@@ -555,6 +555,10 @@ class RuleTester {
 	 * @param {string} ruleName The name of the rule to run.
 	 * @param {RuleDefinition} rule The rule to test.
 	 * @param {{
+	 *   assertionOptions?: {
+	 *     requireMessage?: boolean | "message" | "messageId",
+	 *     requireLocation?: boolean
+	 *   },
 	 *   valid: (ValidTestCase | string)[],
 	 *   invalid: InvalidTestCase[]
 	 * }} test The collection of tests to run.
@@ -563,6 +567,14 @@ class RuleTester {
 	 * @returns {void}
 	 */
 	run(ruleName, rule, test) {
+		if (!test || typeof test !== "object") {
+			throw new TypeError(
+				`Test Scenarios for rule ${ruleName} : Could not find test scenario object`,
+			);
+		}
+
+		const { requireMessage = false, requireLocation = false } =
+			test.assertionOptions ?? {};
 		const testerConfig = this.testerConfig,
 			requiredScenarios = ["valid", "invalid"],
 			scenarioErrors = [],
@@ -582,12 +594,6 @@ class RuleTester {
 			);
 		}
 
-		if (!test || typeof test !== "object") {
-			throw new TypeError(
-				`Test Scenarios for rule ${ruleName} : Could not find test scenario object`,
-			);
-		}
-
 		requiredScenarios.forEach(scenarioType => {
 			if (!test[scenarioType]) {
 				scenarioErrors.push(
@@ -1092,6 +1098,10 @@ class RuleTester {
 			}
 
 			if (typeof item.errors === "number") {
+				assert.ok(
+					!requireMessage && !requireLocation,
+					"Invalid cases must have 'errors' value as an array",
+				);
 				if (item.errors === 0) {
 					assert.fail(
 						"Invalid cases must have 'error' value greater than 0",
@@ -1137,6 +1147,10 @@ class RuleTester {
 
 					if (typeof error === "string" || error instanceof RegExp) {
 						// Just an error message.
+						assert.ok(
+							requireMessage !== "messageId" && !requireLocation,
+							"Invalid cases must have 'errors' value as an array of objects e.g. { message: 'error message' }",
+						);
 						assertMessageMatches(message.message, error);
 						assert.ok(
 							message.suggestions === void 0,
@@ -1157,6 +1171,10 @@ class RuleTester {
 						});
 
 						if (hasOwnProperty(error, "message")) {
+							assert.ok(
+								requireMessage !== "messageId",
+								"The test run requires 'messageId' but the error (also) uses 'message'.",
+							);
 							assert.ok(
 								!hasOwnProperty(error, "messageId"),
 								"Error should not specify both 'message' and a 'messageId'.",
@@ -1170,6 +1188,10 @@ class RuleTester {
 								error.message,
 							);
 						} else if (hasOwnProperty(error, "messageId")) {
+							assert.ok(
+								requireMessage !== "message",
+								"The test run requires 'message' but the error (also) uses 'messageId'.",
+							);
 							assert.ok(
 								ruleHasMetaMessages,
 								"Error can not use 'messageId' if rule under test doesn't define 'meta.messages'.",
@@ -1242,21 +1264,43 @@ class RuleTester {
 						if (hasOwnProperty(error, "line")) {
 							actualLocation.line = message.line;
 							expectedLocation.line = error.line;
+						} else if (requireLocation) {
+							assert.fail(
+								"Test error must specify a 'line' property.",
+							);
 						}
 
 						if (hasOwnProperty(error, "column")) {
 							actualLocation.column = message.column;
 							expectedLocation.column = error.column;
+						} else if (requireLocation) {
+							assert.fail(
+								"Test error must specify a 'column' property.",
+							);
 						}
 
 						if (hasOwnProperty(error, "endLine")) {
 							actualLocation.endLine = message.endLine;
 							expectedLocation.endLine = error.endLine;
+						} else if (
+							requireLocation &&
+							hasOwnProperty(message, "endLine")
+						) {
+							assert.fail(
+								"Test error must specify an 'endLine' property.",
+							);
 						}
 
 						if (hasOwnProperty(error, "endColumn")) {
 							actualLocation.endColumn = message.endColumn;
 							expectedLocation.endColumn = error.endColumn;
+						} else if (
+							requireLocation &&
+							hasOwnProperty(message, "endColumn")
+						) {
+							assert.fail(
+								"Test error must specify an 'endColumn' property.",
+							);
 						}
 
 						if (Object.keys(expectedLocation).length > 0) {
```

## Documentation

This RFC will be documented in the RuleTester documentation, explaining the new options and how to use them.
So mainly here: https://eslint.org/docs/latest/integrate/nodejs-api#ruletester
Additionally, we should write a short blog post to announce for plugin maintainers, to raise awareness of the new options and encourage them to use them in their tests.

## Drawbacks

This proposal adds slightly more complexity to the RuleTester logic, as it needs to handle the new options and enforce the assertions based on them.
Currently, the RuleTester logic is already deeply nested, so adding more options may make it harder to read and maintain.

Due to limitations of the `RuleTester`, the error message does not point to the correct location of the test case.
While this is true already, it gets slightly more relevant, since now it might report missing properties, that cannot be searched for in the test file.

See also: https://github.com/eslint/eslint/issues/19936

Additionally, since we add the options as a second parameter it might interfere with future additions to the parameters.
This could by eleviated by renaming the parameter from `assertionOptions` to `options` (either from the start or when the need for different type of options arises).

If we enable the `requireMessage` and `requireLocation` options by default, it would be a breaking change for existing tests that do not follow these assertion requirements yet.
It is not planned to enable `requireMessage` or `requireLocation` by default.

## Backwards Compatibility Analysis

This change should not affect existing ESLint users or plugin developers, as it only adds new options to the RuleTester and does not change any existing behavior.

## Alternatives

As an alternative to this proposal, we could add a eslint rule that applies the same assertions, but uses the central eslint config.
While this would apply the same assertions for all rule testers, it would be a lot more complex to implement and maintain,
it requires identifying the RuleTester calls in the codebase and might run into issues if the assertions aren't specified inline but via a variable or transformation.

## Open Questions

1. ~~Is there a need for disabling scenarios like `valid` or `invalid`?~~ No, unused scenarios can be omitted using empty arrays. If needed, this option can be added later on.
2. Should we use constructor-based options or test method-based options? Do we support both? Or global options so it applies to all test files? Currently, the method level options (variant 2) is the prefered one.
3. ~~Should we enable the `requireMessage` and `requireLocation` options by default? (Breaking change)~~ No
4. ~~Do we add a `requireMessageId` option or should we alter the `requireMessage` option to support both message and messageId assertions?~~ Just `requireMessage: boolean | 'message' | 'messageid'`
5. ~~Should we add a `strict` option that enables all assertion options by default?~~ No, currently not planned.
6. ~~Should we inline the options?~~ No, this might cause issues with alphabetical object properties.
7. ~~What should the otpions be called?~~ We name it `assertionOptions` to avoid issues with alphabetical object properties. e.g. `invalid, options, valid`

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
