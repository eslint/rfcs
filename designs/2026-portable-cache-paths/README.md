- Repo: eslint/eslint
- Start Date: 2026-10-09
- RFC PR: https://github.com/eslint/rfcs/pull/155
- Authors: @Naismith

## Summary

ESLint should be able to store file paths in the cache file relative to the cache file location, instead of as absolute paths. This should be opt-in through a new `--cache-paths` CLI option and a matching `cachePaths` option in the `ESLint` constructor. Both accept `"absolute"` (the default, which keeps today's behavior) or `"relative"`.

For example, if `.eslintcache` is in the project root, the entry for `/home/alice/repo/src/a.js` should be stored as `src/a.js`. A copy of that cache file in `/home/ci/builds/42/repo` or in a git worktree at `/home/alice/repo-feature` should then give cache hits for `src/a.js`.

## Motivation

Each entry in the cache file uses the absolute path of the linted file as its key. If the same project is linted from a different path, no key matches and ESLint lints every file again.

This prevents reusing the cache across git worktrees, and across CI runs that check out code into a different folder each time (eslint/eslint#16493, eslint/eslint#16741).

## Detailed Design

### High-Level Behavior

When `cachePaths` is `"relative"`, ESLint should store each cache key as a **relative key**: the path from the **cache folder** (the folder that contains the cache file) to the linted file, with `/` as the separator. For example, `src/a.js` or `../packages/app/src/a.js`.

With `"absolute"`, ESLint behaves exactly as it does today.

### CLI Behavior

A new `--cache-paths` option will be added. It accepts `absolute` or `relative`, defaults to `absolute`, and requires `--cache`, the same as `--cache-strategy`:

```bash
eslint --cache --cache-strategy content --cache-paths relative .
```

### Node.js API Behavior

A new `cachePaths` option will be added to the `ESLint` constructor. It accepts `"absolute"` or `"relative"` and defaults to `"absolute"`:

```js
const eslint = new ESLint({
    cache: true,
    cacheStrategy: "content",
    cachePaths: "relative",
});
```

### Restoring `filePath`

Each cached lint result also stores an absolute `filePath`, which would point to the wrong location on another machine. When `cachePaths` is `"relative"`, `setCachedLintResults()` will store the relative key in `filePath`, and `getCachedLintResults()` will set it back to the current absolute path.

### Cache File Name When `--cache-location` Is a Folder

When `--cache-location` points at a folder, `getCacheFile()` names the cache file using a hash of the absolute `cwd`, so the name differs in every location. When `cachePaths` is `"relative"`, ESLint will hash `path.relative(cacheFolder, cwd)` with `/` separators instead.

### Implementation Approach

The implementation would likely touch the following files:

- `lib/options.js`: add the `--cache-paths` String option with `enum: ["absolute", "relative"]`, `default: "absolute"`, and `dependsOn: ["cache"]`, following `--cache-strategy`.
- `lib/cli.js`: map `cachePaths` in `translateOptions()`.
- `lib/eslint/eslint-helpers.js`: validate `cachePaths` in `processOptions()` the same way as `cacheStrategy`, pass it from `createLintResultCache()` to `LintResultCache`, and update `getCacheFile()` as described above.
- `lib/eslint/eslint.js` and `lib/eslint/worker.js`: pass the option when creating a `LintResultCache`, so multithread linting behaves the same as single-thread linting.
- `lib/cli-engine/lint-result-cache.js`: accept and validate the option in the constructor, pass `cwd: cacheFolder` to `fileEntryCache.create()` when it is `"relative"`, convert absolute paths to relative keys before calling `getFileDescriptor()`, and restore `filePath` on read. Reading the file source from disk still uses the absolute path.
- `lib/types/index.d.ts`: add `cachePaths?: "absolute" | "relative"` to `ESLint.Options`.

ESLint already depends on `file-entry-cache` v11, which supports a `cwd` option for relative keys. No dependency change is needed. The core of the change in `LintResultCache`:

```js
const validCachePaths = ["absolute", "relative"];

constructor(cacheFileLocation, cacheStrategy, cachePaths = "absolute") {
    assert(
        validCachePaths.includes(cachePaths),
        'Cache paths must be one of: "absolute", "relative"',
    );

    const cacheFolder = path.dirname(cacheFileLocation);
    const useRelativePaths = cachePaths === "relative";

    this.fileEntryCache = fileEntryCache.create(
        path.basename(cacheFileLocation),
        cacheFolder,
        {
            useModifiedTime: !useCheckSum,
            useCheckSum,
            ...(useRelativePaths ? { cwd: cacheFolder } : {}),
        },
    );
    this.useRelativePaths = useRelativePaths;
    this.cacheFolder = cacheFolder;
}

#toKey(filePath) {
    return this.useRelativePaths
        ? path.relative(this.cacheFolder, filePath).replace(/\\/gu, "/")
        : filePath;
}
```

### Example

Suppose a project has this layout:

```
repo/
├── .eslintcache
└── src/
    └── a.js
```

With `--cache-paths absolute`, the key for `src/a.js` is `/home/ci/builds/42/repo/src/a.js`. With `--cache-paths relative`, it is `src/a.js`, which matches in any copy of the project.

## Documentation

This RFC requires updates to:

- [CLI options documentation](https://eslint.org/docs/latest/use/command-line-interface#caching): add `--cache-paths` with its values and default.
- [Node.js API reference](https://eslint.org/docs/latest/integrate/nodejs-api): add the `cachePaths` constructor option.

## Drawbacks

This adds one more cache option to document, test, and maintain.

## Backwards Compatibility Analysis

This change is fully backwards compatible. The default value is `"absolute"`, which keeps today's behavior. Changing the value causes a one-time full lint, because existing keys no longer match.

## Alternatives

### Store Paths Relative to `cwd`

Keys could be relative to `cwd` instead of the cache folder. This was rejected because `cwd` changes when developers run ESLint from different subfolders of a monorepo, so the same file would get different keys. When `--cache-location` is not set, the cache file is in `cwd`, so the cache folder gives the same result in the common case.

### Make Relative Paths the Default

This would be simpler for users, but it changes cache behavior for everyone and would be a breaking change. Because `--cache-paths` takes a value, a future major version can change the default to `"relative"` without adding or removing an option.

The long-term goal is for `"relative"` to become the default in a future major version. Relative paths have no known downside for single-machine use, and a portable cache is what most users expect. Shipping the option as opt-in first lets the team collect feedback and fix edge cases before the default changes.

## Open Questions

None.

## Help Needed

I can implement this RFC.

## Frequently Asked Questions

None.

## Related Discussions

- eslint/eslint#16493
- eslint/eslint#16741
- eslint/rfcs#106
- eslint/rfcs#114
- jaredwray/cacheable#914
