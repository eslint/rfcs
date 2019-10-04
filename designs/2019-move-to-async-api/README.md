- Start Date: 2019-09-28
- RFC PR: https://github.com/eslint/rfcs/pull/40
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Moving to Asynchronous API

## Summary

This RFC adds a new class `ESLint` that has asynchronous API and deprecates `CLIEngine`.

## Motivation

- Dynamic `import()` has arrived at Stage 4. The dynamic loading of ES modules requires asynchronous API. Migrating to asynchronous API opens up doors to write plugins/configs with ES modules.
- Linting in parallel requires asynchronous API. We can improve linting logic to run it in parallel. And we can improve config loading logic and file enumeration logic to run it in parallel. (E.g., load `extends` list, traverse child directories.)

Because the migration of public API needs long time, we should start to migrate our APIs earlier.

And the name of `CLIEngine`, our primary API, causes confusion to the community. People try `Linter` class at first because the name sounds the main class of ESLint, then they notice it doesn't work as expected. We have a lot of issues that say "please use `CLIEngine` instead."
The name of new class, `ESLint`, is our primary API clearly.

## Detailed Design

### ■ Add new `ESLint` class

This RFC adds a new class `ESLint`. It has almost the same methods as `CLIEngine`, but the return value of some methods are different.

So, for now, `ESLint` class will be a tiny wrapper of `CLIEngine` that modifies the type of the return values.

#### § The `executeOnFiles()` method

This method returns a `AsyncIterator<LintResult>` object iterates the lint result of each file in random order.

<details>
<summary>A rough sketch of the `executeOnFiles()` method.</summary>

```js
class ESLint {
  async *executeOnFiles(patterns) {
    // Verify files and push the results.
    for (const result of this.cliEngine.executeOnFiles(patterns).results) {
      yield result
    }
  }
}
```

</details>

<details>
<summary>Example: Show the results step by step.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()

for await (const result of eslint.executeOnFiles(patterns)) {
  print(result)
}
```

</details>

<details>
<summary>Example: Show the results in the stable order.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()
const results = []

for await (const result of eslint.executeOnFiles(patterns)) {
  results.push(result)
}

results.sort(byFilePath)
print(results)
```

</details>

Once the `executeOnFiles()` method got this change, we can support "linting in parallel", streaming, and plugins/configs which are ES modules in the future.

⚠️ Because this method updates the cache file, if people call this method in parallel then it causes broken. To prevent the broken, it should throw an error if people called this method while the previous call is still running.

#### § The `getFormatter()` method

This method returns a `Promise<Formatter>`. The `Formatter` type is a function `(results: AsyncIterator<LintResult>) => AsyncIterator<string>`. It receives lint results then outputs the formatted text.

This means the `getFormatter()` method wraps the current formatter to align the interface.

<details>
<summary>A rough sketch of the `getFormatter()` method.</summary>

```js
class ESLint {
  async getFormatter(name) {
    const format = this.cliEngine.getFormatter(name)

    // Return the wrapper.
    return async function* formatter(resultIterator) {
      // Collect the results.
      const results = []
      for await (const result of resultIterator) {
        results.push(result)
      }
      results.sort(byFilePath)

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

#### § The `getErrorResults()` method

As related to the above, this method now receives an `AsyncIterator<LintResult>` object and returns an `AsyncIterator<LintResult>` object. Because this method is sandwiched between `executeOnFiles()` and formatters.

<details>
<summary>A rough sketch of the `getErrorResults()` static method.</summary>

```js
class ESLint {
  static async *getErrorResults(results) {
    for await (const result of results) {
      const messages = result.messages.filter(m => m.severity === 2)

      if (messages.length === result.messages.length) {
        yield result
      }
      yield {
        ...result,
        messages,
        warningCount: 0,
        fixableWarningCount: 0,
      }
    }
  }
}
```

</details>

<details>
<summary>Example: Use `getErrorResults()`.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()
const formatter = eslint.getFormatter("stylish")

// Verify files
let results = eslint.executeOnFiles(patterns)
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

#### § The other methods

The following methods return `Promise` which gets fulfilled with each result.

- `executeOnText()`
- `getConfigForFile()`
- `isPathIgnored()`
- `outputFixes()`

Once the former three methods got this change, we can support plugins/configs that are ES modules without breakage in the future. And once the `outputFixes()` method got this change, we can write many files more efficiently in the future.

The following methods are as-is because those don't touch both file system and module system.

- `addPlugin()`
- `getRules()`

The following methods are removed because those don't fit the current API.

- `resolveFileGlobPatterns()` ... ESLint doesn't use this logic since `v6.0.0`, but it has stayed there for backward compatibility. Once [RFC 20](https://github.com/eslint/rfcs/tree/master/designs/2019-additional-lint-targets) is implemented, what ESLint iterates and what the glob of this method iterates will be different, then it will confuse users. This is good timing to remove the legacy.

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
- Using [Streams](https://nodejs.org/dist/latest-v12.x/docs/api/stream.html) instead of [Asynchronous Iteration](https://github.com/tc39/proposal-async-iteration). We can introduce `ESLint` class in a minor release if we used Streams. But because Node.js 8 will be EOL two months later, we should be able to use Asynchronous Iteration soon. Iterator protocol is smaller spec than streams, and it's easy to use.

## Related Discussions

- https://github.com/eslint/eslint/issues/3565 - Lint multiple files in parallel
- https://github.com/eslint/eslint/issues/12319 - `ERR_REQUIRE_ESM` when requiring `.eslintrc.js`
- https://github.com/eslint/rfcs/pull/4 - New: added draft for async/parallel proposal
- https://github.com/eslint/rfcs/pull/11 - New: Lint files in parallel