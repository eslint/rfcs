- Repo: eslint/rewrite
- Start Date: 2025-04-23
- RFC PR: <https://github.com/eslint/rfcs/pull/132>
- Authors: [JoshuaKGoldberg](https://github.com/JoshuaKGoldberg)

# Exporting a `definePlugin` function

## Summary

Proposes a `definePlugin` function to `@eslint/plugin-kit` as the recommended way to group plugin configs, meta, and rules.

## Motivation

Defining the exported values of an ESLint plugin is a "loose" process right now.
ESLint core provides only recommendations, not standardized factory functions; the community also has centralized only on common patterns.
Several dozen variations in how plugins are defined have evolved, ranging from manual declarations of `configs`, to custom scripts, to auto-generated JavaScript or TypeScript files.

This divergence in implementation details results in conceptual and practical overhead for plugin authors.
Without a standardized factory function to define a plugin:

- There is no first-party way for plugins to receive type hints during development the way [`eslint/config`'s `defineConfig`](https://eslint.org/docs/latest/use/configure/configuration-files#configuration-file) assists in authoring configs
- Plugin names are often defined in at least four separate places:
  - `configs[string].name`, such as `"example/recommended"`
  - `configs[string].rules`, such as `{ "example/rule": "error" }`
  - `rules[string]`, such as `{ "example/rule": { ... } }`
  - `meta.name`, such as `"eslint-plugin-example"`
- Onboarding to a new plugin necessitates learning its -often unique- way of generating configs
- `configs` and `rules` often cross-reference each other, so there is no single clean way to define them in one statement:

  - Some plugins imperatively define properties in two steps as suggested in the [ESLint plugin documentation](https://eslint.org/docs/latest/extend/plugins#configs-in-plugins):

    ```js
    const plugin = {
      meta: { name, version },
      configs: {},
      rules,
    };

    Object.assign(plugin.configs, {
      recommended: [
        {
          plugins: {
            example: plugin,
          },
          rules: recommendedRules,
        },
      ],
    });
    ```

  - Other plugins use [a single computed `get()` for `plugins`](https://github.com/facebook/react/blob/914319ae595010cd5d3f0e277c77eb86da18e4f0/packages/eslint-plugin-react-hooks/src/index.ts#L30):

    ```js
    export const plugin = {
      configs: {
        recommended: {
          name: "example/recommended",
          plugins: {
            get example() {
              return plugin;
            },
          },
          rules,
        },
      },
      meta: { name, version },
      rules: recommendedRules,
    };
    ```

This RFC proposes creating a `definePlugin` function to be exported from `@eslint/plugin-kit`.
It would provide a single recommended factory for defining the structure of a typical ESLint plugin.
`definePlugin` would automate the parts of plugin definitions that many plugins manually redefine:

- Using the same name in multiple places
- Generating `configs.recommended` and other configurations based on rule metadata

As with `defineConfig`, `definePlugin` would be a purely optional, additive change.
Existing and future plugins would continue to work as-is without a `definePlugin` call.

## Detailed Design

The `@eslint/plugin-kit` package would export a `definePlugin` function that takes in a single required options object.
The options properties would allow authors to specify:

- `meta` _(required)_: The same `name` as [Meta Data in Plugins](https://eslint.org/docs/latest/extend/plugins#meta-data-in-plugins), along with an optional `version`
- `rules` _(optional)_: The same object as [Rules in Plugins](https://eslint.org/docs/latest/extend/plugins#rules-in-plugins), but also allowing functions that return a rule
- any other plugin properties

It would return a plugin with:

- `configs`: configuration object(s) generated per [Configs](#configs)
- all other provided properties as-is

The following code block creates a standard plugin:

```js
import { definePlugin } from "@eslint/plugin-kit";

import packageData from "../package.json" with { type: "json" };
import { rules } from "./rules/index.js";

export const plugin = definePlugin({
  meta: { name: packageData.name, version: packageData.version },
  rules,
});
```

> `rules` is assumed to be a typical rules definition object like `{ "example/a": { ... }, ... }`.

That definition would be the equivalent of:

```js
// (equivalent to the previous snippet)

import { definePlugin } from "@eslint/plugin-kit";

import packageData from "../package.json" with { type: "json" };
import { rules } from "./rules/index.js";

const plugin = {
  meta: { name: packageData.name, version: packageData.version },
  configs: {},
  rules,
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: Object.fromEntries(
        Object.entries(plugin.rules)
          .filter((rule) => rule.meta.docs?.recommended)
          // (see #### Default Generated Config later for rule severities)
          .map((rule) => [rule, getRuleSeverity(rule)]),
      ),
    },
  ],
});
```

`definePlugin`'s design intentionally avoids referring to a `plugin` value.
No `Object.assign()`, `Object.defineProperty()`, or `get()` will be required in the making of an ESLint plugin with `definePlugin`.

### Configs

`definePlugin` will add two properties to all generated `configs` objects:

- `name`: will default to the plugin's namespace + `"/"` + the config's key
- `plugin`: merged with an object defining the plugin under its name

`configs` is the only property `definePlugin` receives with a different shape than the output plugin object.

- If `configs` is not provided, a default config is generated.
- `configs` may be provided as a shape that does not include the plugin's name.
  In this case, its values will be transformed to have the plugin's name added.

The following two sections describe those two behaviors.

#### Default Generated Config

If `configs` is not provided, then `definePlugin` will create a `recommended` config with the following properties:

- `name`: per [configuration naming conventions](https://eslint.org/docs/latest/use/configure/configuration-files#configuration-naming-conventions)
- `plugins`: an object keying `meta.name` to the plugin object
- `rules`: setting severities for rules whose `meta.docs?.recommended` is one of the following values:
  - `true`: set to `"error"`
  - any valid rule severity: that severity

For example, given this plugin with two rules:

- `my-rule-a`: with `meta.docs.recommended = "error"`:
- `my-rule-b`: with `meta.docs.recommended = "warn"`:

```ts
const plugin = definePlugin({
  meta: {
    name: "example",
  },
  rules: {
    "my-rule-a": myRuleA,
    "my-rule-b": myRuleB,
  },
});
```

The equivalent generated plugin object would be:

```js
// (equivalent to the previous snippet)

const plugin = {
  configs: {},
  meta: {
    name: "example",
  },
  rules: {
    "my-rule-a": myRuleA,
    "my-rule-b": myRuleB,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule-a": "error",
        "example/my-rule-b": "warn",
      },
    },
  ],
});
```

The internal logic would be equivalent to filtering the rules on whether they are recommended:

```js
// (equivalent to the two previous snippets)

function getRuleSeverity(rule) {
  return rule.meta.docs.severity === true ? "error" : rule.meta.docs.severity;
}

const meta = {
  name: "example",
};

const rules = {
  "my-rule-a": myRuleA,
  "my-rule-b": myRuleB,
};

const plugin = {
  configs: {},
  meta,
  rules: Object.values(rules),
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: Object.fromEntries(
        rules
          .filter((rule) => rule.meta.docs?.recommended)
          .map(([ruleName, rule]) => [
            `${meta.name}/${ruleName}`,
            getRuleSeverity(rule),
          ])
      ),
    },
  ],
});
```

#### Generated Configs

If `configs` is provided, each of its objects' `rules` must be provided either as:

- `null | undefined`: to preserve that value.
- An array containing any number of elements.
  Each element may be either:
  - The rule object itself
  - An array containing a rule object and severity
  - An object with string keys and severity values

Each `rules` element will be transformed into key-value pairs.

- Each key will be `${meta.name}/${ruleName}`.
- Each value will be the provided severity if it exists, or `"error"` if not

Those key-value pairs will be merged into into one final `rules` object.

| Element Type                                | Example Input                   | Example Output                               |
| ------------------------------------------- | ------------------------------- | -------------------------------------------- |
| Rule object                                 | `myRule`                        | `{ "example/my-rule": "error" }`             |
| Object with string keys and severity values | `{ "example/my-rule": "warn" }` | `{ "example/my-rule": "warn" }`              |
| Tuple containing rule object and severity   | `[myRule, "warn"]`              | `{ "example/my-rule": "warn" }`              |
| Array of objects                            | `[ { ... }, { ... } ]`          | `[ { name, plugins, ... }, { ... } ]` **\*** |

**\***For a `config` value provided as an array of objects, `name` and `plugins` will be set as default values on only the first element.
This allows plugin authors to define longer plugins with arrays of config objects.
See [Example: Multiple Config Elements](#example-multiple-config-elements).

##### Example: Custom `recommended` Config

This plugin has four rules, `myRuleA`, `myRuleB`, `myRuleC`, and `myRuleD`.
It demonstrates using `definePlugin` to create a single config:

```ts
const plugin = definePlugin({
  configs: {
    recommended: [
      myRuleA,
      [myRuleB, true],
      [myRuleC, "error"],
      [myRuleD, "warn"],
    ],
  },
  meta: {
    name: "example",
  },
  rules: {
    "my-rule-a": myRuleA,
    "my-rule-b": myRuleB,
    "my-rule-c": myRuleC,
    "my-rule-d": myRuleD,
  },
});
```

The equivalent generated plugin object would be:

```js
// (equivalent to the previous snippet)

const plugin = {
  configs: {},
  meta: {
    name: "example",
  },
  rules: {
    "my-rule-a": myRuleA,
    "my-rule-b": myRuleB,
    "my-rule-c": myRuleC,
    "my-rule-d": myRuleD,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule-a": "error",
        "example/my-rule-b": "error",
        "example/my-rule-c": "error",
        "example/my-rule-d": "warn",
      },
    },
  ],
});
```

Note that this plugin declaration is provided for explanatory purposes only.
It would have been cleaner for it to set the `meta.docs.recommended` property in its rules.
See [Default Generated Config](#default-generated-config) for a similar, more implicitly generated plugin.

##### Example: Filtered Custom Configs

A plugin akin to `typescript-eslint` with `recommended`, `strict`, and `*TypeAware` configs could define those four configs by filtering the values of a `rules` object.
Suppose this plugin's `rules` contains four rules:

1. `my-rule-a`: with `meta.docs = { recommended: true, typeAware: false }`
1. `my-rule-b`: with `meta.docs = { recommended: true, typeAware: true }`
1. `my-rule-c`: with `meta.docs = { recommended: "strict", typeAware: false }`
1. `my-rule-d`: with `meta.docs = { recommended: "strict", typeAware: true }`

Its plugin definition could look like:

```js
definePlugin({
  configs: {
    recommended: {
      rules: Object.values(rules).filter((rule) => rule.meta.docs.recommended),
    },
    recommendedTypeAware: {
      rules: Object.values(rules).filter(
        (rule) => rule.meta.docs.recommended && rule.meta.docs.typeAware
      ),
    },
    strict: {
      rules: Object.values(rules).filter(
        (rule) => rule.meta.docs.recommended === "strict"
      ),
    },
    strictTypeAware: {
      rules: Object.values(rules).filter(
        (rule) =>
          rule.meta.docs.recommended === "strict" && rule.meta.docs.typeAware
      ),
    },
  },
  meta: {
    name: "example",
  },
  rules: {
    "my-rule-a": myRuleA,
    "my-rule-b": myRuleB,
    "my-rule-c": myRuleC,
    "my-rule-d": myRuleD,
  },
});
```

The equivalent generated plugin object would be:

```js
// (equivalent to the previous snippet)

const plugin = {
  configs: {},
  meta: {
    name: "example",
  },
  rules: {
    "my-rule-a": myRuleA,
    "my-rule-b": myRuleB,
    "my-rule-c": myRuleC,
    "my-rule-d": myRuleD,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule-a": "error",
      },
    },
  ],
  recommendedTypeAware: [
    {
      name: "example/recommendedTypeAware",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule-a": "error",
        "example/my-rule-b": "error",
      },
    },
  ],
  strict: [
    {
      name: "example/strict",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule-a": "error",
        "example/my-rule-c": "error",
      },
    },
  ],
  strictTypeAware: [
    {
      name: "example/strictTypeAware",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule-a": "error",
        "example/my-rule-b": "error",
        "example/my-rule-c": "error",
        "example/my-rule-d": "error",
      },
    },
  ],
});
```

##### Example: Including Additional Options

A plugin that provides additional properties such as `languageOptions` could define them in its `configs.recommended`.
This plugin provides:

- A single rule, `myRule`, with `meta.docs.recommended = true`:
- `languageOptions`: as part of its `recommended` config

```js
definePlugin({
  configs: {
    recommended: {
      languageOptions: {
        globals: {
          myGlobal: "readonly",
        },
      },
      rules: [myRule],
    },
  },
  meta: {
    name: "example",
  },
  rules: {
    "my-rule": myRule,
  },
});
```

The equivalent generated plugin object would be:

```js
// (equivalent to the previous snippet)

const plugin = {
  configs: {},
  meta: {
    name: "example",
  },
  rules: {
    "my-rule": myRule,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      languageOptions: {
        globals: {
          myGlobal: "readonly",
        },
      },
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule": "error",
      },
    },
  ],
});
```

##### Example: Multiple Config Elements

If a plugin provides multiple objects in a `configs` array, only the first object will have `name` and `plugins` added in as defaults.
For example, this plugin is set up akin to the current `eslint-plugin-markdown`.
It provides multiple entries in its `recommended` config and only specifies `rules` in the last element:

```js
definePlugin({
  configs: {
    recommended: [
      {
        name: "example/recommended/plugin",
        rules: null,
      },
      {
        name: "example/recommended/processor",
        files: ["**/*.md"],
        processor: "example/markdown",
      },
      {
        name: "example/recommended/code-blocks",
        files: ["**/*.md/**"],
        languageOptions: {
          parserOptions: {
            ecmaFeatures: {
              impliedStrict: true,
            },
          },
        },
        rules,
      },
    ],
  },
  meta,
  processors: {
    markdown: processor,
  },
  rules,
});
```

The equivalent generated plugin object would be:

```js
// (equivalent to the previous snippet)

const plugin = {
  configs: {},
  meta: {
    name: "example",
  },
  processors: {
    markdown: processor,
  },
  rules: {
    "my-rule": myRule,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended/plugin",
      rules: null,
    },
    {
      name: "example/recommended/processor",
      files: ["**/*.md"],
      processor: "example/markdown",
    },
    {
      name: "example/recommended/code-blocks",
      files: ["**/*.md/**"],
      languageOptions: {
        parserOptions: {
          ecmaFeatures: {
            impliedStrict: true,
          },
        },
      },
      rules: {
        "example/my-rule": "error",
      },
    },
  ],
});
```

##### Example: Referencing Other Plugins' Rules

If a `config` value contains an objects with string keys and severity values, those objects' keys will not be modified.
This means objects can refer to other plugins' rules.

For example, this plugin sets three rules:

1. `myRule`: its own rule
2. `import/no-anonymous-default-export`: from an external plugin
3. `react-hooks/rules-of-hooks`: from an external plugin

Its definition is equivalent to the current `eslint-plugin-storybook`'s referencing of rules from other plugins:

```js
definePlugin({
  configs: {
    recommended: {
      rules: [
        myRule,
        {
          "import/no-anonymous-default-export": "off",
          "react-hooks/rules-of-hooks": "off",
        },
      ],
    },
  },
  meta: {
    name: "example",
  },
  rules: {
    "my-rule": myRule,
  },
});
```

The equivalent generated plugin object would be:

```js
// (equivalent to the previous snippet)

const plugin = {
  configs: {},
  meta: {
    name: "example",
  },
  rules: {
    "my-rule": myRule,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule": "error",
        "import/no-anonymous-default-export": "off",
        "react-hooks/rules-of-hooks": "off",
      },
    },
  ],
});
```

## Documentation

- The README.md for [`@eslint/plugin-kit`](https://www.npmjs.com/package/@eslint/plugin-kit) will have a section added for `definePlugin`
- The existing [Create Plugins](https://eslint.org/docs/latest/extend/plugins) page will be updated to use the new helper
  - It can also link to the `@eslint/plugin-kit` README.md section for expanded documentation
  - It will also explicitly note that the helper is optional
- Explanatory documentation in the new `eslint-plugin-eslint-plugin` lint rules

## Drawbacks

Adding one more function to learn increases the barrier to entry to authoring plugins.
Developers may not want a helper that they need to understand alongside the existing concepts.
Some might find that obfuscating plugin definitions in this way is conceptual overhead costlier than the current act of writing portions manually.

Adding a first-party standardized config also runs the risk of forcing the community into patterns it does not want.
Authors may find the `definePlugin`-style `configs` cumbersome, for example, and might opt not to use the helper factory as a result.
If a super-majority of plugin authors don't onboard to `definePlugin` then this RFC might lead to an increase in plugin definition styles.

This RFC believes that the net benefits of a first-party `definePlugin` outweigh the risks.
Given how many existing plugins seem to adhere to similar patterns, it is likely that this RFC will net reduce, not increase, implementation divergence.

## Backwards Compatibility Analysis

Fully backward compatible with no breaking changes.
The new `definePlugin` would be opt-in and optional.

### Design Pattern Analysis

Beyond technical breaking changes, this design also needs to work conceptually with how community authors want to define their plugins.
`configs` in particular is a design change that needs to be not break existing patterns.

All non-deprecated plugins with a ✅ _Status_ in [📈 Tracking: Flat Config Support](https://github.com/eslint/eslint/issues/18093) can be represented with the `configs` functions style of definition.

Emoji key:

- ☑️: Exposes just a `recommended` config, optionally alongside `all`, `flat/*`, `legacy-*`, and/or `*-legacy` variants
- ✔️: Exposes multiple configs that are or could be auto-generated from existing values in code
- 🛠️: Exposes multiple configs that could be represented this way by adding 1-2 properties to `meta.rule.docs`
- ❎: Does not export configs with its own rules to begin with

| Plugin                                                                                            | Config(s)                      | Generation Notes                                                 | Representable |
| ------------------------------------------------------------------------------------------------- | ------------------------------ | ---------------------------------------------------------------- | ------------- |
| [@graphql-eslint](https://github.com/graphql-hive/graphql-eslint)                                 | Multiple                       | generated by `pnpm generate:configs`                             | ✔️            |
| [@nuxt/eslint](https://github.com/nuxt/eslint)                                                    | _(none)_                       |                                                                  | ❎            |
| [@redwoodjs/eslint-plugin](https://github.com/redwoodjs/redwood/tree/main/packages/eslint-plugin) | _(none)_                       |                                                                  | ❎            |
| [@typescript-eslint](https://github.com/typescript-eslint/typescript-eslint)                      | Multiple                       | generated by `yarn generate:configs`                             | ✔️            |
| [angular](https://github.com/angular-eslint/angular-eslint)                                       | `all`, `recommended`           | manually matched to `meta.docs.recommended`                      | ☑️            |
| [astro](https://github.com/ota-meshi/eslint-plugin-astro)                                         | Multiple                       | generated by `npm run update`                                    | ✔️            |
| [check-file](https://github.com/dukeluo/eslint-plugin-check-file)                                 | _(none)_                       |                                                                  | ❎            |
| [compat](https://github.com/amilajack/eslint-plugin-compat)                                       | `recommended`, `flat/*`        | one plugin rule                                                  | ☑️            |
| [cypress](https://github.com/cypress-io/eslint-plugin-cypress)                                    | `all`, `recommended`           | manually matched to `meta.docs.recommended`                      | ☑️            |
| [ember](https://github.com/ember-cli/eslint-plugin-ember)                                         | `base`, `recommended`          | manually authored with external rules                            | ✔️            |
| [es-x](https://github.com/eslint-community/eslint-plugin-es-x)                                    | Multiple                       | generated by `scripts/update-lib-configs.js`                     | ✔️            |
| [eslint-comments](https://github.com/eslint-community/eslint-plugin-eslint-comments)              | `recommended`                  | generated by `scripts/update.js`                                 | ☑️            |
| [eslint-plugin](https://github.com/eslint-community/eslint-plugin-eslint-plugin)                  | Multiple                       | dynamically based on `meta.docs.recommended`                     | ✔️            |
| [expect-type](https://github.com/JoshuaKGoldberg/eslint-plugin-expect-type)                       | `recommended`                  | dynamically based on `meta.docs.recommended`                     | ☑️            |
| [functional](https://github.com/eslint-functional/eslint-plugin-functional)                       | Multiple                       | dynamically based on `meta.docs.recommended` with external rules | ✔️            |
| [import-x](https://github.com/un-ts/eslint-plugin-import-x)                                       | Multiple                       | manually authored                                                | 🛠️            |
| [import](https://github.com/import-js/eslint-plugin-import)                                       | Multiple                       | manually authored                                                | 🛠️            |
| [jest](https://github.com/jest-community/eslint-plugin-jest)                                      | Multiple                       | manually authored                                                | 🛠️            |
| [jsdoc](https://github.com/gajus/eslint-plugin-jsdoc)                                             | Multiple                       | manually authored                                                | 🛠️            |
| [jsonc](https://github.com/ota-meshi/eslint-plugin-jsonc)                                         | Multiple                       | generated by `npm run update`                                    | ✔️            |
| [jsx-a11y](https://github.com/jsx-eslint/eslint-plugin-jsx-a11y)                                  | Multiple                       | manually authored                                                | 🛠️            |
| [markdown](https://github.com/eslint/eslint-plugin-markdown)                                      | Multiple                       | generated by `tools/build-rules.js`                              | ✔️            |
| [mocha](https://github.com/lo1tuma/eslint-plugin-mocha)                                           | `all`, `recommended`           | manually authored                                                | 🛠️            |
| [n](https://github.com/eslint-community/eslint-plugin-n)                                          | `recommended`, `recommended-*` | manually matched to `meta.docs.recommended`                      | ✔️            |
| [nx](https://github.com/nrwl/nx)                                                                  | Multiple                       | manually authored configs that only provide external rules       | ❎            |
| [package-json](https://github.com/JoshuaKGoldberg/eslint-plugin-package-json)                     | `recommended`, `legacy-*`      | dynamically based on `meta.docs.recommended`                     | ☑️            |
| [perfectionist](https://github.com/azat-io/eslint-plugin-perfectionist)                           | Multiple                       | dynamically with different options per config                    | ✔️            |
| [playwright](https://github.com/playwright-community/eslint-plugin-playwright)                    | Multiple                       | manually matched to `meta.docs.recommended`                      | ✔️            |
| [prettier](https://github.com/prettier/eslint-plugin-prettier)                                    | `recommended`                  | one plugin rule with external rules                              | ✔️            |
| [promise](https://github.com/eslint-community/eslint-plugin-promise)                              | `recommended`, `flat/*`        | manually authored                                                | 🛠️            |
| [qunit](https://github.com/platinumazure/eslint-plugin-qunit)                                     | `recommended`                  | manually matched to `meta.docs.recommended`                      | ✔️            |
| [react-hooks](https://github.com/facebook/react/tree/main/packages/eslint-plugin-react-hooks)     | `recommended`, `recommended-*` | manually authored                                                | 🛠️            |
| [react-refresh](https://github.com/ArnaudBarre/eslint-plugin-react-refresh)                       | Multiple                       | one rule with different options                                  | ✔️            |
| [react](https://github.com/jsx-eslint/eslint-plugin-react)                                        | Multiple                       | partially dynamically based on `meta.docs.recommended`           | ✔️            |
| [regexp](https://github.com/ota-meshi/eslint-plugin-regexp)                                       | Multiple                       | generated by `npm run update`                                    | ✔️            |
| [security](https://github.com/eslint-community/eslint-plugin-security)                            | `recommended`, `legacy-*`      | manually matched to `meta.docs.recommended`                      | ☑️            |
| [simple-import-sort](https://github.com/lydell/eslint-plugin-simple-import-sort)                  | _(none)_                       |                                                                  | ❎            |
| [solid](https://github.com/solidjs-community/eslint-plugin-solid)                                 | Multiple                       | manually authored                                                | 🛠️            |
| [sonarjs](https://github.com/SonarSource/SonarJS/blob/master/packages/jsts/src/rules/README.md)   | `recommended`, `*-legacy`      | dynamically based on `meta.docs.recommended`                     | ☑️            |
| [storybook](https://github.com/storybookjs/eslint-plugin-storybook)                               | Multiple                       | generated by `pnpm run update-all`                               | ✔️            |
| [stylistic](https://github.com/eslint-stylistic/eslint-stylistic)                                 | Multiple                       | dynamically based on `meta.docs.recommended`                     | ✔️            |
| [svelte](https://github.com/sveltejs/eslint-plugin-svelte)                                        | Multiple                       | manually matched to `meta.docs.recommended`                      | ✔️            |
| [tailwindcss](https://github.com/francoismassart/eslint-plugin-tailwindcss)                       | `recommended`, `flat/*`        | manually matched to `meta.docs.recommended`                      | ☑️            |
| [TanStack Query](https://github.com/TanStack/query)                                               | `recommended`, `flat/*`        | manually matched to `meta.docs.recommended`                      | ☑️            |
| [testing-library](https://github.com/testing-library/eslint-plugin-testing-library)               | Multiple                       | dynamically based on `meta.docs.recommended`                     | ✔️            |
| [turbo](https://github.com/vercel/turbo)                                                          | `recommended`, `flat/*`        | manually matched to `meta.docs.recommended`                      | ✔️            |
| [unicorn](https://github.com/sindresorhus/eslint-plugin-unicorn)                                  | `all`, `recommended`, `flat/*` | dynamically based on `meta.docs.recommended`                     | ☑️            |
| [vitest](https://github.com/veritem/eslint-plugin-vitest)                                         | Multiple                       | manually matched to `meta.docs.recommended`                      | ✔️            |
| [vue-i18n](https://github.com/intlify/eslint-plugin-vue-i18n)                                     | Multiple                       | generated by `jiti scripts/update.ts`                            | ✔️            |
| [vue](https://github.com/vuejs/eslint-plugin-vue)                                                 | Multiple                       | generated by `npm run update`                                    | ✔️            |
| [vuejs-accessibility](https://github.com/vue-a11y/eslint-plugin-vuejs-accessibility)              | `recommended`, `flat/*`        | manually authored                                                | ☑️            |
| [wdio](https://github.com/webdriverio/webdriverio)                                                | `recommended`, `flat/*`        | manually authored                                                | ☑️            |
| [yml](https://github.com/ota-meshi/eslint-plugin-yml)                                             | Multiple                       | generated by `npm run update`                                    | ✔️            |

> Note: this summary table was hand-authored.
> Although it was double-checked, it is susceptible to human error.
> This RFC believes slight inaccuracies in the table do not invalidate its general conclusion: that a most plugins would benefit from a `definePlugin`.

## Alternatives

It might be preferable to release `definePlugin` as a community-authored plugin before standardizing it in `@eslint/plugin-kit`.
Doing so could allow for more rapid changes as community plugins adopt it.
This RFC prefers making the function first-party for visibility and to encourage confidence in plugin author adoption.

### Lazy-Loading Rules

This RFC previously suggested providing features and lint rules around lazy-loading rules.
Doing so would allow plugins to reduce their startup times.
However, lazy-loading requires a non-trivial increase in design complexity.
This RFC now treats lazy-loading as out of scope.

See [5d2557 > 2025-define-plugin/README.md#L705](https://github.com/eslint/rfcs/blob/5d2557787574e85c0e9413ba8cfa75f9215673b8/designs/2025-define-plugin/README.md?plain=1#L705) for the latest version of the RFC that proposed lazy-loading rules.

## Open Questions

### Config Arrays

`eslint-plugin-markdown` defines an unusual `plugin.configs.recommended`:

- An array of config objects instead of just one
- `name` with `/plugin` at the end
- `rules` in the last config object rather than the first

Is it acceptable to change `eslint-plugin-markdown`'s structure to what's suggested under [Example: Multiple Config Elements](#example-multiple-config-elements)

### Legacy Configs

Many plugins include `legacy-*` eslintrc configs as suggested in [Backwards Compatibility for Legacy Configs](https://eslint.org/docs/latest/extend/plugins#backwards-compatibility-for-legacy-configs).
Dropping those configs would be a breaking change that those plugins may want to avoid.
This conflicts with `definePlugin` explicitly only supporting flat config definitions.

Plugin developers can work around that lack of support by manually adding those config properties:

```js
const plugin = definePlugin({ meta, rules });

// TODO: Remove when we drop support for ESLint <=9 / eslintrc
plugin.configs["legacy-recommended"] = {
  plugin: ["example"],
  rules: Object.fromEntries(
    Object.entries(rules)
      .filter(([, rule]) => rule.meta.docs?.recommended)
      .map(([ruleName]) => [ruleName, "error"])
  ),
};
```

Is this an acceptable workaround strategy?

### Rule Creators

This RFC's scope intentionally does include a "rule creator" factory akin to [`@typescript-eslint/rule-creator`](https://typescript-eslint.io/developers/custom-rules/#rulecreator).
Creating such a factory in ESLint could be useful but does not seem to be required for any of the problems `definePlugin` aims to solve.

Should a "rule creator" factory be tackled separately?

## Help Needed

I would love to implement this RFC.

I can also:

- Post this RFC in relevant Discords to draw plugin author attention to it
- Work with `eslint-plugin-eslint-plugin` on feature requests for the proposed lint rules as part of this RFC
- Send draft PRs to community plugins to help onboard them to the new `definePlugin`

## Frequently Asked Questions

### `configs`: Can config generation be made more automatic based on `meta.docs`?

Or: why not define only one function that, given a rule, generates a config or array of configs?
That would suggest using something the following function by default:

```js
definePlugin({
  configs: (rule) =>
    rule.meta.docs.recommended === true
      ? "recommended"
      : rule.meta.docs.recommended,
  meta,
  rules,
});
```

This design would make it difficult to create additional properties for individual configs.
For example, if a plugin exports two configs and only one adds `languageOptions`, it would require additional complexity added to the singular function.

Furthermore, dynamically generating strings is a more complex mental model for plugin authors.
As a result it's also a more complex system to model in type system types for `definePlugin`.
Doing so would at the very least require `definePlugin` to have a type parameter representing the shape of `rule.meta.docs`.
Any operation beyond rudimentary boolean checks or string concatenation would be difficult or impossible to describe even with that information.

This RFC believes requiring a single function per config makes for more clear, readable plugin definitions and is worth the -typically small- number of extra lines.

### `configs`: Should an `all` config also be automatically generated?

Not according to how plugins seem to be authored today.
Some plugins do contain config(s) enabling all their rules in some way.
However, no plugins prominently recommend those "all" configs, and many don't have one at all.

### `configs`: Will the added restrictions break existing plugin design patterns?

Not according to how plugins seem to be authored today.
See [Design Pattern Analysis](#design-pattern-analysis).

This RFC believes that the proposed style of definition directly maps to how users generally expect to understand configs.
As in: each config represents a set of rules filtered from the full list of plugin rules.
Standardizing config logic will encourage plugin authors to continue making predictable plugin config designs.

Plugin authors can always opt out of using `definePlugin` if they want their own, very different style.

### `meta`: Why is `meta.name` required?

It's used to generate keys in `configs[string].rules` objects.

### `configs`: Why generate at all?

The majority of described logic in this RFC is to support automatically generating `configs` for users.
An alternative, leaner approach would be to omit any kind of config generation logic.
Doing so would keep `definePlugin` as a type system / editor Intellisense helper akin to `defineConfig()`.

However, because plugins' `configs[string].plugins` properties may refer to the plugin, there is no way to create a `definePlugin` function that both:

- Allows users to generate a plugin in one statement (i.e. without an `Object.assign` or equivalent)
- Avoids any kind of dynamic `configs` generation (e.g. adding in `plugins` to a config)

The leanest possible approach that keeps plugin generation to one statement would be to only add in the `plugins` property.
For example, this plugin with a single rule could use a version of `definePlugin` that doesn't modify any `rules` properties:

```ts
export const plugin = definePlugin({
  configs: {
    recommended: {
      rules: {
        "example/my-rule": myRule,
      },
    },
  },
  meta: {
    name: "example",
  },
  rules: {
    "my-rule": myRule,
  },
});
```

It might then generate the equivalent config with `plugins`:

```js
// (equivalent to the previous snippet)

import { definePlugin } from "@eslint/plugin-kit";

const plugin = {
  meta: {
    name: "example",
  },
  rules: {
    "my-rule": myRule,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended",
      plugins: {
        example: plugin,
      },
      rules: {
        "example/my-rule": myRule,
      },
    },
  ],
});
```

This RFC believes that including automatic `rule` generation is worth inclusion.
It allows for even more straightforward, simpler plugin definitions.

## Related Discussions

- [vuejs/eslint-plugin-vue#2732](https://github.com/vuejs/eslint-plugin-vue/issues/2732)
- [typescript-eslint/typescript-eslint#10383 Enhancement: Move RuleCreator into its own package with fewer dependencies than utils](https://github.com/typescript-eslint/typescript-eslint/issues/10383)
