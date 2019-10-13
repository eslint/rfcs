- Start Date: 2019-09-28
- RFC PR: https://github.com/eslint/rfcs/pull/40
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Moving to Asynchronous API

## Summary

This RFC adds a new class `ESLint` that has asynchronous API and deprecates `CLIEngine`.

## Motivation

- We have the functionality that we cannot support with synchronous API. For example, ESLint verifies files in parallel, formatters print progress state, or formatters print results in streaming.
- Dynamic `import()` has arrived at Stage 4. The dynamic loading of ES modules requires asynchronous API. Migrating to asynchronous API opens up doors to write plugins/configs with ES modules.
- The name of `CLIEngine`, our primary API, has caused confusion to the community. People try `Linter` class at first because the name sounds the main class of ESLint, then they notice it doesn't work as expected. We have a lot of issues that say "please use `CLIEngine` instead." The name of the new class, `ESLint`, is our primary API clearly.

## Detailed Design

### ■ Add new `ESLint` class

This RFC adds a new class `ESLint`. It has almost the same methods as `CLIEngine`, but the return value of some methods are different.

So, for now, `ESLint` class will be a tiny wrapper of `CLIEngine` that modifies the type of the return values.

#### § The `executeOnFiles()` method

This method returns an object that has two properties two methods `then()` and `[Symbol.asyncIterator]()`, and we can use the returned object with `await` expression and `for-await-of` statement.

- If you used the returned object with `for-await-of` statement, it iterates the lint result of each file in random order. This way yields each lint result immediately. Therefore you can print the results in streaming, or print progress state. ESLint may spend time to lint files (for example, ESLint needs about 20 seconds to lint our codebase), to print progress state will be useful.

  ```js
  const { ESLint } = require("eslint")
  const eslint = new ESLint()

  for await (const result of eslint.executeOnFiles(patterns)) {
    print(result)
  }
  ```

- If you used the returned object with `await` expression, it returns the array that contains all lint results in random order. This way waits for all lint results then returns it.

  ```js
  const { ESLint } = require("eslint")
  const eslint = new ESLint()

  const results = await eslint.executeOnFiles(patterns)
  // Optionally you can sort the results.
  // results.sort(ESLint.compareResultsByFilePath)
  print(results)
  ```

Either way, we can support "linting in parallel", loading configs/plugins with `import()`, and other stuff that needs asynchronous logic.

##### Move the `usedDeprecatedRules` property

The returned object of `CLIEngine#executeOnFiles()` has the `usedDeprecatedRules` property that includes the deprecated rule IDs which the linting used. But the location doesn't fit the use with `for-await-of` statement. Therefore, this RFC mvoes the `usedDeprecatedRules` property to each lint result.

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()

for await (const result of eslint.executeOnFiles(patterns)) {
  console.log(result.usedDeprecatedRules)
}
```

As a side-effect, formatters gets the capability to print the used deprecated rules. (!?)

##### Fail-fast on parallel calling

Because this method updates the cache file, if people call this method in parallel then it causes broken. To prevent the broken, it should throw an error if people called this method while the previous call is still running.

##### Implementation

<details>
<summary>A rough sketch of the `executeOnFiles()` method.</summary>

A tiny wrapper of `CLIEngine`.

```js
class ESLint {
  executeOnFiles(patterns) {
    let promise
    try {
      const report = this.cliEngine.executeOnFiles(patterns)
      promise = Promise.resolve(report.results)
    } catch (error) {
      promise = Promise.reject(error)
    }

    return {
      then(onFulfilled, onRejected) {
        return promise.then(onFulfilled, onRejected)
      },
      async *[Symbol.asyncIterator]() {
        yield* await promise
      },
    }
  }
}
```

</details>

#### § The `executeOnText()` method

This method returns the same type of an object as the `executeOnFiles()` method.

Because the returned object of `CLIEngine#executeOnText()` method is the same type as the `CLIEngine#executeOnFiles()` method. The `ESLint` class inherits that mannar.

<details>
<summary>Example: Show the result.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()

const [result] = await eslint.executeOnText(text, filePath)
print(result)
```

</details>

<details>
<summary>Example: Using along with the `executeOnFiles()` method.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()

const report = useStdin
  ? eslint.executeOnText(text, filePath)
  : eslint.executeOnFiles(patterns)

for await (const result of report) {
  print(result)
}
```

</details>

#### § The `getFormatter()` method

This method returns a `Promise<Formatter>`. The `Formatter` type is a function `(results: AsyncIterable<LintResult>) => AsyncIterableIterator<string>`. It receives lint results then outputs the formatted text.

This means the `getFormatter()` method wraps the current formatter to adapt the interface.

<details>
<summary>A rough sketch of the `getFormatter()` method.</summary>

```js
class ESLint {
  async getFormatter(name) {
    const format = this.cliEngine.getFormatter(name)

    // Return the wrapper.
    return async function* formatter(resultIterator) {
      // Collect and sort the results.
      const results = []
      for await (const result of resultIterator) {
        results.push(result)
      }
      results.sort(ESLint.compareResultsByFilePath)

      // Make `rulesMeta`.
      const rules = this.cliEngine.getRules()
      const rulesMeta = getRulesMeta(rules)

      // Format the results with the formatter of the current spec.
      yield format(results, { rulesMeta })
    }
  }
}
```

</details>

<details>
<summary>Example: Use the formatter.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()
const formatter = eslint.getFormatter("stylish")

// Verify files
const results = eslint.executeOnFiles(patterns)
// Format and write the results
for await (const textPiece of formatter(results)) {
    process.stdout.write(textPiece)
}
```

</details>

Once the `getFormatter()` method got this change, we can update the specification of custom formatters without breakage in the future to support streaming.

#### § The other methods

The following methods receive an `AsyncIterable<LintResult>` object as the first argument then return an `AsyncIterableIterator<LintResult>` object because those methods are sandwiched between `executeOnFiles()` and formatters.

- `outputFixes()`
- `getErrorResults()`

<details>
<summary>Example: Use `outputFixes()` and `getErrorResults()`.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()
const formatter = eslint.getFormatter("stylish")

// Verify files
let results = eslint.executeOnFiles(patterns)
// Update the files of the results if needed
// (This must be done before `--quiet`)
if (process.argv.includes("--fix")) {
    results = ESLint.outputFixes(results)
}
// Filter the results if needed
if (process.argv.includes("--quiet")) {
    results = ESLint.getErrorResults(results)
}
// Format and write the results
for await (const textPiece of formatter(results)) {
    process.stdout.write(textPiece)
}
```

</details>

The following methods return `Promise` which gets fulfilled with each result in order to support plugins/configs that are ES modules without breakage in the future.

- `getConfigForFile()`
- `isPathIgnored()`

The following methods are as-is because those don't touch both file system and module system.

- `addPlugin()`
- `getRules()`

The following method is removed because it doesn't fit the current API.

- `resolveFileGlobPatterns()` ... ESLint doesn't use this logic since `v6.0.0`, but it has stayed there for backward compatibility. Once [RFC 20](https://github.com/eslint/rfcs/tree/master/designs/2019-additional-lint-targets) is implemented, what ESLint iterates and what the glob of this method iterates will be different, then it will confuse users. This is good timing to remove the legacy.

#### § A new `compareResultsByFilePath` static method

This method receives two lint results then returns `+1`, `-1`, or `0`. This method is intended to use in order to sort results.

<details>
<summary>A rough sketch of the `compareResultsByFilePath()` method.</summary>

```js
class ESLint {
  static compareResultsByFilePath(a, b) {
    if (a.filePath < b.filePath) {
      return -1
    }
    if (a.filePath > b.filePath) {
      return 1
    }
    return 0
  }
}
```

</details>

### ■ Deprecate `CLIEngine` class

This RFC soft-deprecates `CLIEngine` class.

Because it's tough to maintain two versions (sync and async) of implementation. The two are almost copy-pasted stuff, but hard to share the code. Therefore, this RFC deprecates the sync version to improve our code with the way which needs asynchronous behavior in the future. For example, `CLIEngine` cannot support parallel linting, plugins/configs as ES modules, etc...

### ■ Out of scope

- Not change API for rules. This RFC doesn't change APIs that rule implementation uses. We may be able to support asynchronous stuff in rules in the future, but it's out of this RFC's scope.
- Not change internal logics. This RFC just adds the public interface that is asynchronous. It would be a wrapper of `CLIEngine` for now.

## Documentation

- The "[Node.js API](https://eslint.org/docs/developer-guide/nodejs-api)" page should describe the new public API and deprecation of `CLIEngine` class.

## Drawbacks

People that use `CLIEngine` have to update their application with the new API. It will need hard work.

## Backwards Compatibility Analysis

This is a breaking change.

Deprecating `CLIEngine` is a drastic change. But people can continue to use `CLIEngine` as-is until we decide to remove it.

The new API depends on [Asynchronous Iteration](https://github.com/tc39/proposal-async-iteration) syntax. Node.js supports the syntax since `10.0.0`, so we have to drop Node.js `8.x`. Because the `8.x` will be EOL in December 2019 (two months later!), we can work on this soon.

## Alternatives

- Adding `engine.executeAsyncOnFiles()`-like methods and we maintain it along with the existing synchronous API. But as what I wrote in the "[Deprecate `CLIEngine` class](#deprecate-cliengine-class)" section, it would be tough.
- Using [Streams](https://nodejs.org/dist/latest-v12.x/docs/api/stream.html) instead of [Asynchronous Iteration](https://github.com/tc39/proposal-async-iteration). We can introduce `ESLint` class in a minor release if we used Streams. But because Node.js 8 will be EOL two months later, we should be able to use Asynchronous Iteration soon. And Streams don't support `for-await-of` syntax until `v11.14.0`. Iterator protocol is smaller spec than streams, and it's easy to use.

## Related Discussions

- https://github.com/eslint/eslint/issues/3565 - Lint multiple files in parallel
- https://github.com/eslint/eslint/issues/12319 - `ERR_REQUIRE_ESM` when requiring `.eslintrc.js`
- https://github.com/eslint/rfcs/pull/4 - New: added draft for async/parallel proposal
- https://github.com/eslint/rfcs/pull/11 - New: Lint files in parallel
