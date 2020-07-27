- Start Date: 2020-07-15
- RFC PR: https://github.com/eslint/rfcs/pull/63
- Authors: @c-home

# Add option to allow use of file contents for cache

## Summary

This RFC proposes improving cache for CI by providing an option to use a hash (instead of `mtime` and `fsize`) comparison to determine whether a file has changed. 

## Motivation

Motivation for this change is the same as [#11490](https://github.com/eslint/eslint/issues/11490), where details of files like `mtime` are not preserved in the CI environment. With a hash comparison, the cache can be used in CI and would significantly reduce the time ESLint takes to run.

ESLint uses [`file-entry-create`](https://github.com/royriojas/file-entry-cache) as its cache provider. [fileEntryCache#create](https://github.com/royriojas/file-entry-cache#createcachename-directory-usechecksum) gives the option to use a md5 hash through the `useChecksum` argument, but ESLint currently does not utilize it.

## Detailed Design

This RFC adds a `--cache-strategy` CLI [option](https://github.com/eslint/eslint/blob/e71e2980cd2e319afc70d8c859c7ffd59cf4157b/lib/options.js#L198). Users can specify the option to be:
- `contents`, for the use of an md5 hash
- `metadata`, for the use of `mtime` and `fsize`

Modified time and size (`metadata`), does not need to be specified as it will remain the default.

The implementation will be similar to [#11487](https://github.com/eslint/eslint/pull/11487), with differences in the naming of the options. The options were kept general so if the underlying implementation of the comparison method were to change, the usage and documentation can remain the same.

The majority of changes will be made to [`LintResultCache`](https://github.com/eslint/eslint/blob/e71e2980cd2e319afc70d8c859c7ffd59cf4157b/lib/cli-engine/lint-result-cache.js#L47). The `cache-strategy` will be added to the constructor of `LintResultCache`. If `cache-strategy` is set as `contents`, an md5 hash will be used in [fileEntryCache#create](https://github.com/royriojas/file-entry-cache#createcachename-directory-usechecksum).

## Documentation

Documentation for ESLint CLI will be updated with a description of the option.

## Drawbacks

- Specifying the details of the cache behaviour through a public option can make it more difficult to change the design and implementation [without breaking its usage](https://github.com/eslint/eslint/issues/11490#issuecomment-471400143).
- Comparing files based on their contents is slower than comparing files by their file metadata. However, a file contents comparison is not the default.
- Like any new feature, this flag will slightly increase the complexity and maintenance costs of ESLint.

## Backwards Compatibility Analysis

This change is backwards-compatible. It adds a new CLI option while keeping the behavior the same if the option is not specified.

## Alternatives

Mentioned as an alternative in [#11487](https://github.com/eslint/eslint/pull/11487), and if `file-entry-cache` [#14](https://github.com/royriojas/file-entry-cache/pull/14) were to be merged, an environment variable could be passed through `eslint` to `file-entry-cache`.

## Open Questions

None

## Frequently Asked Questions

None yet

## Related Discussions

- https://github.com/eslint/eslint/issues/11319 
- https://github.com/eslint/eslint/issues/11490 
- https://github.com/eslint/eslint/pull/11487
- https://github.com/royriojas/file-entry-cache/pull/14
