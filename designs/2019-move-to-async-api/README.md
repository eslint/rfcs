- Start Date: 2019-09-28
- RFC PR: https://github.com/eslint/rfcs/pull/40
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# `ESLint` Class Replacing `CLIEngine`

## Summary

This RFC adds a new class `ESLint` that provides asynchronous API and deprecates `CLIEngine`.

## Motivation

- We have functionality that cannot be supported with the current synchronous API. For example, ESLint verifying files in parallel, formatters printing progress state, formatters printing results in streams etc. A move to an asynchronous API would be beneficial and a new `ESLint` class can be created with an async API in mind from the start.
- Node.js will support [ES modules](https://nodejs.org/api/esm.html) stably on `13.0.0` at last. Node.js doesn't provide any way that loads ES modules synchronously from CJS. This means that ESLint (CJS) cannot load configs/plugins that are written as ES modules synchronously. Migrating to asynchronous API opens up doors to support those.
- The name of `CLIEngine`, our primary API, has caused confusion in the community and is sub-optimal. We have a lot of issues that say "please use `CLIEngine` instead.". A new class, `ESLint`, while fixing other issues, will also make our primary API more clear.

## Detailed Design

### Add new `ESLint` class

This RFC adds a new class `ESLint`. It has almost the same methods as `CLIEngine`, but the return value of some methods are different.

- [constructor()](#-constructor)
- [executeOnFiles()](#-the-executeonfiles-method)
- [executeOnText()](#-the-executeontext-method)
- [getFormatter()](#-the-getformatter-method)
- [static outputFixesInIteration()](#-the-outputfixesiniteration-method) (rename)
- [getConfigForFile()](#-the-other-methods)
- [getRules()](#-the-other-methods)
- [isPathIgnored()](#-the-other-methods)
- [static compareResultsByFilePath()](#-new-methods) (new)

Initially the `ESLint` class will be a wrapper around `CLIEngine`, modifying return types. Later it can take on a more independent shape as `CLIEngine` gets more deprecated.

#### ● Constructor

The constructor has mostly the same options as `CLIEngine`, but with some differences:

- It throws fatal errors if the options contain unknown properties or an option is invalid type ([eslint/eslint#10272](https://github.com/eslint/eslint/issues/10272)).
- It disallows the deprecated `cacheFile` option.
- It has a new `errorsOnly` option as the successor of `getErrorResults()` method. This option is a boolean value and defaults to `false`. If `true` is present then ESLint disables the rules which are configured with the warning severity. This option corresponds to [`--quiet` CLI option](https://eslint.org/docs/user-guide/command-line-interface#--quiet). See also "[The other methods](#-the-other-methods)" section.
- It has a new `pluginImplementations` option as the successor of `addPlugin()` method. This is an object that keys are plugin IDs and each value is the plugin object. See also "[The other methods](#-the-other-methods)" section.

##### Implementation

<details>
<summary>A rough sketch of the constructor.</summary>

```js
class ESLint {
  constructor({
    allowInlineConfig = true,
    baseConfig = null,
    cache = false,
    cacheLocation = ".eslintcache",
    configFile = null,
    cwd = process.cwd(),
    envs = [],
    errorsOnly = false,
    extensions = [".js"],
    fix = false,
    fixTypes = ["problem", "suggestion", "layout"],
    globInputPaths = true,
    globals = [],
    ignore = true,
    ignorePath = null,
    ignorePattern = [],
    parser = "espree",
    parserOptions = null,
    pluginImplementations = null,
    plugins = [],
    reportUnusedDisableDirectives = false,
    resolvePluginsRelativeTo = cwd,
    rulePaths = [],
    rules = null,
    useEslintrc = true,
    ...unknownOptions
  } = {}) {
    // Throws on unknown options
    if (Object.keys(unknownOptions).length >= 1) {
      //...
    }
    // Throws on the invalid value of options
    if (typeof allowInlineConfig !== "boolean") {
      // ...
    }
    if (typeof baseConfig !== "object") {
      // ...
    }
    // and other options...

    // Initialize CLIEngine because this is a tiny wrapper.
    const engine = (this._cliEngine = new CLIEngine({
      allowInlineConfig,
      baseConfig,
      cache,
      cacheLocation,
      configFile,
      cwd,
      envs,
      extensions,
      fix,
      fixTypes,
      globInputPaths,
      globals,
      ignore,
      ignorePath,
      ignorePattern,
      parser,
      parserOptions,
      plugins,
      reportUnusedDisableDirectives,
      resolvePluginsRelativeTo,
      rulePaths,
      rules,
      useEslintrc,
    }))

    // Apply `pluginImplementations` option.
    if (pluginImplementations) {
      for (const [id, definition] of Object.entries(pluginImplementations)) {
        engine.addPlugin(id, definition)
      }
    }
  }
}
```

</details>

#### ● The `executeOnFiles()` method

This method returns an object that has two methods `then()` and `[Symbol.asyncIterator]()`, so we can use the returned object with `await` expression and `for-await-of` statement.

- If you used the returned object with `for-await-of` statement, it iterates the lint result of each file in random order. This way yields each lint result immediately. Therefore you can print the results in streaming, or print progress state. ESLint may spend time to lint files (for example, ESLint needs about 20 seconds to lint [our codebase](https://github.com/eslint/eslint)), to print progress state will be useful.

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

Either way, we can support [linting in parallel](https://github.com/eslint/rfcs/pull/42), loading configs/plugins with `import()`, and other stuff that needs asynchronous logic.

This method must not throw any errors synchronously.

##### Iterate only one time

We can iterate the returned object of this method only one time similar to generators.

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()

const resultGenerator = eslint.executeOnFiles(patterns)
for await (const result of resultGenerator) {
  print(result)
}
// ↓ Throw "This generator has been consumed already"
for await (const result of resultGenerator) {
  print(result)
}
```

##### Move the `usedDeprecatedRules` property

The returned object of `CLIEngine#executeOnFiles()` has the `usedDeprecatedRules` property that includes the deprecated rule IDs which the linting used.

But the location doesn't fit asynchronous because the used deprecated rule list is not determined until the iteration finished. Therefore, this RFC moves the `usedDeprecatedRules` property to each lint result.

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()

for await (const result of eslint.executeOnFiles(patterns)) {
  console.log(result.usedDeprecatedRules)
}
```

As a side-effect, formatters gets the capability to print the used deprecated rules. Previously, ESLint has not passed the returned object to formatters, so the formatters could not print used deprecated rules. After this RFC, each lint result has the `usedDeprecatedRules` property and the formatters receive those.

##### Disallow execution in parallel

Because this method updates the cache file, it will break the cache file if called multiple times in parallel. To prevent that, every call of `executeOnFiles()` must wait for the previous call finishes.

##### Abort linting

The iterator protocol has optional [`return()` method](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/return) that forces to finish the iterator. The `for-of`/`for-await-of` syntax calls the `return()` method automatically if the loop is stopped through a `braek`, `return`, or `throw`.

ESLint aborts linting when the `return()` method is called. The first `return()` method call does:

- ESLint cancels the linting of all pending files.
- ESLint updates the cache file with the current state. Therefore, the next time, ESLint can use the cache of the already linted files and lint only the canceled files.
- ESLint will terminate all workers if [RFC42](https://github.com/eslint/rfcs/pull/42) is implemented.

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()

for await (const result of eslint.executeOnFiles(patterns)) {
  if (Math.random() < 0.5) {
    break // abort linting.
  }
}
```

The second and later calls do nothing.

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

But once [RFC42](https://github.com/eslint/rfcs/pull/42) is implemented, the returned object will be an instance of [`LintResultGenerator`](https://github.com/eslint/eslint/blob/836c0e48704d70bc1a5cbdbf0211368b0ada942d/lib/eslint/lint-result-generator.js#L136) class. The class implements async iterable protocol, async iterator protocol, and thenable.

</details>

#### ● The `executeOnText()` method

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

#### ● The `getFormatter()` method

This method returns a `Promise<Formatter>`. The `Formatter` type is a function `(results: AsyncIterable<LintResult>) => AsyncIterableIterator<string>`. It receives lint results then outputs the formatted text.

This means the `getFormatter()` method wraps the current formatter to adapt the interface.

Once the `getFormatter()` method got this change, in the future, we can update the specification of custom formatters without breakage to support streaming.

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

#### ● The `outputFixesInIteration()` method

The original `CLIEngine.outputFixes()` static method writes the fix results to the source code files.

The goal of this method is same as the `CLIEngine.outputFixes()` method, but we cannot share async iterators with this method and formatters, so this method receives an `AsyncIterable<LintResult>` object as the first argument then return an `AsyncIterableIterator<LintResult>` object. This method is sandwiched between `executeOnFiles()` and formatters.

<details>
<summary>Example: Use `outputFixesInIteration()`.</summary>

```js
const { ESLint } = require("eslint")
const eslint = new ESLint()
const formatter = eslint.getFormatter("stylish")

// Verify files
let results = eslint.executeOnFiles(patterns)
// Update the files of the results if needed
if (process.argv.includes("--fix")) {
    results = ESLint.outputFixesInIteration(results)
}
// Format and write the results
for await (const textPiece of formatter(results)) {
    process.stdout.write(textPiece)
}
```

</details>

#### ● The other methods

The following methods return `Promise` which gets fulfilled with each result in order to support plugins/configs that are ES modules without breakage in the future.

- `getConfigForFile()`
- `getRules()`
- `isPathIgnored()`

The following methods are removed because those don't fit the new API.

- `addPlugin()` ... This method has caused to confuse people. We have introduced this method to add plugin implementations and expected people to use this method to test plugins, but people have thought that this method loads a new plugin for the following linting. And this method is only one that mutates the state of `CLIEngine` objects (except cache) and messes all caches. Therefore, this RFC moves this functionality to a constructor option. See also "[Constructor](#-constructor)" section.
- `getErrorResults()` ... This method is a utility to filter warning messages and is used for `--quiet` CLI option. But disabling the rules which are configured with the warning severity is more efficient than the current way. Therefore, this RFC moves this functionality to a constructor option. See also "[Constructor](#-constructor)" section.
- `resolveFileGlobPatterns()` ... ESLint doesn't use this logic since `v6.0.0`, but it has stayed there for backward compatibility. Once [RFC20](https://github.com/eslint/rfcs/tree/master/designs/2019-additional-lint-targets) is implemented, what ESLint iterates and what the glob of this method iterates will be different, then it will confuse users. This is good timing to remove the legacy.

#### ● A new `compareResultsByFilePath` static method

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

Because it's tough to maintain two versions (sync and async) of implementation. The two are almost copy-pasted stuff, but hard to share the code. Therefore, this RFC deprecates the sync version to improve our code with the way which needs asynchronous behavior in the future. For example, `CLIEngine` cannot support [linting in parallel](https://github.com/eslint/rfcs/pull/42), plugins/configs as ES modules, etc...

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

### Alternatives for the new class

Adding `CLIEngine#executeOnFilesAsync()` method is an alternative.

**Pros:**

- It's smaller change than adding `ESLint` class.

**Cons:**

- We have to maintain both synchronous and asynchronous APIs. It's kind of hard works.
  - We can deprecate the synchronous version, but in that case, I feel odd because `executeOnFiles()` is the essential name of `executeOnFilesAsync()`.
- We need the asynchronous version of the other APIs in the future. The `getFormatterAsync()` is needed for formatters with progress. The asynchronous version of the other methods is needed for ES modules.
  - `CLIEngine` will get huge because we cannot share the most code between asynchronous API and synchronous API.
  - API users need the number of API migrations.
- It causes confusion for API users. As Node.js built-in libraries are so, I guess that people expect the two (`executeOnFiles()` and `executeOnFilesAsync()`) to have exactly the same features. However, it's impossible. We have a bundle of the features that we cannot support on synchronous API, such as linting in parallel, formatters with progress, ES modules, etc.

Therefore, I think that introducing the new class that has only asynchronous API makes more sense. This RFC solves:

- We can freeze the code of the synchronous version of our API by deprecating `CLIEngine`. This reduces the maintenance cost of duplicate codes.
- We can reduce the number of API migrations for API users.
- We can reduce the confusion of similar but different methods.
- And as a bonus, we can reduce the confusion of the name of `CLIEngine`.

### Alternatives for [Asynchronous Iteration](https://github.com/tc39/proposal-async-iteration)

Using [streams](https://nodejs.org/dist/latest-v12.x/docs/api/stream.html) instead is an alternative.

**Pros:**

- We can introduce `ESLint` class in a minor release. (To use Async Iteration, we have to wait for [RFC44](https://github.com/eslint/rfcs/pull/44)).

**Cons:**

- Streams are problematic a bit about error propagation.
  - [straem.pipeline()](https://nodejs.org/api/stream.html#stream_stream_pipeline_streams_callback) function reduces this pain, but [it doesn't cover all cases](https://github.com/nodejs/node/issues/26311).
- We have to wait for Node.js `v11.14.0` to use streams with `for-await-of` syntax.

Because Node.js 8 will be EOL two months later, we should be able to use Asynchronous Iteration soon. And Iterator protocol is smaller spec than streams, and it's easy to use.

### Alternatives for disallow execution in parallel

- Throwing if the previous call has not finished yet (fail-fast).
- Aborting the previous call then run (steal ownership of the cache file).

are alternatives.

Both cases stop the previous or current call. It may be surprising users.

The access of the cache file finishes regardless of the progress of the iterator that the method returned. It means that API users cannot know when the file access finished. On the other hand, because `ESLint` objects know when the file access finished, it can execute the next call at the proper timing.

## Related Discussions

- https://github.com/eslint/eslint/issues/1098 - Show results for each file as eslint is running
- https://github.com/eslint/eslint/issues/3565 - Lint multiple files in parallel
- https://github.com/eslint/eslint/issues/10272 - Validate options passed to CLIEngine API
- https://github.com/eslint/eslint/issues/10606 - Make CLIEngine and supporting functions async via Promises
- https://github.com/eslint/eslint/issues/12319 - `ERR_REQUIRE_ESM` when requiring `.eslintrc.js`
- https://github.com/eslint/rfcs/pull/4 - New: added draft for async/parallel proposal
- https://github.com/eslint/rfcs/pull/11 - New: Lint files in parallel
- https://github.com/eslint/rfcs/pull/42 - New: Lint files in parallel if many files exist
