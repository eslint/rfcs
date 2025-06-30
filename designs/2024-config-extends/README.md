- Start Date: 2024-11-13
- RFC PR: https://github.com/eslint/rfcs/pull/126
- Authors: Nicholas C. Zakas (@nzakas)
- repo: eslint/eslint

# Flat Config Extends

## Summary

This proposal provides a way to implement `extends` in flat config objects. The `extends` key was an integral part of the eslintrc configuration format and was not included in flat config initially. 

## Motivation

One of the goals of flat config was to greatly simplify the creation of config files. We eliminated a lot of the extraneous features, grouped related keys together, and settled on a single, flat array with config objects as the overall format. The intent was to turn ESLint configuration into just an array of objects that could easily be manipulated in a variety of ways, limited only by what the JavaScript language enabled.

Now, several months after the general availability of flat config, some common complaints are emerging.

### Loss of standarized way to expose configs

First, it's difficult to know how a plugin exports configurations. A configuration can be an object or an array, and it may be accessible via a `configs` key or via a module specifier. Further, typescript-eslint actually [recommends](https://typescript-eslint.io/packages/typescript-eslint/#flat-config-extends) the use of their own utility to create configs, which [confuses users](https://github.com/typescript-eslint/typescript-eslint/issues/8496). That means you can have multiple ways of importing a config into your own. Here's an example:

```js
import js from "@eslint/js";
import tailwind from "eslint-plugin-tailwindcss";
import reactPlugin from "eslint-plugin-react";
import eslintPluginImportX from 'eslint-plugin-import-x';

export default [
    js.configs.recommended,
    ...tailwind.configs["flat/recommended"],
    ...reactPlugin.configs.flat.recommended,
    eslintPluginImportX.flatConfigs.recommended,
];
```

### Confusion over `...` syntax

As discussed in the previous section, configs may be objects or arrays, and the latter requires the use of the spread syntax (`...`) to include its contents into the overall array. We've found that JavaScript beginners don't know or understand spread syntax, making this distinction between objects and arrays even more confusing. 

### Difficult extending configs for specific file patterns

Further adding to the confusion over objects and arrays is how to take an existing config and apply it to a subset of files. This explicitly requires you to know if something is an object or array because how you extend it varies depending on the type. For objects, you do this:

```js
// eslint.config.js
import js from "@eslint/js";

export default [
    {
        ...js.configs.recommended,
        files: ["**/src/safe/*.js"]
    }
];
```

For arrays you do this:

```js
// eslint.config.js
import exampleConfigs from "eslint-config-example";

export default [
    ...exampleConfigs.map(config => ({
        ...config,
        files: ["**/src/safe/*.js"]
    })),

    // your modifications
    {
        rules: {
            "no-unused-vars": "warn"
        }
    }
];
```

In both cases, users need to use the spread syntax, and in both cases, just glancing at the config object doesn't really tell you what exactly is happening. It's a lot of syntax for what used to be a single statement in eslintrc.

## Goals

There are two goals with this design:

1. **Make it easier to configure ESLint.** For users, we want to reduce unnecessarily boilerplate and guesswork as much as possible.
1. **Encourage plugins to use `configs`.** For plugin authors, we want to encourage the use of a consistent entrypoint, `configs`, where users can find predefined configurations. 


## Detailed Design

Design Summary:

1. Introduce a `defineConfig()` function
1. Allow arrays in config arrays (in addition to objects)
1. Introduce an `extends` key in config objects

### Definitions

* **Base Object** - an object containing the `extends` key.
* **Extended Object** - an object that is a member of an `extends` array.
* **Generated Object** - an object that results from applying an extended object to a base object.

### Introduce a `defineConfig()` Function

Rather than changing either `FlatConfigArray` or `ConfigArray`, we'll create a `defineConfig()` function that will contain all of the functionality described in this RFC. Many other tools support a `defineConfig()` function, so this won't seem unusual to ESLint users. Examples include: [Rollup](https://rollupjs.org/command-line-interface/#config-intellisense), [Astro](https://docs.astro.build/en/guides/configuring-astro/#the-astro-config-file), [Vite](https://vite.dev/config/#config-intellisense), [Nuxt](https://nuxt.com/docs/getting-started/configuration). While those tools use `defineConfig()` primarily as a means to support editor Intellisense, we will also use it an abstraction layer between the base flat config behavior and the desired new behavior.

The `defineConfig()` function will be defined in a new `@eslint/config-helpers` package contained in the [rewrite](https://github.com/eslint/rewrite) repository. The `eslint` package will depend on `@eslint/config-helpers` and export `defineConfig()` directly, so users of ESLint versions that support `defineConfig()` natively will not have to install a separate package. For older versions of ESLint, they'll still be able to use `defineConfig()` by manually installing the `@eslint/config-helpers` package.

The `defineConfig()` function can accept both objects and arrays, and will flatten its arguments to create a final, flat array. As a rough sketch:

```js
export function defineConfig(...configs) {

    const finalConfigs = configs.map(processConfig);
    
    return finalConfigs.flat(Infinity);
}
```

The intent is to allow both of the following uses:

```js
// use case 1: array
import { defineConfig } from "eslint";
import js from "@eslint/js";

export default defineConfig([
    {
        ...js.configs.recommended,
        files: ["**/src/safe/*.js"]
    },
    {
        files: ["**/*.cjs"],
        languageOptions: {
            sourceType: "commonjs"
        }
    }
]);
```

```js
// use case 2: multiple arguments
import { defineConfig } from "eslint";
import js from "@eslint/js";

export default defineConfig(
    {
        ...js.configs.recommended,
        files: ["**/src/safe/*.js"]
    },
    {
        files: ["**/*.cjs"],
        languageOptions: {
            sourceType: "commonjs"
        }
    }
);
```

### Allow Arrays in Config Arrays

This was part of the [original flat config RFC](https://github.com/eslint/rfcs/tree/main/designs/2019-config-simplification#extending-another-config) but was disabled over concerns about the complexity involved with importing configs from packages and having no idea whether it was an object, an array, or an array with a mix of objects and arrays. This mattered most when users needed to use `map()` to apply different `files` or `ignores` to configs, but with `extends`, that concern goes away.

The flattening logic will live in `defineConfig()` rather than turning on flattening with `ConfigArray`. This allows users of older ESLint versions to also have access to this functionality. The first example in this RFC can be rewritten as:

```js
import { defineConfig } from "eslint";
import js from "@eslint/js";
import tailwind from "eslint-plugin-tailwindcss";
import reactPlugin from "eslint-plugin-react";
import eslintPluginImportX from 'eslint-plugin-import-x'

export default defineConfig([
    
    // object
    js.configs.recommended,
    
    // array
    tailwind.configs["flat/recommended"],
    
    // array
    reactPlugin.configs.flat.recommended,
    
    // object
    eslintPluginImportX.flatConfigs.recommended,
]);
```

### Introduce an `extends` Key

The `extends` key is an array that may contain other configurations. When used, ESLint will expand the config object into multiple config objects in order to create a flat structure to evaluate. 

#### Extending Objects

You can pass one or more objects in `extends`, like this:

```js
import { defineConfig } from "eslint";
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default defineConfig([

    {
        files: ["**/src/*.js"],
        extends: [js.configs.recommended, example.configs.recommended]
    }

]);
```

How these configs are merged depends on what `files` and `ignores` are present on both the base object and the extended objects.

##### Base Objects and Extended Objects without `files` or `ignores`

If the base object and the extended objects don't have any `files` or `ignores`, then the objects are placed directly into the config array with the extended objects first and the base object last. For example:

```js
import { defineConfig } from "eslint";

const config1 = {
    rules: {
        semi: "error";
    }
};

const config2 = {
    rules: {
        "prefer-const": "error"
    }
};

export default defineConfig([
    {
        extends: [config1, config2],
        rules: {
            "no-console": "error"
        }
    }
]);
```

This config would result in the following array:

```js
export default [
    {
        rules: {
            semi: "error";
        }
    },
    {
        rules: {
            "prefer-const": "error"
        }
    },
    {
        rules: {
            "no-console": "error"
        }
    }
];
```

##### Base Object with `files` and `ignores` and Extended Objects without `files` or `ignores`

When the base object has `files` and/or `ignores` and the extended objects do not, the extended objects are added with the same `files` and `ignores` as the base object. For example:

```js
import { defineConfig } from "eslint";

const config1 = {
    rules: {
        semi: "error";
    }
};

const config2 = {
    rules: {
        "prefer-const": "error"
    }
};

export default defineConfig([
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        extends: [config1, config2],
        rules: {
            "no-console": "error"
        }
    }
]);
```

This config would result in the following array:

```js
export default [
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        rules: {
            semi: "error";
        }
    },
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        rules: {
            "prefer-const": "error"
        }
    },
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        rules: {
            "no-console": "error"
        }
    }
];
```

##### Base Object with `files` and `ignores` and Extended Object with `files` or `ignores`

If the objects in `extends` contain `files`, then ESLint will intersect those values (AND operation); if the object in `extends` contains `ignores`, then ESLint will merge those values (OR operation). For example:

```js
import { defineConfig } from "eslint";

const config1 = {
    files: ["**/*.cjs.js"],
    rules: {
        semi: "error";
    }
};

const config2 = {
    files: ["**/*.js"],
    ignores: ["**/tests/*.js"],
    rules: {
        "prefer-const": "error"
    }
};

export default defineConfig([
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        extends: [config1, config2],
        rules: {
            "no-console": "error"
        }
    }
]);
```

This config would result in the following array:

```js
export default [
    {
        files: [["src/*.js", "**/*.cjs.js"]],
        ignores: ["src/__tests/**/*.js"],
        rules: {
            semi: "error";
        }
    },
    {
        files: [["src/*.js", "**/*.js"]],
        ignores: ["src/__tests/**/*.js", "**/tests/**/*.js"],
        rules: {
            "prefer-const": "error"
        }
    },
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        rules: {
            "no-console": "error"
        }
    }
];
```

When an extended config and the base config both have multiple `files` entries, then the result is a `files` entry containing all combinations. For example:

```js
import { defineConfig } from "eslint";

export default defineConfig([
    {
        files: ["src/**", "lib/**"],
        extends: [{ files: ["**/*.js", "**/*.mjs"] }]
    }
]);
```

This config would be calculated as the following:

```js
export default [
    {
        files: [
            ["src/**", "**/*.js"],
            ["lib/**", "**/*.js"],
            ["src/**", "**/*.mjs"],
            ["lib/**", "**/*.mjs"]
        ]
    }
];
```

Notes:

1. The `files` and `ignores` values from the base config always come first in the calculated `files` and `ignores`. If there's not a match, then there's no point in continuing on to use the patterns from the extended configs.
1. These behaviors differ from the [typescript-eslint extends helper](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/typescript-eslint/src/config-helper.ts). The `tseslint.config()` helper overwrites the `files` and `ignores` fields on any config contained in `extends` whereas this proposal creates combination `files` and `ignores`, merging the base config with the extended configs.

##### Extended Object with Only `ignores`

When a base config extends an object with only `ignores` (global ignores), the extended object `ignores` is placed in a new object with just `ignores`. For example:

```js
import { defineConfig } from "eslint";

const config1 = {
    ignores: ["build"]
};

const config2 = {
    ignores: ["dist"]
};

export default defineConfig([
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        extends: [config1, config2],
        rules: {
            "no-console": "error"
        }
    }
]);
```

This config would result in the following array:

```js
export default [
    {
        ignores: ["build"]
    },
    {
        ignores: ["dist"]
    },
    {
        files: ["src/*.js"],
        ignores: ["src/__tests/**/*.js"],
        rules: {
            "no-console": "error"
        }
    }
];
```

In short, global ignores are always inherited without any modification.

##### Calculating Config Names

To provide for easier debugging, each generated object will have a `name` key that is the concatenation of the base object name and the extended object name, separated by a `>`. For example:

```js
import { defineConfig } from "eslint";

const config1 = {
    name: "config1",
    rules: {
        semi: "error";
    }
};

const config2 = {
    name: "config2",
    rules: {
        "prefer-const": "error"
    }
};

export default defineConfig([
    {
        name: "base config",
        extends: [config1, config2],
        rules: {
            "no-console": "error"
        }
    }
]);
```

This config would result in the following array:

```js
export default [
    {
        name: "base config > config1",
        rules: {
            semi: "error";
        }
    },
    {
        name: "base config > config2",
        rules: {
            "prefer-const": "error"
        }
    },
    {
        name: "base config",
        rules: {
            "no-console": "error"
        }
    }
];
```

Notes:

1. If the base config doesn't have a `name` key, then it will be assigned a name in the format `"user config N"` where `N` is the index in the array. If the base object is, itself, contained in another array, then `N` will be the concatenation of the array indices from each parent array, such as `1,5,2` that would allow you to walk the arrays to find the object.
1. If an extended config is specified as a string name, then that is the name that will be used. Otherwise, its `name` will be used.
1. If an extended config doesn't have a name, it will be assigned a name of `"extended config N"` where `N` is the index in the `extends` array. Similar to base objects, if an extended object is contained in a nested array then `N` represents the concatenation of array indices starting from the `extends` array.

#### Extending Arrays

Arrays can also be used in `extends` (to eliminate the guesswork of what type a config is). When evaluating `extends`, ESLint internally calls `.flat(Infinity)` on the array and then processes the config objects as discussed in the previous example. Consider the following:

```js
import { defineConfig } from "eslint";
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default defineConfig([

    {
        files: ["**/src/*.js"],
        extends: [
            [js.configs.recommended, example.configs.recommended]
        ]
    }

]);
```

This is equivalent to the following:

```js
import { defineConfig } from "eslint";
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default defineConfig([

    {
        files: ["**/src/*.js"],
        extends: [js.configs.recommended, example.configs.recommended]
    }

]);
```

The extended objects are evaluated in the same order and result in the same final config.

#### Extending Named Configs

While the previous examples are an improvement over current situation, we can go a step further and restore the use of named configs by allowing strings inside of the `extends` array. In eslintrc, you could do something like this:

```json
{
    "plugins": [
        "react"
    ],
    "extends": [
        "eslint:recommended",
        "plugin:react/recommended"
    ],
    "rules": {
       "react/no-set-state": "off"
    }
}
```

The `plugin:` prefix was necessary because plugin configs were implemented after shared configs, and we needed a way to differentiate. The special `"eslint:recommended"` string was necessary to differentiate it from both shareable configs and plugin configs. This made the eslintrc version of `extends` a bit messy.

With flat config, however, we can fix that by omitting the `plugin:` prefix and we no longer need `"eslint:recommended"`. (Packages that only export a single config can be placed directly into the `extends` array.) The same config written using this proposal looks like this:

```js
import { defineConfig } from "eslint";
import js from "@eslint/js";
import react from "eslint-plugin-react";

export default defineConfig([
    {
        plugins: {
            js,
            react
        },
        extends: ["js/recommended", "react/recommended"],
        rules: {
            "react/no-set-state": "off"
        }
    }
]);
```

Here, `"js/recommended"` refers to the plugin defined as `js`. Internally, ESLint will look up the plugin with the name `js`, looks at the `configs` key, and retrieve the `recommended` key as a replacement for the string `"js/recommended"`. The same process occurs for `"react/recommended"`.

This has several advantages:

1. It gives ESLint knowledge of where the config is coming from. When passing in an object or an array, ESLint doesn't know the config's origin. With more information, we can give better error messages when things go wrong.
1. The config is guaranteed to have a name (the string in `extends`) that can be used for debugging purposes.
1. It encourages plugin authors to use the `configs` key on their plugin in order to allow this usage.
1. It allows ESLint to modify configs before they are used (see below).

#### Reassignable Plugin Configs

When using named configs, this gives ESLint the opportunity to inspect and change the config before applying it. This means we have the opportunity to improve the plugin config experience.

Right now, bundling a config in a plugin is a bit of a messy process for two reasons:

1. You need to include the plugin in the config directly, which means that you can't just have an single object literal defined as the plugin. You need to define the plugin first and then add the configs with the plugin reference second.
1. You don't actually know what namespace a user will assign the plugin to, so while you might assume someone will import `eslint-plugin-import` as `import` and you then refer to rules from the plugin as `import/rule`, there is nothing preventing the user from assigning the plugin to a different name. In that case, the user won't know how to override rule options or assign different languages from the plugin.

Here's an actual example from [`@eslint/json`](https://github.com/eslint/json):

```js
const plugin = {
	meta: {
		name: "@eslint/json",
		version: "0.6.0", // x-release-please-version
	},
	languages: {
		json: new JSONLanguage({ mode: "json" }),
		jsonc: new JSONLanguage({ mode: "jsonc" }),
		json5: new JSONLanguage({ mode: "json5" }),
	},
	rules: {
		"no-duplicate-keys": noDuplicateKeys,
		"no-empty-keys": noEmptyKeys,
	},

    // can't include the plugin here because it must reference `plugin`
	configs: {},
};

Object.assign(plugin.configs, {
	recommended: {

        // now we can reference plugin
		plugins: { json: plugin },
		rules: {
			"json/no-duplicate-keys": "error",
			"json/no-empty-keys": "error",
		},
	},
});

export default plugin;
```

Here, we are hardcoding the namespace `json` even though that might not be the namespace that the user assigns to this plugin. This is something we can now address with the use of `extends` because we have the ability to alter the config before inserting it.

Instead of requiring plugin configs to include the plugin in the config, we can have the plugin define a preferred namespace to indicate that any rules, processors, and plugins containing that namespace need to be rewritten using the namespace the user assigned. For example, we could rewrite the JSON plugin like this:

```js
export default {
	meta: {
		name: "@eslint/json",
        namespace: "json",
		version: "0.6.0", // x-release-please-version
	},
	languages: {
		json: new JSONLanguage({ mode: "json" }),
		jsonc: new JSONLanguage({ mode: "jsonc" }),
		json5: new JSONLanguage({ mode: "json5" }),
	},
	rules: {
		"no-duplicate-keys": noDuplicateKeys,
		"no-empty-keys": noEmptyKeys,
	},

	configs: {
        recommended: {
            rules: {
                "json/no-duplicate-keys": "error",
                "json/no-empty-keys": "error",
            },
        },
    },
};
```

In an `eslint.config.js` file:

```js
import { defineConfig } from "eslint";
import json from "@eslint/json";

export default defineConfig([
    {
        plugins: {
            j: json
        },
        files: ["**/*.json"],
        extends: ["j/recommended"],
        rules: {
            "j/no-empty-keys": "warn"
        }
    }
]);
```

Here, the user has assigned the namespace `j` to the JSON plugin. When ESLint loads this config, it sees `extends` with a value of `"j/recommended"`. It looks up the plugin referenced by `j` and then the config named `"recommended"`. It then inserts an entry for `j` in `plugins` that references the plugin. The next step is to look through the config for all the `json` references and replace that with `j` so the references are correct (which we can determine by looking at `meta.namespace`).

**Note:** The `meta.namespace` value is also valuable outside of this use case. If we ever run into a case where someone has used `json/` as a prefix for a rule, language, or processor, but has only ever namespaced the plugin as `j`, we can then search through registered plugins to see if the JSON plugin is included elsewhere. This gives us the opportunity to let the user know of their error with instructions on how to fix it.

## Documentation

This will require documentation changes and an introductory blog post.

At a minimum, these pages will have to be updated:

* https://eslint.org/docs/latest/use/getting-started#configuration
* https://eslint.org/docs/latest/use/configure/configuration-files
* https://eslint.org/docs/latest/extend/plugins#configs-in-plugins
* https://eslint.org/docs/latest/use/configure/combine-configs

We should also update the documentation for the `files` array to indicate that it can contain an array of arrays.

## Drawbacks

1. While this may reduce complexity for users, it increases the complexity of config evaluation inside of ESLint. This necessarily means a performance cost that we won't be able to quantify until implementation.
1. Users will have to know which version of ESLint supports `defineConfig` in order to use it. There really isn't an easy way to feature test this capability other than to inspect the version of the current ESLint package.
1. We'll be introducing a difference in the expected config object format from what ESLint uses internally. Specifically, only `defineConfig()` arguments will support `extends`, so anyone who chooses not to use `defineConfig()` might be confused as to why `extends` doesn't work.
1. We'll create another package to maintain and need to coordinate releases with the `eslint` package.

## Backwards Compatibility Analysis

With `defineConfig()` available in the `@eslint/config-helpers` package, users of flat config in ESLint v8.x and earlier v9.x will be able to use this functionality even though they'll need to manually install the package.

There are no breaking changes in this proposal.

## Alternatives

### Enable `extends` and Flattening in ESLint Directly

The original proposal for this RFC was to add `extends` into the implementation of `FlatConfigArray` (or alternatively in `ConfigArray`). This approach would allow writing a config file such as this:

```js
import globals from "globals";

const config1 = {
    name: "config1",
    files: ["**/*.cjs.js"],
    languageOptions: {
        sourceType: "commonjs",
        globals: {
            ...globals.node
        }
    }
};

const config2 = {
    name: "config2",
    files: ["**/*.js"],
    ignores: ["**/tests"],
    rules: {
        "no-console": "error"
    }
};


export default [

    {
        name: "myconfig",
        files: ["**/src/*.js"],
        ignores: ["**/*.test.js"]
        extends: [config1, config2],
        rules: {
            semi: "error"
        }
    }

];
```

The benefit of this approach is that everything is still controlled inside of ESLint itself. There's nothing additional to learn as a user aside from using the `extends` key. 

However, there were several identified problems with this approach:

1. **Older ESLint versions locked out.** Users would have to upgrade to the latest ESLint version to use `extends`, meaning that folks who were still on ESLint v8.x or earlier v9.x would not be able to use this functionality at all.
1. **Ecosystem compatibility.** There was a danger that plugins could export configs with `extends`, which users in earlier ESLint versions would attempt to use and get an error. Because support for `extends` would be tied to a specific ESLint version, that made it more difficult to create a config that would work in all ESLint versions that support flat config.
1. **Config Inspector compatibility.** The original plan to implement the functionality in `FlatConfigArray` posed a problem because `FlatConfigArray` is not exported from the `eslint` package. We would either have to export `FlatConfigArray` or implement the functionality in `ConfigArray`. However, `ConfigArray`, being a generic base class, has no concept of plugins, so we'd either have to introduce that concept or not implemented name config extends. There were a lot of different tradeoffs to consider here.

### An `extend()` function

As [mentioned by Kirk Waiblinger](https://github.com/eslint/rfcs/pull/126/files#r1860258048), we could also do a smaller version of `defineConfig()` approach by just defining an `extend()` function that users could use whenever they want to extend a config, such as:

```js
import { extend } from "eslint";
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default [
  extend(
    {
      files: ["**/src/*.js"],
    },
    [js.configs.recommended, example.configs.recommended],
  ),
  {
    files: ["**/*.cjs"],
    languageOptions: {
        sourceType: "commonjs"
    }
  }
];
```

This has similar upsides and downsides to `defineConfig`, with the additional downside that this moves the config further away from the JSON-like visual that makes it easier to scan config files.

## Open Questions

1. **How should Config Inspector show extended configs?** Right now, Config Inspector would receive the already-flattened config array. It could infer from the `>` in config names which configs were related to one another, but should how would that work when a config didn't have a name? Should we maybe provide additional data in the form of a symbol property config objects that Config Inspector can read to determine the relationship between config objects? And would Config Inspector alter its view in some way to show this relationship?

## Frequently Asked Questions

### Why are named configs a part of this RFC instead of a separate one?

Named configs are included here for three reasons:

1. It helps to normalize the application of extended configs.
1. It encourages plugins to export configs on the `configs` key.
1. It directly affects how this proposal is implemented.

### Why is `files` intersected with extended objects?

This matches the behavior of the eslintrc `overrides.extends` behavior, and it ensures that extending arrays results in predictable behavior.

The alternative, replacing `files` completely with the base config values, means that you can't safely extend an array of config objects that might only intend to match specific file patterns. Blindly overwriting `files` can lead to hard-to-detect errors in this case.

### What will happen for plugins that don't define `meta.namespace`?

They will be treated the same as they are now.

### Will plugins eventually be required to define `meta.namespace`?

No. This is a feature we want to encourage but will not require. We want all existing plugins to continue working.

## Related Discussions

* https://github.com/eslint/eslint/discussions/16960#discussioncomment-11147407
* https://github.com/eslint/eslint/discussions/16960#discussioncomment-11160226
* https://github.com/eslint/eslint/issues/18040
* https://github.com/eslint/eslint/issues/18752
