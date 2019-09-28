- Start Date: 2019-09-28
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Moving to Asynchronous API

## Summary

This RFC changes the methods of `CLIEngine` class returning `Promise` and adds `CLIEngineSync` class for soft-migration.

## Motivation

- Dynamic loading of ES modules requires asynchronous API. Migrating to asynchronous API opens up doors to write plugins/configs with ES modules.
- Linting in parallel requires asynchronous API. We can improve linting logic to run it in parallel. And we can improve config loading logic and file enumeration logic to run it in parallel. (E.g., load `extends` list, traverse child directories.)

## Detailed Design

### Change the methods of `CLIEngine` class

This RFC changes the following methods as returning `Promise`.

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

### Add `CLIEngineSync` class

This RFC adds a new public class `CLIEngineSync` for soft-migration. The `CLIEngineSync` class has the completely same interface as the current `CLIEngine`.

### Deprecate `CLIEngineSync` class

This RFC soft-deprecates the new public class `CLIEngineSync`.

Because it's tough to maintain two versions (sync and async) of implementation. The two are almost copy-pasted stuff, but hard to share the code. Therefore, this RFC deprecates the sync version to improve our code with the way which needs asynchronous behavior in the future. For example, `CLIEngineSync` cannot support parallel linting, plugins/configs as ES modules, etc...

### Out of scope

- Not change API for rules. This RFC doesn't change APIs that rule implementation uses. We may be able to support asynchronous stuff in rules in the future, but it's out of this RFC's scope.
- Not change internal logics. This RFC just changes the public interface. The internal logic is still synchronous.

## Documentation

- This change needs the entry in the migration guide because of a breaking change.
- The "[Node.js API](https://eslint.org/docs/developer-guide/nodejs-api)" page should describe the new public API.

## Drawbacks

People that use `CLIEngine` have to update their application with the new asynchronous API. This may be hard work.

## Backwards Compatibility Analysis

Yes, this is a drastic breaking change. New `CLIEngineSync` class is for soft-migration.

## Alternatives

- Adding `engine.executeAsyncOnFiles()`-like methods rather than changing existing methods.
- Adding new `CLIEngineAsync`-like class that has async API rather than we move the existing APIs to `CLIEngineSync`.

## Related Discussions

- https://github.com/eslint/eslint/issues/3565 - Lint multiple files in parallel
- https://github.com/eslint/eslint/issues/12319 - `ERR_REQUIRE_ESM` when requiring `.eslintrc.js`
- https://github.com/eslint/rfcs/pull/4 - New: added draft for async/parallel proposal
- https://github.com/eslint/rfcs/pull/11 - New: Lint files in parallel
