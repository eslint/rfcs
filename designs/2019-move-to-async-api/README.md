- Start Date: 2019-09-28
- RFC PR: https://github.com/eslint/rfcs/pull/40
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# `LinterShell` Class Replacing `CLIEngine`

## Summary

This RFC adds a new class `LinterShell` that provides asynchronous API and deprecates `CLIEngine`.

## Motivation

- We have functionality that cannot be supported with the current synchronous API. For example, ESLint verifying files in parallel. A move to an asynchronous API would be beneficial and a new `LinterShell` class can be created with an async API in mind from the start.
- Node.js has supported [ES modules](https://nodejs.org/api/esm.html) stably since `13.2.0`. Because Node.js doesn't provide any way that loads ES modules synchronously from CJS, ESLint cannot load configs/plugins that are written as ES modules. And migrating to asynchronous API opens up doors to support those.
- The name of `CLIEngine`, our primary API, has caused confusion in the community and is sub-optimal. We have a lot of issues that say "please use `CLIEngine` instead.". A new class, `LinterShell`, while fixing other issues, will also make our primary API more clear.

## Detailed Design

### Add new `LinterShell` class

This RFC adds a new class `LinterShell`. It has almost the same methods as `CLIEngine`, but the return value of some methods are different.

- [constructor()](#-constructor)
- [lintFiles()](#-the-lintfiles-method) (rename from `executeOnFiles()`)
- [lintText()](#-the-linttext-method) (rename from `executeOnText()`)
- [getFormatter()](#-the-getformatter-method)
- [getConfigForFile()](#-the-other-methods)
- [isPathIgnored()](#-the-other-methods)
- [static outputFixes()](#-the-other-methods)
- [static getErrorResults()](#-the-other-methods)
- ~~addPlugin()~~ (move to a constructor option)
- ~~getRules()~~ (delete)
- ~~resolveFileGlobPatterns()~~ (delete)
- [static compareResultsByFilePath()](#-new-methods) (new)

Initially the `LinterShell` class will be a wrapper around `CLIEngine`, modifying return types. Later it can take on a more independent shape as `CLIEngine` gets more deprecated.

#### ● Constructor

The constructor has mostly the same options as `CLIEngine`, but with some differences:

- It throws fatal errors if the options contain unknown properties or an option is invalid type ([eslint/eslint#10272](https://github.com/eslint/eslint/issues/10272)).
- It disallows the deprecated `cacheFile` option.
- The array of the `plugins` option can contain objects `{ id: string; definition: Object }` along with strings. If the objects are present, the `id` property is the plugin ID and the `definition` property is the definition of the plugin. This is the successor of `addPlugin()` method. See also "[The other methods](#-the-other-methods)" section.

<details>
<summary>A rough sketch of the constructor.</summary>

```js
class LinterShell {
  constructor({
    allowInlineConfig = true,
    baseConfig = null,
    cache = false,
    cacheLocation = ".eslintcache",
    configFile = null,
    cwd = process.cwd(),
    envs = [],
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
      plugins: plugins.map(p => (typeof p === "string" ? p : p.id)),
      reportUnusedDisableDirectives,
      resolvePluginsRelativeTo,
      rulePaths,
      rules,
      useEslintrc,
    }))

    // Add the definitions of the `plugins` option.
    if (plugins) {
      for (const plugin of plugins) {
        if (typeof plugin === "object" && plugin !== null) {
          engine.addPlugin(plugin.id, plugin.definition)
        }
      }
    }
  }
}
```

</details>

<details>
<summary>For `plugins` example.</summary>

```js
const { LinterShell } = require("eslint")
const linter = new LinterShell({
  plugins: [
    "foo",
    "eslint-plugin-bar",
    { id: "abc", definition: require("./path/to/a-plugin") },
  ],
})
```

</details>

#### ● The `lintFiles()` method

##### Parameters

| Name       | Type       | Description                         |
| :--------- | :--------- | :---------------------------------- |
| `patterns` | `string[]` | The glob patterns for target files. |

##### Return Value

This method returns a Promise object that will be fulfilled with the array of lint results.

##### Description

This method corresponds to `CLIEngine#executeOnFiles()`.

```js
const { LinterShell } = require("eslint")
const linter = new LinterShell()

for (const result of await linter.lintFiles(patterns)) {
  print(result)
}
```

ESLint doesn't guarantee the order of the lint results in the array. If we implemented parallel linting, the result of the file that finished linting earlier will be in the front of the array. For backward compatibility, the wrapper of formatters sorts the results then gives the formatters the sorted results. See also [getFormatter()](#-the-getformatter-method). And we provide [compareResultsByFilePath()](#-new-methods) method to sort the results in the same order as `CLIEngine`.

This method must not throw any errors synchronously. If an error happened, the returned promise goes rejected.

<details>
<summary>A rough sketch of the `lintFiles()` method.</summary>

A tiny wrapper of `CLIEngine`.

```js
class LinterShell {
  async lintFiles(patterns) {
    return this._cliEngine.executeOnFiles(patterns).results
  }
}
```

</details>

Once we got this change, we can realize the following things:

- [RFC42] We can implement linting in parallel by worker threads. It will reduce spending time of linting much.
- (no RFC yet) We can support ES modules for shareable configs, plugins, and custom parsers.

##### Move the `usedDeprecatedRules` property

The returned object of `CLIEngine#executeOnFiles()` has the `usedDeprecatedRules` property that includes the deprecated rule IDs which the linting used.

But the location is problematic because it requires the plugin uniqueness in spanning all files. Therefore, this RFC moves the `usedDeprecatedRules` property to each lint result.

```js
const { LinterShell } = require("eslint")
const linter = new LinterShell()

for (const result of await linter.lintFiles(patterns)) {
  console.log(result.usedDeprecatedRules)
}
```

As a side-effect, formatters gets the capability to print the used deprecated rules. Previously, ESLint has not passed the returned object to formatters, so the formatters could not print used deprecated rules. After this proposal, each lint result has the `usedDeprecatedRules` property and the formatters receive those.

##### Write cache safely (best effort)

Because this method updates the cache file, it will break the cache file if called multiple times in parallel. To prevent that, this method doesn't write the cache file if the cache file has been updated since this method read.

This method does this check with the best effort (e.g., check `mtime` of the cache file) because Node.js doesn't provide the way that reads/writes a file exclusively.

If the cache file was broken, this method should ignore the cache file and does lint.

##### Abort linting

This proposal doesn't provide the way to abort linting because we cannot abort linting at this time (the method does linting synchronously internally). We can discuss it along with parallel linting (I'm guessing we can use [AbortSignal] that is the Web Standard. Passing it as `options.signal`).

#### ● The `lintText()` method

##### Parameters

| Name                  | Type      | Description                                                      |
| :-------------------- | :-------- | :--------------------------------------------------------------- |
| `code`                | `string`  | The source code to lint.                                         |
| `options`             | `Object`  | Optional. The options.                                           |
| `options.filePath`    | `string`  | Optional. The path to the file of the source code.               |
| `options.warnIgnored` | `boolean` | Optional. If `true`, it warns the `filePath` is an ignored path. |

##### Return Value

This method returns a Promise object that will be fulfilled with the array of lint results.

##### Description

This method corresponds to `CLIEngine#executeOnText()`.

Because the returned object of `CLIEngine#executeOnText()` method is the same type as the `CLIEngine#executeOnFiles()` method, the `LinterShell` class also returns the same type. Therefore, the returned value is an array that contains one result.

```js
const { LinterShell } = require("eslint")
const linter = new LinterShell()

for (const result of await linter.lintText(text, filePath)) {
  print(result)
}
```

Example: Using along with the `lintFiles()` method.

```js
const { LinterShell } = require("eslint")
const linter = new LinterShell()

const results = await (useStdin
  ? linter.lintText(text, filePath)
  : linter.lintFiles(patterns))

for (const result of results) {
  print(result)
}
```

#### ● The `getFormatter()` method

##### Parameters

| Name   | Type     | Description               |
| :----- | :------- | :------------------------ |
| `name` | `string` | The formatter ID to load. |

##### Return Value

This method returns a Promise object that will be fulfilled with the wrapper of the loaded formatter. The wrapper is the object that has `format` method.

| Name     | Type                                | Description                                        |
| :------- | :---------------------------------- | :------------------------------------------------- |
| `format` | `(results: LintResult[]) => string` | The function that converts lint results to output. |

##### Description

This method corresponds to `CLIEngine#getFormatter()`.

But as different from that, it returns a wrapper object instead of the loaded formatter. Because we have experienced withdrawn some features because of breaking `getFormatter()` API in the past. The wrapper glues `getFormatter()` API and formatters.

```js
const { LinterShell } = require("eslint")
const linter = new LinterShell()
const formatter = await linter.getFormatter("stylish")

// Verify files
const results = await linter.lintFiles(patterns)
// Format and write the results
process.stdout.write(formatter.format(results))
```

Currently, the wrapper does:

1. sort given lint results.
1. create `rulesMeta`.

```js
class LinterShell {
  async getFormatter(name) {
    const format = this._cliEngine.getFormatter(name)

    return {
      format(results) {
        let rulesMeta = null

        results.sort(LinterShell.compareResultsByFilePath)

        return format(results, {
          get rulesMeta() {
            if (!rulesMeta) {
              rulesMeta = createRulesMeta(this._cliEngine.getRules())
            }
            return rulesMeta
          },
        })
      },
    }
  }
}
```

Once we got this change, we can realize the following things:

- (no RFC yet) We can support ES modules for custom formatters.

#### ● The other methods

The following methods return `Promise` which gets fulfilled with each result. Once we got this change, we can support ES modules for shareable configs, plugins, and custom parsers without more breaking changes.

- `getConfigForFile()`
- `isPathIgnored()`

The following methods return a `Promise`. Once we got this change, we can use asynchronous `fs` API to write files.

- `static outputFixes()`

The following methods are as-is.

- `static getErrorResults()`

The following methods are removed because those don't fit the new API.

- `addPlugin()` ... This method has caused to confuse people. We have introduced this method to add plugin implementations and expected people to use this method to test plugins. But people have often thought that this method loads a new plugin for the following linting so they can use plugins rules without `plugins` setting. And this method is only one that mutates the state of `CLIEngine` objects and messes all caches. Therefore, this RFC moves this functionality to a constructor option. See also "[Constructor](#-constructor)" section.
- `getRules()` ... This method returns the map that contains core rules and the rules of the plugin that the previous `executeOnFiles()` method call used. This behavior is surprised and forces us to store the config objects that the previous `executeOnFiles()` method call used. This proposal removes this method and a separated RFC (maybe [RFC47]) will add the successor.
- `resolveFileGlobPatterns()` ... ESLint doesn't use this logic since `v6.0.0`, but it has stayed there for backward compatibility. Once [RFC20] is implemented, what ESLint iterates and what the glob of this method iterates will be different, then it will confuse users. This is good timing to remove the legacy.

#### ● New methods

- `static compareResultsByFilePath()` ... This method receives two lint results then returns `+1`, `-1`, or `0`. This method is intended to use in order to sort results.

  <details>
  <summary>A rough sketch of the `compareResultsByFilePath()` method.</summary>

  ```js
  class LinterShell {
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

This RFC soft-deprecates `CLIEngine` class. Because:

- It's tough to maintain two versions (sync and async) of implementation. The two are almost copy-pasted stuff, but hard to share the code. We can freeze the synchronous version of code by deprecation.
- In the future, `CLIEngine` get not-supported features such as [RFC42], ES modules, etc because of synchronous API. This difference may be surprising for API users, but we can explain that as "Because `CLIEngine` has been deprecated, we don't add any new features into that class."

### ■ Out of scope

- Not change API for rules. This RFC doesn't change APIs that rule implementation uses. We may be able to support asynchronous stuff in rules in the future, but it's out of this RFC's scope.
- Not change internal logics. This RFC just adds the public interface that is asynchronous. It would be a wrapper of `CLIEngine` for now.

## Documentation

- It needs an entry in the migration guide.
- The "[Node.js API](https://eslint.org/docs/developer-guide/nodejs-api)" page should describe the new public API and deprecation of `CLIEngine` class.

## Drawbacks

People that use `CLIEngine` have to update their application with the new API. It will need hard work.

## Backwards Compatibility Analysis

This RFC is a drastic change, but not a breaking change until we decide to remove `CLIEngine` class.

This RFC just adds `LinterShell` class and deprecates `CLIEngine` class. We can do both in a minor release.

## Alternatives

### Alternatives for the new class

Adding `CLIEngine#executeOnFilesAsync()` method is an alternative.

**Pros:**

- It's smaller change than adding `LinterShell` class.

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

### Alternatives for disallow execution in parallel

- Throwing if the previous call has not finished yet (fail-fast).
- Aborting the previous call then run (steal ownership of the cache file).

are alternatives. Both cases stop the previous or current call. It may be surprising users.

- Waiting the previous call internally.

It will work fine in a thread. However, the `LinterShell` objects cannot know the existence of other threads and processes that write the cache file.

The current way has a smaller risk than the alternatives.

## Related Discussions

- https://github.com/eslint/eslint/issues/1098 - Show results for each file as eslint is running
- https://github.com/eslint/eslint/issues/3565 - Lint multiple files in parallel
- https://github.com/eslint/eslint/issues/10272 - Validate options passed to CLIEngine API
- https://github.com/eslint/eslint/issues/10606 - Make CLIEngine and supporting functions async via Promises
- https://github.com/eslint/eslint/issues/12319 - `ERR_REQUIRE_ESM` when requiring `.eslintrc.js`
- https://github.com/eslint/rfcs/pull/4 - New: added draft for async/parallel proposal
- https://github.com/eslint/rfcs/pull/11 - New: Lint files in parallel
- https://github.com/eslint/rfcs/pull/42 - New: Lint files in parallel if many files exist
- https://github.com/eslint/rfcs/pull/44 - New: Drop supports for Node.js 8.x and 11.x
- https://github.com/eslint/rfcs/pull/45 - New: Formatter v2

[abortcontroller]: https://dom.spec.whatwg.org/#interface-abortcontroller
[abortsignal]: https://dom.spec.whatwg.org/#interface-abortsignal
[rfc04]: https://github.com/eslint/rfcs/pull/4
[rfc11]: https://github.com/eslint/rfcs/pull/11
[rfc20]: https://github.com/eslint/rfcs/tree/master/designs/2019-additional-lint-targets
[rfc42]: https://github.com/eslint/rfcs/pull/42
[rfc47]: https://github.com/eslint/rfcs/pull/47
