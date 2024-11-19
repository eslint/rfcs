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
import eslintPluginImportX from 'eslint-plugin-import-x'

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

1. Allow arrays in config arrays (in addition to objects)
1. Introduce an `extends` key in config objects

### Allow Arrays in Config Arrays

This was part of the original RFC but was disabled over concerns about the complexity involved with importing configs from packages and having no idea whether it was an object, an array, or an array with a mix of objects and arrays. This mattered most when users needed to use `map()` to apply different `files` or `ignores` to configs, but with `extends`, that concern goes away.

Support for nested arrays is already in `ConfigArray`, it's just disabled.

### Introduce an `extends` Key

The `extends` key is an array that may contain other configurations. When used, ESLint will expand the config object into multiple config objects in order to create a flat structure to evaluate. 

#### Extending Objects

You can pass one or more objects in `extends`, like this:

```js
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default [

    {
        files: ["**/src/*.js"],
        extends: [js.configs.recommended, example.configs.recommended]
    }

];
```

Assuming these config objects do not contain `files` or `ignores`, ESLint will convert this config into the following:

```js
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default [
    {
        ...js.configs.recommended
        files: ["**/src/*.js"],
    },
    {
        ...example.configs.recommended
        files: ["**/src/*.js"],
    },
];
```

If the objects in `extends` contain `files` or `ignores`, then ESLint will merge those values with the values found in the config using `extends` such that they intersect. For example:

```js
import globals from "globals";

const config1 = {
    name: "config1",
    files: ["**/*.cjs"],
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
        extends: [config1, config2],
        rules: {
            semi: "error"
        }
    }

];
```

Here, the `files` keys will be combined and the `ignores` key will be inherited, resulting in a final config that looks like this:

```js

export default [

    // combined files
    {
        name: "myconfig > config1",
        files: [["**/src/*.js", "**/*.cjs"]],
        languageOptions: {
            sourceType: "commonjs",
            globals: {
                ...globals.node
            }
        }
    },

    // combined files and inherited ignores
    {
        name: "myconfig > config2",
        files: [["**/src/*.js", "**/*.js"]],
        ignores: ["**/tests"],
        rules: {
            "no-console": "error"
        }
    },

    // original config
    {
        name: "myconfig",
        files: ["**/src/*.js"],
        rules: {
            semi: "error"
        }
    }

];
```

Note that the `name` key is generated for calculated config objects so that it indicates the inheritance from another config that doesn't exist in the final representation of the config array.

#### Extending Arrays

Arrays can also be used in `extends` (to eliminate the guesswork of what type a config is). When evaluating `extends`, ESLint internally calls `.flat()` on the array and then processes the config objects as discussed in the previous example. Consider the following:

```js
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default [

    {
        files: ["**/src/*.js"],
        extends: [
            [js.configs.recommended, example.configs.recommended]
        ]
    }

];
```

This is equivalent to the following:

```js
import js from "@eslint/js";
import example from "eslint-plugin-example";

export default [

    {
        files: ["**/src/*.js"],
        extends: [js.configs.recommended, example.configs.recommended]
    }

];
```

The extended objects are evaluated in the same order and result in the same final config.

#### Extending Named Configs

While the previous examples are an improvement over current situation, we can go a step further and restore the use of named configs by allowing strings inside of the `extends` array. When provided, the string must refer to a config contained in a plugin. For example:

```js
import js from "@eslint/js";

export default [

    {
        plugins: {
            js
        },
        files: ["**/src/*.js"],
        extends: ["js/recommended"]
    }

];
```

Here, `js/recommended` refers to the plugin defined as `js`. Internally, ESLint will look up the plugin with the name `js`, looks at the `configs` key, and retrieve the `recommended` key as a replacement for the string `"js/recommended"`. 

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

Instead of using a hardcoded plugin namespace, plugins can instead use `#` to indicate that they'd like to have the plugin itself included and referenced using the namespace the user assigned. For example, we could rewrite the JSON plugin like this:

```js
export default {
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

	configs: {
        recommended: {
            plugins: { "#": null },
            rules: {
                "#/no-duplicate-keys": "error",
                "#/no-empty-keys": "error",
            },
        },
    },
};
```

In an `eslint.config.js` file:

```js
import json from "@eslint/json";

export default [
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
];
```

Here, the user has assigned the namespace `j` to the JSON plugin. When ESLint loads this config, it sees `extends` with a value of `"j/recommended"`. It looks up the plugin referenced by `j` and then the config named `"recommended"`. It then sees that there's a plugin entry for `"#"` and replaces that with an entry for `j` that points to the JSON plugin. The next step is to look through the config for all the `#` references and replace that with `j` so the references are correct. 

### Implementation Details

The implementation of this feature requires the following changes:

1. Enable nested arrays in `FlatConfigArray`
1. Create a new `ConfigExtender` class that encapsulates the functionality for extending configs.
1. Update `FlatConfigArray` to use `ConfigExtender` inside of the `normalize()` and `normalizeSync()` methods.

#### Enable Nested Arrays in `FlatConfigArray`

Pass [`extraConfigTypes: ["array"]`](https://github.com/eslint/rewrite/blob/a957ee351c27ac1bf22966768cf8aac8c12ce0d2/packages/config-array/src/config-array.js#L572) to [`super()`](https://github.com/eslint/eslint/blob/fd33f1315ac59b1b3828dbab8e1e056a1585eff0/lib/config/flat-config-array.js#L95-L98) to enable nested arrays.

#### The `ConfigExtender` Class

The `ConfigExtender` class is the primary new class for handling the `extends` key in config objects.

```ts
interface ConfigExtender {
    evaluate(config: ConfigObject): Array<ConfigObject>;
}
```

The `ConfigExtender#evaluate()` method accepts a single config object and returns an array of compatible config objects. If the config object doesn't contain `extends` then it just passes that object back in an array.

#### Update `FlatConfigArray` to use `ConfigExtender`

The [`normalize()`](https://github.com/eslint/eslint/blob/fd33f1315ac59b1b3828dbab8e1e056a1585eff0/lib/config/flat-config-array.js#L141) and [`normalizeSync()`](https://github.com/eslint/eslint/blob/fd33f1315ac59b1b3828dbab8e1e056a1585eff0/lib/config/flat-config-array.js#L160) methods will be updated to use a `ConfigExtender` instance to evaluate any `extends` keys. To do this, we'll need to evaluate each object for `extends` before allowing calling the superclass method. Here's an example of what `normalize()` will look like:

```js

const extender = new ConfigExtender();

class FlatConfigArray {

    // snip

    normalize(context) {

        // make sure not to make any changes if already normalized
        if (this.isNormalized()) {
            throw new Error("...");
        }

        // replace each element with an array
        this.forEach((element, i) => {
            const configs = Array.isArray(element) ? element : [element];
            this[i] = configs.flat().map(config => extender.evaluate(config))
        });

        // proceed as usual
        return super.normalize(context)
            .catch(error => {
                if (error.name === "ConfigError") {
                    throw wrapConfigErrorWithDetails(error, this[originalLength], this[baseLength]);
                }

                throw error;

            });
    }
}
```

## Documentation

This will require documentation changes and an introductory blog post.

At a minimum, these pages will have to be updated:

* https://eslint.org/docs/latest/use/getting-started#configuration
* https://eslint.org/docs/latest/use/configure/configuration-files
* https://eslint.org/docs/latest/extend/plugins#configs-in-plugins
* https://eslint.org/docs/latest/use/configure/combine-configs

## Drawbacks

1. Introducing a new key to flat config at this point means that users of ESLint v8 won't be able to use it. Even though v8 is EOL, we had made an attempt to provide forward compatibility to make people's transition easier.
1. While this may reduce complexity for users, it increases the complexity of config evaluation inside of ESLint. This necessarily means a performance cost that we won't be able to quantify until implementation.
1. Users will have to know which version of ESLint supports `extends` in order to use it. There really isn't an easy way to feature test this capability other than to inspect the version of the current ESLint package.

## Backwards Compatibility Analysis

This proposal is additive and does not affect the way existing configurations are evaluated.

## Alternatives

1. **A utility package.** Instead of including `extends` in flat config itself, we could create a utility package that provides a method to accomplish the same thing. This has the advantage that ESLint v8 users could also use this utility package, but the downside that it would require users to install Yet Another Package to do something that many feel should be available out of the box.
1. **A utility function.** A variation of the previous approach is to export a function from the `eslint` package that people could use to extend configs. This has the advantage that it would be easier to feature test for but the downside that it still requires an extra step to use `extends`.

## Open Questions

1. **Should `files` and `ignores` be merged (as in this proposal) or just dropped completely?** I can see an argument for dropping them completely, but I also know that some folks share configs with multiple different `files` patterns in an array that is meant to be used together.
1. **Does the `#` pattern in plugin configs cause any unintended side effects?** The string `"#"` would be an invalid package name, so it's unlikely to cause any naming collisions.
1. **Should `extends` be allowed in plugin configs?** This RFC does not allow `extends` in plugin configs for simplicity. Allowing this would mean the possibility of nested `extends`, which may be desirable but also more challenging to implement. Further, if plugins export configs with `extends`, then that automatically means those configs cannot be used in any earlier versions of ESLint. However, not allowing it also means having different schemas for user-defined and plugin configs, which is a significant downside.

## Frequently Asked Questions

### Why is `"#"` assigned as `null` in the example?

This allows us to very quickly check `plugin.plugins["#"]` to see if there's a `"#"` that needs evaluating. The value actually doesn't matter because it will be replaced, so `null` seemed like a nice shorthand. Without the `"#"` entry in `plugins`, we'd need to search through `rules`, `processor`, `language`, and potentially other future keys to see if `"#"` was referenced. I considered that initially, but I think being explicit is a better idea.

### Why is this functionality added to `FlatConfigArray` instead of `@eslint/config-array`

In order to support named configs, we need the concept of a plugin. The generic `ConfigArray` class has no concept of plugins, which means the functionality needs to live in `FlatConfigArray` in some way. There may be an argument for supporting `extends` with just objects and arrays in `ConfigArray`, with `FlatConfigArray` overriding that to support named configs, but that would increase the complexity of implementation.

If we end up not supporting named configs, then we can revisit this decision.

## Related Discussions

* https://github.com/eslint/eslint/discussions/16960#discussioncomment-11147407
* https://github.com/eslint/eslint/discussions/16960#discussioncomment-11160226
* https://github.com/eslint/eslint/issues/18040
* https://github.com/eslint/eslint/issues/18752
