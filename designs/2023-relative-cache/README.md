 Repo: eslint/eslint
- Start Date: 2023-02-14
- RFC PR: <https://github.com/eslint/rfcs/pull/106>
- Authors: Christian Schulz (@cschulz) and cs6cs6

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

I have a branch with the following design already. It is not fully tested, so I'm not 100% sure this will be the final implementation but I think it is a good start. 

The LintResultCache takes a file path as parameter to find or store a file cache entry.

The suggested approach is to truncate the given absolute path of the file to a relative path in the cache.

### Adding the command line parameter
- conf/default-cli-options.js: add the property 'shareableCache' with a default of false. It should be put in the section with the other cache variables, below cacheStrategy. 
- docs/src/use/command-line-interface.md: Add an explanation of the 'shareable-cache' variable with the other cache variables: "By default, an eslintcache contains full file paths and thus cannot readily be shared across developers or ci machines. "False" is that default. "True" changes the internal storage to store a relative path to the process execution directory, thus making the eslintcache shareable across developers and ci machines. . If you change this setting, you must regenerate your entire eslint cache."
- eslint-helpers.js: add shareableCache variable (set to false) to the processOptions
- lib/options.js: Add an option 'shareable-cache' of type Boolean with a nice description for people to read: By default, an eslintcache contains full file paths and thus cannot readily be shared across developers or ci machines. "False" is that default. "True" changes the internal storage to store a relative path to the process execution directory, thus making the eslintcache shareable across developers and ci machines. . If you change this setting, you must regenerate your entire eslint cache.


### Changing cache file serialization
- lib/eslint/flat-eslint.js: Define the shearableCache property at the top, with the other properties, as a boolean
- lib/eslint/flat-eslint.js: Around line 873, where the flat-eslint.js file is saving the cache in the cachefile, if processOptions.shareableCache is set to true, pass a PARTIAL path to the lintResultCache.setCachedLintResults() function instead of the full path. 

## Documentation

Add the new CLI flag to the CLI documentation. The new cli flag will be called shareable-cache.

## Drawbacks

Adding a new CLI flag adds some complexity for the end user. They will need to understand why the feature exists. 

Furthermore, defaulting to the existing behavior does help make upgrading less painful, but it is possible most people expect a shareable cache by default. This may cause confusion.

## Backwards Compatibility Analysis

The new flag, shareable-cache, passed in on the command line, will default to false. Unless people explicitly set it to true, then the old algorithms will
be used. They will not need to regenerate their cache unless they set this flag. 

## Alternatives

A CLI tool which translates the existing cache to the current folder structure.

## Open Questions

No.

## Help Needed

I have a branch with code changes, but I am trying to figure out how to build an artifact to test with. Using the eslint.js artifact in the browser does not make sense
with a cache change, and I would like to test it in my project. How can I build a package that I can install with NPM into my project to test?? I am able to run the tests
on the eslintproject and they all pass. My fork is here: https://github.com/cs6cs6/eslint_patch/tree/fork/shareable-eslintcache

## Frequently Asked Questions

TBD

## Related Discussions

[Change Request: Eslintcache relative #16493](https://github.com/eslint/eslint/issues/16493)
