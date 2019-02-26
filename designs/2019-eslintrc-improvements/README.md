- Start Date: 2019-02-25
- RFC PR: (leave this empty, to be filled in later)
- Authors: (the names of everyone contributing to this RFC)

# Config File Improvements

## Summary

This proposal improves the internal logic of configuration files to maintain our codebase easier and add enhancements easier. This is a change of architecture and large refactoring.

This RFC fixes two bugs I found while I make a PoC (I guess we don't want to make surprised behaviors on purpose).

- A fatal error by unused dependencies (E.g. [eslint/eslint#11396]).
- A surprised behavior of `overrides` (see [details](#️-fix-a-surprised-behavior-of-overrides)).

To make sure if this refactoring is effective to make easier to enhance, the PoC of this RFC has added enhancements. I don't think this RFC requires those enhancements, but we can add.

- ESLint checks the files which are matched by `overrides[].files` property automatically (see [details](#3-it-changes-the-processing-order-to-config-then-files-from-files-then-config)).
- `overrides` supports `extends` and nested `overrides` (the logic is just recursive) ([eslint/eslint#8813]).

And, sorry, this RFC contains the change of the plugin resolution logic because I had misunderstood it has been accepted in [#7].

- ESLint looks plugins up from the location where the config file is. This is a similar thing to [#14].

## Motivation

- The codebase about configuration files is complicated.
- The current process determines target files at first. Next, it finds configuration files on the directory where each target file exists and that ancestors. This "files-then-config" order prevents adding some enhancements such as `--ext` functionality to our configuration file system.
- Several good ideas have been born in [#9]. We can simplify the logic about configuration files by the ideas.

## Detailed Design

Proof of Concept (implementation): https://github.com/eslint/eslint/tree/proof-of-concept/config-array-in-eslintrc

1. It simplifies the internal structure of configuration files by an array.
1. The config object owns loaded plugins and parsers rather than registers those to somewhere.
1. It changes the processing order to "config-then-files" from "files-then-config".
1. It restructures the files about CLIEngine and lookup.

And some side-effects are:

- `overrides` supports `extends` and nested `overrides`.
- A surprised behavior about `overrides` is fixed.
- Unused parser settings no longer throw ([eslint/eslint#11396]).

### 1. It simplifies the internal structure of configuration files by an array.

When it reads configuration files, it flattens the `extends` property and `overrides` property of the configuration. This is the following order:

1. The loaded configurations of `extends` property.
1. The configuration except `extends` property and `overrides` property.
1. The configurations of `overrides` property.

If it cannot load a configuration of `extends` property, it throws an error immediately.

If a loaded configuration of `extends` property has `extends` property or `overrides` property, it flattens those recursively.

If a configuration of `overrides` property has `extends` property or `overrides` property, it flattens those recursively. The `files` property and `excludedFiles` property of the configuration are applied to every flattened item. If a flattened item has own `files` property and `excludedFiles` property, it composes those by logical AND.

For duplicated settings, a later element in the array has precedence over an earlier element in the array.

<details><summary>Example:</summary>

```jsonc
{
    "extends": ["eslint:recommended", "plugin:node/recommended"],
    "rules": { ... },
    "overrides": [
        {
            "files": ["*.ts"],
            "extends": ["plugin:@typescript-eslint/recommended"],
            "rules": { ... },
        }
    ]
}
```

is flattend to:

```jsonc
[
    // extends
    {
        "name": "eslint:recommended",
        "filePath": null,
        "rules": { ... }
    },
    {
        "name": "plugin:node/recommended",
        "filePath": "node_modules/eslint-plugin-node/lib/index.js",
        "env": { ... },
        "parserOptions": { ... },
        "plugins": { ... },
        "rules": { ... }
    },

    // main
    {
        "name": ".eslintrc.json",
        "filePath": ".eslintrc.json",
        "rules": { ... }
    },

    // overrides (because it flattens recursively, extends in overrides is here)
    {
        "name": "plugin:@typescript-eslint/recommended",
        "filePath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js",
        // `matchFile` is merged from the parent `overrides` entry and itself.
        "matchFile": { "includes": ["*.ts"], "excludes": null },
        "parser": { ... },
        "parserOptions": { ... },
        "plugins": { ... },
        "rules": { ... },
    },
    {
        "name": ".eslintrc.json#overrides[0]",
        "filePath": ".eslintrc.json",
        "matchFile": { "includes": ["*.ts"], "excludes": null }
    }
]
```

</details>

### 2. The config object owns loaded plugins and parsers rather than registers those to somewhere.

The loading logic of configuration files is complicated because it has complicated relationships between the config, plugins, parsers, and environments.

![Current relationship graph](diagrams/current-deps.svg)

The main reason is registration. The loading logic has side-effects that register loaded plugins to the plugin manager, and plugins have side-effects that register rules and environments to other managers.

The codebase can get simple by the removal of the registration. Instead, the internal structure of configuration owns loaded plugins and parsers.

![New relationship graph](diagrams/new-deps.svg)

Surprisingly, now the return value of `ConfigArrayFactory.loadFile(filePath)` has all needed information to check files. Previously, we also needed information that was registered somewhere.

<details><summary>Example:</summary>

Note this is an internal structure. This proposal doesn't change config file's format.

```jsonc
{
    "name": "plugin:@typescript-eslint/recommended",
    "filePath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js",
    "parser": {
        "definition": { ... }, // the parser implementation.
        "id": "@typescript-eslint/parser",
        "filePath": "node_modules/@typescript-eslint/parser/dist/index.js",
        "importerPath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js"
    },
    "parserOptions": {
        "sourceType": "module"
    },
    "plugins": {
        "@typescript-eslint": {
            "definition": { ... }, // the plugin implementation.
            "id": "@typescript-eslint",
            "filePath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js",
            "importerPath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js"
        }
    },
},
```

</details>

If arbitrary errors happen while loading a plugin or a parser, the config array stores the error information rather than throws it. Because the plugin or the parser might not be used finally.

When `ConfigArray#extractConfig(filePath)` method extracted configuration for a file, if the final configuration contains the error, it throws the error. Here, "extract" means merge the elements in the config array as filtering it by `files` property and `excludedFiles` property.

> Note: once we implemented [#3], one file can be parsed as multiple virtual files. We can implement [#3] without file access because `ConfigArray` has complete information.

<details><summary>Example (error):</summary>

Note this is an internal structure. This proposal doesn't change config file's format.

```jsonc
{
    "name": "plugin:@typescript-eslint/recommended",
    "filePath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js",
    "parser": {
        "error": Error, // an error object (maybe "Module Not Found").
        "id": "@typescript-eslint/parser",
        "importerPath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js"
    },
    "parserOptions": {
        "sourceType": "module"
    },
    "plugins": {
        "@typescript-eslint": {
            "definition": { ... }, // the plugin implementation.
            "id": "@typescript-eslint",
            "filePath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js",
            "importerPath": "node_modules/@typescript-eslint/eslint-plugin/dist/index.js"
        }
    },
},
```

</details>

### 3. It changes the processing order to "config-then-files" from "files-then-config".

Currently, first it finds target files by globs, next it finds configs for each target file. Therefore, we could not change target files by configuration files because of this ordering.

This proposal changes that ordering.

1. When the file enumerator entered into a directory,
    1. It finds `.eslintrc.*` file on the directory.
        - If found, it concatenates the found configuration to the parent configuration (just `Array#concat`).
    1. It enumerates files on the directory.
        - If a file is a regular file and matched the current criteria, yields the pair of the file and the current configuration.
        - If a file is a directory, enters into the directory (go step 1).

Therefore, the file enumerator reuses configuration instances naturally without a special cache logic.

We can change target files by settings of configuration files. In this proposal, if any of `overrides` matches a file, the file enumerator yields the file additionally. For example, if `overrides[].files` includes `"*.ts"`, the file enumerator yields `*.ts` files additionally.

### 4. It restructures the files about CLIEngine and lookup.

It's premature optimization if we moved a logic only one functionality is using to the shared utility directory. The shared utility directory is similar to global variables, so it makes hard to know who use the utilities.

This proposal moves some utility files to the directory of lookup functionality.

We should be able to understand the lookup logic only the files in [lib/lookup](https://github.com/eslint/eslint/tree/proof-of-concept/config-array-in-eslintrc/lib/_lookup).

## Documentation

There are no changes because this is refactoring.

## Drawbacks

This is a large change. Currently we have multiple pull requests around configuration files, this will conflict with those pull requests severely. And this will be a large change for tests. Our tests really depend on internal structures seriously.

## Backwards Compatibility Analysis

Most cases work fine as is.
But I guess there are some breaking changes.

### ✅ CLIEngine#addPlugin

It can work fine as is.
`CLIEngine` has a `Map<string, Plugin>` for the added plugin, and it gives `FileEnumerator` (then `ConfigArrayFactory.load*` methods) the map, and the loading logic uses the map to load plugins.

### ✅ Linter

It can work fine as is.

Only if a given `config` has `extractConfig` method, `Linter` does the setup process for the config. Otherwise, it works the same as currently.

### ✅ LintResultCache

It can work fine as is.

The `ConfigArray` has all information to distinguish if the config was changed since the last time.

### ⚠️ `overrides` affects target files.

For example, `files: ["*.ts"]` in configuration makes ESLint checking TypeScript files as well.
This is a breaking change, but I think reasonable. If we don't want this change, we can remove this feature from this RFC.

### ⚠️ Fix a surprised behavior of `overrides`.

Currently, `overrides` is applied after all `overrides` properties are merged. This means that the `overrides` in a shareable config priors to the setting in your `.eslintrc`.

```yml
extends:
    - foo
rules:
    eqeqeq: error # silently ignored if the "foo" has the "eqeqeq" setting in "overrides".
```

After this proposal, the setting in your `.eslintrc` priors to the setting in shareable configs always.
This is a breaking change, but I think reasonable.

### ⚠️ It looks plugins up from the location where the config file is.

This means we get the consistent way to load dependencies on `extends`, `parser`, and `plugins`.

Currently, ESLint looks shareable configs and parsers relatively from the location where the config file is. But ESLint looks plugins from the project root. This is forcing inconvenient to shareable config maintainers ([eslint/eslint#3458], [eslint/eslint#10643]).

This proposal makes those behaviors consistent.

Each element of `ConfigArray` has the loaded plugins. When ESLint merges those by `ConfigArray#extractConfig(filePath)`, if one same plugin loaded from different config files (except self-loading), it throws a confliction error because of https://gist.github.com/not-an-aardvark/169bede8072c31a500e018ed7d6a8915.

This behavior is described on [#14] as well. This change has huge impact, probably we should discuss on [#14] individually.

People can resolve this problem with `.eslintrc.js` and to install needed plugins:

```js
const foo = require("eslint-config-foo")
const bar = require("eslint-config-bar")

module.exports = {
    overrides: [
        Object.assign({ files: "*.js" }, foo),
        Object.assign({ files: "*.js" }, bar),
        {
            files: "*.js",
            rules: {
                "a-problematic-rule": "off",
                /* some overrides for this project */
            }
        }
    ]
}
```

Then two shareable configs which depend on a same plugin can work together with a choosen version of the plugin.

## Alternatives

- [#9] ... Double duplicate features cause confusion for the ecosystem. For newcomers, a mix of articles about two config systems makes hard to understand ESLint. For non-English users, the official document is far.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

- [#14]
- [#9]
- [#7]
- [#5]
- [eslint/eslint#3458]
- [eslint/eslint#6732]
- [eslint/eslint#8813]
- [eslint/eslint#9897]
- [eslint/eslint#10125]
- [eslint/eslint#10643]
- [eslint/eslint#10891]
- [eslint/eslint#11223]
- [eslint/eslint#11396]

Especially, this proposal is inspired by the discussion on [#9].

[#14]: https://github.com/eslint/rfcs/pull/14
[#9]: https://github.com/eslint/rfcs/pull/9
[#7]: https://github.com/eslint/rfcs/pull/7
[#5]: https://github.com/eslint/rfcs/pull/5
[#3]: https://github.com/eslint/rfcs/pull/3
[eslint/eslint#3458]: https://github.com/eslint/eslint/issues/3458
[eslint/eslint#6732]: https://github.com/eslint/eslint/issues/6732
[eslint/eslint#8813]: https://github.com/eslint/eslint/issues/8813
[eslint/eslint#9897]: https://github.com/eslint/eslint/issues/9897
[eslint/eslint#10125]: https://github.com/eslint/eslint/issues/10125
[eslint/eslint#10643]: https://github.com/eslint/eslint/issues/10643
[eslint/eslint#10891]: https://github.com/eslint/eslint/issues/10891
[eslint/eslint#11223]: https://github.com/eslint/eslint/issues/11223
[eslint/eslint#11396]: https://github.com/eslint/eslint/issues/11396
