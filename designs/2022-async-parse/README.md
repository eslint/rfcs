- Repo: eslint/eslint
- Start Date: 2022-03-17
- RFC PR: (leave this empty, to be filled in later)
- Authors: Milos Djermanovic

# Support `async parse()` / `async parseForESLint()`

## Summary

ESLint now requires that `parse` methods of custom parsers return `AST` synchronously. This RFC proposes to allow the `parse` methods of custom parsers to return `Promise<AST>`. Similarly, the `parseForESLint` methods of custom parsers would be allowed to return `Promise<ParseResult>` instead of `ParseResult`.

## Motivation

Per the [original issue](https://github.com/eslint/eslint/issues/15475), this feature will allow for the use of ESLint with custom parsers written in languages that compile to Wasm or other binary formats.

Compiled parsers are expected to work faster than JS-interpreted parsers and therefore bring performance improvements to the linting process.

Other use cases may include asynchronous loading of external parser configs and parser plugins, which could be performed by the parser inside `async parse()` / `async parseForESLint()`. 

**Note**: This RFC does not include parallel parsing and linting.

## Detailed Design

* Working solution: https://github.com/mdjermanovic/eslint/tree/async-parse
* Diff: https://github.com/mdjermanovic/eslint/pull/3/files

### `Linter` class

#### Summary

Linting will become exclusively async - public methods `verify` and `verifyAndFix` will always return promises.

#### Details

The changes begin with the first place where the parse results are handled, in the internal [`function parse`](https://github.com/eslint/eslint/blob/76a235a31718312c2ed202fdde030d329ca62486/lib/linter/linter.js#L778), which becomes `async`:

```diff
- function parse(text, languageOptions, filePath) {
+ async function parse(text, languageOptions, filePath) {
```

The parse results will always be `await`-ed:

```diff
const parseResult = (typeof parser.parseForESLint === "function")
-    ? parser.parseForESLint(textToParse, parserOptions)
-    : { ast: parser.parse(textToParse, parserOptions) };
+    ? await parser.parseForESLint(textToParse, parserOptions)
+    : { ast: await parser.parse(textToParse, parserOptions) };
```

This changes the return type of `function parse` to a Promise. Consequently, all functions that are directly or indirectly using this function will need to be updated:

* `_verifyWithoutProcessors` (private method) becomes `async`.
* `_verifyWithProcessor` (private method) becomes `async`.
* `_verifyWithConfigArray` (private method) becomes `async`.
* `verifyAndFix` (public method) becomes `async`.
* `verify` (public method) becomes `async`.

**Note**: The above list assumes that these changes will be implemented in a version of `Linter` that supports only [flat config](https://github.com/eslint/eslint/issues/13481). In particular, it assumes that there will be no separate methods such as `_verifyWithFlatConfigArray`.

Most changes are straightforward. Functions that operate on results from other async functions will become `async` functions themselves, and they'll `await` those results. Functions that just `return` results from other async functions don't need to `await` those results, but we'll nevertheless convert them to `async` functions to ensure consistent signature.

`_verifyWithProcessor` will lint code blocks one by one, as before this change. The linting of the second code block will start after the linting of the first code block has finished, and so on.

[`configArray.normalizeSync()`](https://github.com/eslint/eslint/blob/18f5e05bce10503186989d81ca484abb185a2c9d/lib/linter/linter.js#L1464) call can be replaced with `configArray.normalize()`, to support async configs.

### `ESLint` class

#### Summary

There are no changes in the public API.

#### Details

TBD - waiting for `FlatESLint` class. The working solution provided with this RFC includes changes to the current `ESLint` and `CLIEngine` classes, but I assume that this RFC will actually be implemented in the new flat config version of the `ESLint` class.

`ESLint` methods are already `async`, so the changes are expected to be small.

* A new instance of `Linter` will be created for each file being linted. `Linter` instances have mutable internal state that includes data related to a particular run of `verify()`/`verifyAndFix()`, in particular `lastSourceCode`, `lastConfigArray ` and `lastSuppressedMessages`. Therefore, the same instance of `Linter` should not be used concurrently. With the current sync API of the `Linter` class, this wasn't a problem because linting always runs to completion. With the new async API, it's safer to always create a new instance, and I don't see any particular benefits of reusing the same instance of `Linter` for all files.
* `linter.verifyAndFix()` will be `await`-ed.
* `ESLint#lintFiles` will still lint files one by one. This allows for all objects created for the purpose of linting one file, such as its `AST` and rule listeners, to be garbage-collected before proceeding to the next file. Otherwise, `lintFiles` might be linting all files at once (step by step, for example: parse file1.js -> parse file2.js -> ... -> parse fileN.js -> run rules on file1.js -> run rules on file2.js -> ... -> run rules on fileN.js) and thus holding references to all those objects till the end. If there could be some benefits of starting `linter.verifyAndFix()` on the next file before the previous one has finished, or even starting all at the same time, that is not in the scope of this RFC (see Open Questions).

### `RuleTester` class

#### Summary

Test functions passed to `it` will always return promises. `itDefaultHandler` will be removed.

#### Details

`RuleTester` works by running `Linter#verify` and asserting the results. Since these results must now be `await`-ed, the following functions should be changed:

* `runRuleForItem` (internal function) becomes `async`.
* `testValidTemplate` (internal function) becomes `async`.
* `testInvalidTemplate` (internal function) becomes `async`.

Consequently, all tests become asynchronous. We'll update `it` callbacks to return promises returned from `testValidTemplate` / `testInvalidTemplate`.

```diff
this.constructor[invalid.only ? "itOnly" : "it"](
    sanitize(invalid.name || invalid.code),
-    () => {
-        testInvalidTemplate(invalid);
-    }
+    () => testInvalidTemplate(invalid)
);
```

Mocha and Jest support returning a Promise. The test will pass when the returned Promise is fulfilled, or fail if the Promise is rejected or timeout expires. Existing usages of `RuleTester#run()` with Mocha or Jest should "just work" without any changes.

[Wrapper parsers](https://github.com/eslint/eslint/blob/18f5e05bce10503186989d81ca484abb185a2c9d/lib/rule-tester/rule-tester.js#L272) will always be async parsers, and they'll `await` the wrapped parser's `parse()` / `parseForESLint()`:

```diff
    if (typeof parser.parseForESLint === "function") {
        return {
            [parserSymbol]: parser,
-            parseForESLint(...args) {
-                const ret = parser.parseForESLint(...args);
+            async parseForESLint(...args) {
+                const ret = await parser.parseForESLint(...args);

                defineStartEndAsErrorInTree(ret.ast, ret.visitorKeys);
                return ret;
            }
        };
    }
```

`itDefaultHandler` will be removed. If a custom `RuleTester.it` was not set and there's no global `it`, `RuleTester#run()` will throw an error.

```diff
static get it() {
-    return (
-        this[IT] ||
-        (typeof it === "function" ? it : itDefaultHandler)
-    );
+    if (this[IT]) {
+        return this[IT];
+    }
+
+    if (typeof it === "function") {
+        return it;
+    }
+
+    throw new Error("Cannot find 'it' function. Use a testing framework like Mocha or Jest, or set a custom 'RuleTester.it'.");
}
```

This means that `RuleTester` can no longer be used without a testing framework or a custom `it` handler. Currently, if there is no custom/global `it`, `RuleTester` runs the tests directly by calling `itDefaultHandler` which just calls the test function. If the tests are not passing, `RuleTester#run()` throws an assertion error from the first failing test. With async tests, we can no longer support the functionality of throwing an error when the tests are not passing, so it will be removed.

### Internal tools

#### Summary

`eslint-fuzzer` uses `Linter#verify` and `Linter#verifyAndFix`, so it must become async. Consenquently, `Makefile.js` needs to be updated to handle the now async fuzz check appropriately.

#### Details

TODO: as these are internal tools, I left out the details from the first version of this RFC.

## Documentation

* [Working with Custom Parsers](https://github.com/eslint/eslint/blob/main/docs/developer-guide/working-with-custom-parsers.md) should be updated with info that `parse` / `parseForESLint` can return a Promise, and an example for `async parseForESLint()`.
* [Node.js API - Linter](https://github.com/eslint/eslint/blob/main/docs/developer-guide/nodejs-api.md#linter) should be updated with the new return types for `verify` and `verifyAndFix`.
* [Node.js API - RuleTester](https://github.com/eslint/eslint/blob/main/docs/developer-guide/nodejs-api.md#ruletester) should be updated with new requirements for its use.
* We should announce the breaking changes in a "What's coming in ESLint v9.0.0" blog post.

## Drawbacks

* Added complexity. While the code stays pretty much the same thanks to the async-await syntax, asynchronous execution is inherently more complex to reason about and brings additional concerns such as concurrency issues.
* Performance with sync parsers. Converting a sync flow to an async flow with the exact same steps should add some time to the execution, though I don't see any differences when running `npm run perf` on the [async-parse](https://github.com/mdjermanovic/eslint/tree/async-parse) branch.
* This requires breaking changes in the `Linter` API and new requirements for the use of `RuleTester`.

## Backwards Compatibility Analysis

### End Users

There should be no breaking changes for end users.

### API

Breaking changes in the public API:

* `Linter#verify` will always return `Promise<LintMessage[]>` instead of `LintMessage[]`.
* `Linter#verifyAndFix()` will always return `Promise<{fixed:boolean,messages:LintMessage[],output:string}>` instead of `{fixed:boolean,messages:LintMessage[],output:string}`.

We provide the `Linter` class mostly for the use in browsers. Projects using this class will have to update their code for the new async API.

ESLint integrations are expected to use the `ESLint` class. There will be no changes for them.

### Plugins

Breaking changes for plugins are only related to `RuleTester`:

* `RuleTester` will require a testing framework or custom `it` handler.
* `it` must support returning a Promise from passed callbacks. It should await for the Promise to be settled and treat the test as "passing" only if the Promise is fulfilled. If the Promise is not settled within a certain timeframe or is rejected, the test should be treated as "failing". Mocha and Jest already support this. 

Projects that use `RuleTester` with Mocha or Jest wouldn't be required to change anything.

Projects that run tests directly or use a testing framework that doesn't support returning a Promise from callbacks will have to switch to a framework like Mocha or Jest.

## Alternatives

### Alternatives to `async parse()` / `async parseForESLint()`

If the use of compiled parsers with ESLint is the only use case for `async parse` / `async parseForESLint` we are aware of, then the question is - is `async parse` / `async parseForESLint` the only way to enable it? In particular, if the parser only needs a one-time async initialization step (I believe that would be the case with Wasm), then the async initialization can be performed in `eslint.config.js` on loading, since the new flat config system supports [async function configs](https://github.com/eslint/rfcs/tree/main/designs/2019-config-simplification#can-a-config-be-an-async-function).

### Alternatives to async-only `Linter`

Instead of removing support for synchronous linting, `verify()` and `verifyAndFix()` could return results synchronously when the parser is sync, or we could add separate methods `verifySync()` and `verifyAndFixSync()`.

This would add a lot of complexity as we would have to maintain both sync and async code paths throughout the `Linter`. It would become even more complicated when we add other async features, like async rules.

### Alternatives to requiring a testing framework

Instead of throwing an error if custom/global `it` was not found, `RuleTester#run` could run all test directly and return a Promise, but that could make the behavior of this method quite confusing for users.

We could add a separate method `RuleTester#runDirectly` that always runs the tests directly and always returns a Promise, but the question is whether anyone needs it.

## Open Questions

* Do we want to explore parallel parsing-linting as described in https://github.com/eslint/eslint/discussions/14733 in this RFC or separately? 
* Assuming that we plan to implement both `async parse()` support and Flat Config for v9.0.0, can we split `FlatLinter` out of the `Linter` now? The work on `async parse()` overlaps a lot with the work on flat config. With the current dual-mode `Linter`, it doesn't seem feasible to start implementing support for `async parse()` before the final v9.0.0 flat config changes are merged. If we split out `FlatLinter`, most of the work could be finished much earlier and we could have a working support for async parsers in the "flat" mode (the mode that uses `FlatESLint` class).

## Help Needed

I can implement this myself.

## Frequently Asked Questions

* Is this related to [RFC#42 Lint files in parallel](https://github.com/eslint/rfcs/pull/42)? - no.

## Related Discussions

* https://github.com/eslint/eslint/issues/15475
* https://github.com/eslint/eslint/discussions/14295
* https://github.com/eslint/eslint/discussions/14733 ?
