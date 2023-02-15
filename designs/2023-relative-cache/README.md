- Repo: eslint/eslint
- Start Date: 2023-02-14
- RFC PR: <https://github.com/eslint/rfcs/pull/106>
- Authors: Christian Schulz (@cschulz)

# Relative cache location strategy

## Summary

To be able to share the created eslint cache between different systems f.e. local machine and CI, there is a need for a relative path stored in the cache.

## Motivation

The overall motivation for doing this is performance during the eslint process.

Most people are running eslint on local developer machine and additional on the CI workers.
In high changing repositories there are several pull requests at the same time changing different files requiring linting all the time.
This is a time consuming process depending on the enabled rules like type checking rules.

If the CI worker is able to cache its last run, it can just pick this cache up and use it.
Also developers can reuse this cache or commit it as part of a repository for the CI.

## Detailed Design

The LintResultCache takes a file path as parameter to find or store a file cache entry.

One approach would be to truncate the given absolute path of the file by the path of the cache location.

Another approach to get a relative path is to use the parent eslint config file if we are thinking about config hierarchies.

The configuration should be done by CLI flags like the cache strategy option itself.

## Documentation

Add the new CLI flag to the CLI documentation.

## Drawbacks

No.

## Backwards Compatibility Analysis

Existing caches needs to be depleted, because there is no cache version entry existing.

Another way would be to ignore existing absolute paths and write the new relative path if enabled.

## Alternatives

A CLI tool which translates the existing cache to the current folder structure.

## Open Questions

No.

## Help Needed

TBD

## Frequently Asked Questions

TBD

## Related Discussions

[Change Request: Eslintcache relative #16493](https://github.com/eslint/eslint/issues/16493)
