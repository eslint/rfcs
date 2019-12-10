- Start Date: 2019-12-09
- RFC PR: https://github.com/eslint/rfcs/pull/52
- Authors: @manuth

# Public `ConfigArray` API

## Summary

Allow 3rd-party developers to determine which configuration-files are considered by `eslint`.

## Motivation

In some cases people might want to automatically trigger a new linting process as soon as the configuration of eslint has changed.
This is a key feature for language servers which provide warnings and fixes to IDEs.

Therefore the language-server needs to know which config-files are considered by `eslint`.

## Detailed Design

A method called `getConfigArrayForFile()` shall be added to the `CLIEngine` class.
By providing such a feature people can easily watch the correct files for changes as soon as the configuration changes.

***Example:***
```js
import fs = require("fs");

let engine = new CLIEngine({});
let file = "./App/src/index.ts";
let configArray = engine.getConfigArrayForFile(file);

for (let config of configArray) {
    if (config.filePath.length > 0) {
        fs.watchFile(
            conig.filePath,
            () =>
            {
                lint();
            });
    }
}

function lint() {
    engine.executeOnFiles([file]);
}

lint();
```

## Documentation

I think it might be a good idea to blog about this new feature as this is a key feature for writing language servers for IDEs.
That way the integration of `eslint` in all sorts of text-editors can be improved to its best.

## Drawbacks

The only downside I can think of is that on every change to the `ConfigArray` or `CascadingConfigArray` API either a new major version has to be released or a separate implementation of said classes has to be added for the public API.

## Backwards Compatibility Analysis

Adding this feature provides full backwards compatibility.

## Alternatives

Add a method which only returns filepaths and a value indicating whether said filepath is `root`.  
That way the drawbacks could be fixed but the developers don't have the benefit to further analyze the config-array.

A similar method already exists: The `CLIEngine#getConfigForFile`-method. Though this method does not indicate which files are considered by `eslint` which draws developers of language-servers unable to hot-reload `eslint` as soon as a config-file changes.

## Open Questions

  - Is it reasonable to return the whole config-array or should such a method only return the `filePath`s?
  - Does it make sense to also return the `root`-directory in order to let language-servers run `eslint` with `root` as its `cwd`?
  - Is there some more key information other than `filePath` which should be returned? (Such as `name`, for instance)

## Help Needed

I think it should be pretty easy to implement such a feature as the only thing that is required is a simple call to `CascadingConfigArrayFactory#getConfigArrayForFile` and, if required, some data extraction/manipulation.

## Frequently Asked Questions

<!-- - No FAQs added yet --> 

## Related Discussions

 - https://github.com/eslint/eslint/issues/12631
