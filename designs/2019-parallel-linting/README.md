- Start Date: 2019-01-31
- RFC PR: (leave this empty, to be filled in later)
- Authors: Ilya Volodin

# Parallel Linting in ESLint

## Summary

Linting files in parallel on a large code-base should provide significant boost to the performance of ESLint.

## Motivation

Currently, ESLint lints all of the files in the same process. While that works in most situations, with the large code-base, this is inefficient, because Node will only run in the single processing core, leaving the rest idle. This has been one of the more often requested features by the community for a long while. There are existing solutions outside of ESLint (https://github.com/pinterest/esprint, https://github.com/alansouzati/eslint-parallel) however, due to the limitation of the API, it's not possible to create a fully optimized solution outside of the core. Implementation of the partial solution of parallelized linting yielded almost 3x improvement in case of [esprint](https://github.com/pinterest/esprint), but any external wrapper around ESLint API will not be able to share cache and config cache efficiently, that can only be done in the core. Our internal solution should have even better performance characteristics.

## Detailed Design

### General design

This feature will use process pool (where pool limit will be number of available CPUs on the client's machine) to process individual files in separate processes. Split from a single process to the pool will happen in async version of `executeOnFiles(patterns)` everything beyond that function will still continue to be synchronous.

### Command line option

This feature will add a new cli switch (`-p`, `--parallel`). This switch will enable parallel linting. By default this switch is off. This switch is incompatible with some of the existing options:

* --stdin
* --stdin-filename
* --init
* -h, --help
* -v, --version
* --print-config

### Changes to the core

In order to enable parallel linting, some of the core functions will need to be changed to be async and some others will have to be duplicated to provide the same functionality in async form.

Following functions will have to be made permanently async:
* bin/eslint.js - `global`
* lib/cli.js - `execute(args, text)`

Following functions will have to have a async clone:
* lib/cli-engine.js - `executeOnFiles(patterns)`

## Documentation

As this is a long awaited features by the community, after implementation, we should release a blog post describing the new option and providing some basic benchmarks.

Documentation of the feature will be done in the normal format as a new CLI option.

## Drawbacks

This feature will create additional complexities. Since a few entry functions will have to be permanently converted to async versions, it will complicated debugging.

## Backwards Compatibility Analysis

This feature is fully backwards compatible.

## Alternatives

There are already a few versions of this implementation available by the community. Unfortunately, those versions are forced to use ESLint API, which doesn't provide enough information to optimize the run completely.

## Open Questions

* Since we are going to be converting some of the functions to async versions, should we try to change our IO operations to async versions as well?
* Enabling this flag to work with `--debug` flag will create a lot of complexity. Should we skip it for the initial implementation?
* Since files will be linted in parallel, should we change formatters to support streaming and output as the information becomes available?

## Help Needed

I should be able to implement this myself, the problem is finding time and energy to do so.

## Related Discussions

[Lint multiple files in parallel](https://github.com/eslint/eslint/issues/3565)
