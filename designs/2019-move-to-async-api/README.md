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

And the name of `CLIEngine`, our primary API, causes confusion to the community. People try `Linter` class at first, then they notice it doesn't work as expected. We have a lot of issues that say "please use `CLIEngine` instead."
The name of new class, `ESLint`, is our primary API clearly.

## Detailed Design

### Add new `ESLint` class

This RFC adds a new class `ESLint`. It has almost the same methods as `CLIEngine`, but the following methods return `Promise`.

- `executeOnFiles()`
- `executeOnText()`
- `getConfigForFile()`
- `getFormatter()`
- `isPathIgnored()`
- `outputFixes()`

The following methods are as-is because those don't touch both file system and module system.

- `addPlugin()`
- `getErrorResults()`
- `getRules()`
- `resolveFileGlobPatterns()`

### Deprecate `CLIEngine` class

This RFC soft-deprecates `CLIEngine` class.

Because it's tough to maintain two versions (sync and async) of implementation. The two are almost copy-pasted stuff, but hard to share the code. Therefore, this RFC deprecates the sync version to improve our code with the way which needs asynchronous behavior in the future. For example, `CLIEngine` cannot support parallel linting, plugins/configs as ES modules, etc...

### Out of scope

- Not change API for rules. This RFC doesn't change APIs that rule implementation uses. We may be able to support asynchronous stuff in rules in the future, but it's out of this RFC's scope.
- Not change internal logics. This RFC just adds the public interface that is asynchronous. It would be a wrapper of `CLIEngine` for now.

## Documentation

- The "[Node.js API](https://eslint.org/docs/developer-guide/nodejs-api)" page should describe the new public API and deprecation of `CLIEngine` class.

## Drawbacks

People that use `CLIEngine` have to update their application with the new API. It will need hard work.

## Backwards Compatibility Analysis

Deprecating `CLIEngine` is a drastic change. But people can continue to use `CLIEngine` as-is until we decide to remove it. The decision would not be near future.

We can do both adding a new class and the deprecation in a minor release.

## Alternatives

- Adding `engine.executeAsyncOnFiles()`-like methods and we maintain it along with the existing synchronous API. But as what I wrote in the "[Deprecate `CLIEngine` class](#deprecate-cliengine-class)" section, it would be tough.

## Related Discussions

- https://github.com/eslint/eslint/issues/3565 - Lint multiple files in parallel
- https://github.com/eslint/eslint/issues/12319 - `ERR_REQUIRE_ESM` when requiring `.eslintrc.js`
- https://github.com/eslint/rfcs/pull/4 - New: added draft for async/parallel proposal
- https://github.com/eslint/rfcs/pull/11 - New: Lint files in parallel
