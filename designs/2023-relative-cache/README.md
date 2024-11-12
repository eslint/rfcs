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

### Example existing cache file

Here is an example existing cache file for an extremely small project with two source files. It has been formatted for readability. 

You can see that the full path of the file is included in the filenames, '/home/USER/git/samplecode'. There is also an obscure field, the value
1k0siqs. That is the hash-value of a stringified ConfigArray object that contains all the configuration for the entire eslint run. It is there so
that any rule changes or configuration changes will cause a complete eslint recheck on all files. 

```
[
  {
    "/home/USER/git/samplecode/src/formatter.ts": "1",
    "/home/USER/git/samplecode/src/vite-env.d.ts": "2"
  },
  {
    "size": 2584,
    "mtime": 1695823503788,
    "results": "3",
    "hashOfConfig": "4"
  },
  {
    "size": 75,
    "mtime": 1695814197580,
    "results": "5",
    "hashOfConfig": "4"
  },
  {
    "filePath": "6",
    "messages": "7",
    "suppressedMessages": "8",
    "errorCount": 0,
    "fatalErrorCount": 0,
    "warningCount": 0,
    "fixableErrorCount": 0,
    "fixableWarningCount": 0
  },
  "1k0siqs",
  {
    "filePath": "9",
    "messages": "10",
    "suppressedMessages": "11",
    "errorCount": 0,
    "fatalErrorCount": 0,
    "warningCount": 0,
    "fixableErrorCount": 0,
    "fixableWarningCount": 0
  },
  "/home/USER/git/samplecode/src/formatter.ts",
  [],
  [],
  "/home/USER/git/samplecode/src/vite-env.d.ts",
  [],
  []
]

```

### Suggested updated cache file

Here is a suggested updated cache file, with relative paths instead of full paths. The `.` is the current working  
directory. Typically, eslint is always run from the same place, so using a relative path should not cause a problem.

Note that the obscure hash value is still there, and is now Ak0sovs. This hash value will need to be calculated using
relative paths only. 

``` 
[
  {
    "./src/formatter.ts": "1",
    "./src/vite-env.d.ts": "2"
  },
  {
    "size": 2584,
    "mtime": 1695823503788,
    "results": "3",
    "hashOfConfig": "4"
  },
  {
    "size": 75,
    "mtime": 1695814197580,
    "results": "5",
    "hashOfConfig": "4"
  },
  {
    "filePath": "6",
    "messages": "7",
    "suppressedMessages": "8",
    "errorCount": 0,
    "fatalErrorCount": 0,
    "warningCount": 0,
    "fixableErrorCount": 0,
    "fixableWarningCount": 0
  },
  "Ak0sovs",
  {
    "filePath": "9",
    "messages": "10",
    "suppressedMessages": "11",
    "errorCount": 0,
    "fatalErrorCount": 0,
    "warningCount": 0,
    "fixableErrorCount": 0,
    "fixableWarningCount": 0
  },
  "./src/formatter.ts",
  [],
  [],
  "./src/vite-env.d.ts",
  [],
  []
]
```

### Adding the command line parameter
- `conf/default-cli-options.js`: add the property `shareableCache` with a default of false. It should be put in the section with the other cache variables, below `cacheStrategy`. 
- `docs/src/use/command-line-interface.md`: Add an explanation of the `shareable-cache` variable with the other cache variables: "By default, an eslintcache contains full file paths and thus cannot readily be shared across developers or ci machines. "False" is that default. "True" changes the internal storage to store a relative path to the process execution directory, thus making the eslintcache shareable across developers and ci machines. . If you change this setting, you must regenerate your entire eslint cache."
- `eslint-helpers.js`: add `shareableCache` variable (set to false) to the `processOptions`, and make sure it must be a boolean. 
- `lib/options.js`: Add an option `shareable-cache` of type Boolean with a nice description for people to read: By default, an eslintcache contains full file paths and thus cannot readily be shared across developers or ci machines. "False" is that default. "True" changes the internal storage to store a relative path to the process execution directory, thus making the eslintcache shareable across developers and ci machines. . If you change this setting, you must regenerate your entire eslint cache.
- `lib/cli.js`: Add the `shareableCache` option, defaulted to false, in the `translateOptions `function. 

### Changing cache file serialization
- `lib/cli-engine/lint-result-cache.js`: Add the properties `cwd` and `shareableCache` to the `LintResultCache` class. `cwd` is a string, the current working directory. `shareableCache` is a boolean, the result of the user's passed in command line parameter. Use these values in two main places: 
    1. Whenever we store or retrieve file results in the cache, use the results of the new class function, `getFileKey(filePath)`. If `shareableCache` is false, it uses a full file path as the key. If `shareableCache` is true, uses a relative file path. 
    2. Update the function `hashOfConfigFor`, so that when the configuration object is hashed into a hash key and put into the eslintcache, if `shareableCache` is set to true, the file paths hashed are all relative paths. Implementation detail:
```js
function hashOfConfigFor(config, cwd) {
    if (!configHashCache.has(config)) {
        // replace the full directory path in the config string to make the hash location-neutral
        // implementation TBD; please note configHashCache is a complex object with over 100 fields.        
    }
``` 
- lib/cli-engine/cli-engine.js:  Update the constructor call to LintResultCache to add the new parameters cwd and shareableCache. 

## Documentation

Add the new CLI flag to the CLI documentation. The new cli flag will be called shareable-cache.

## Drawbacks

Adding a new CLI flag adds some complexity for the end user. They will need to understand why the feature exists. 

Furthermore, defaulting to the existing behavior does help make upgrading less painful, but it is possible most people expect a shareable cache by default. This may cause confusion.

## Backwards Compatibility Analysis

The new flag, shareable-cache, passed in on the command line, will default to false. Unless people explicitly set it to true, then the old algorithms will
be used. Users of the eslint cache will still need to regenerate their cache no matter how they set this flag, because the current implementation forces a cache rebuild if there are
any configuration changes. Adding a new property counts as a change. 

## Alternatives

A CLI tool which translates the existing cache to the current folder structure. This could be any sort of script that will basically do a replace-all on the paths in 
the .eslintcache file and also recalculates the hash so that they are the new location rather than the old. A downside to this is that any tool based on string 
replacement would be fragile at best. 

## Open Questions

Question: Should we make every cache shareable, or keep the shreable-cache flag default to false?

Answer: We don't know how people are currently using the cache file, so we can't make assumptions about what is and isn't safe to change. Even if we made a shareable cache the default (which would be a breaking change), we'd still need a way to enable the previous behavior in case there is a use case we aren't aware of..

## Help Needed

None at this time.

## Frequently Asked Questions

TBD

## Related Discussions

[Change Request: Eslintcache relative #16493](https://github.com/eslint/eslint/issues/16493)
