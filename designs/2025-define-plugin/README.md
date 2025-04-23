- Repo: eslint/rewrite
- Start Date: 2025-04-23
- RFC PR: <https://github.com/eslint/rfcs/pull/132>
- Authors: [JoshuaKGoldberg](https://github.com/JoshuaKGoldberg)

# Exporting a `definePlugin` function

## Summary

Adds a `definePlugin` function to `@eslint/plugin-kit` as a recommended way to group plugin configs, meta, and rules.

## Motivation

Defining the internals an ESLint plugin is a "loose" process right now: although ESLint provides recommendations and the community has centralized on common patterns, nothing in code explicitly defines many of those recommendations or patterns.
Several dozen variations in how plugins are defined have evolved, ranging from manual declarations of `configs` to custom scripts to auto-generated JavaScript or TypeScript files.

This divergence in implementation details results in conceptual and practical overhead for plugin authors.
Without a standardize factory function to define a plugin:

- There is no first-party way for plugins to receive type hints during development the way [`eslint/config`'s `defineConfig`](https://eslint.org/docs/latest/use/configure/configuration-files#configuration-file) assists in authoring configs
- Plugin names are often defined in at least four separate places:
  - `configs[string].name`, such as `"example/recommended"`
  - `configs[string].rules`, such as `{ "example/rule": "error" }`
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

  - Optimization requests such as [typescript-eslint/typescript-eslint#11029 Enhancement: Support Lazy Loading Rules](https://github.com/typescript-eslint/typescript-eslint/issues/11029) must be implemented per-plugin

This RFC proposes creating a `definePlugin` function to be exported from `@eslint/plugin-kit`.
It would provide a single recommended factory for defining the structure of a typical ESLint plugin.
`definePlugin` would automate the parts of plugin definitions that many plugins manually redefine:

- Using the same name in multiple places
- Generating `configs.recommended` and other configurations based on rule metadata

It would also allow definition styles that allow internal optimizations.
Most notably, it would allow lazy-loading rules if provided as functions and manually listed in `configs`.
An `eslint-plugin-eslint-plugin` lint rule will be created that keeps `configs` and `rules` in sync.

As with `defineConfig`, `definePlugin` would be a purely optional, additive change.
Existing and future plugins would continue to work as-is without a `definePlugin` call.

## Detailed Design

The `@eslint/plugin-kit` package would export a `definePlugin` function that takes in a single required options object.
The options properties would require two properties:

- `meta`: The same `name` as [Meta Data in Plugins](https://eslint.org/docs/latest/extend/plugins#meta-data-in-plugins), along with an optional `version`
- `rules`: The same object as [Rules in Plugins](https://eslint.org/docs/latest/extend/plugins#rules-in-plugins), but also allowing functions that return a rule

It would return a plugin with:

- `configs`: by default, an object containing:
  - `recommended`: a config array of a single object containing:
    - `name`: per [configuration naming conventions](https://eslint.org/docs/latest/use/configure/configuration-files#configuration-naming-conventions)
    - `plugins`: an object assigning `meta.name` to the plugin
    - `rules`: all rules whose `meta.docs?.recommended === true`, with severity set to `"error"`
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

That definition would be the equivalent of:

```js
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
      rules: Object.from(
        Object.entries(rules)
          .filter(([, rule]) => rule.meta.docs?.recommended)
          .map(([ruleName]) => [`example/${ruleName}`, "error"]),
      ),
    },
  ],
});
```

`definePlugin` should allow plugin authors to avoid repeating the plugin's name or repeatedly referring to a `plugin` value.
No `Object.assign()`, `Object.defineProperty()`, or `get()` should be required in the making of an ESLint plugin.

### Configs

`configs` is the only property `definePlugin` changes from the current form of definitions.
Its properties are modified in the following ways:

- `name` will default to the plugin's name + `"/"` + the config's name
- `plugins` may not be provided; instead, `definePlugin` will add a `plugins` object property defining the plugin under its name
- `rules` must be provided as an array of rules, each of which is either:
  - A rule object included in the plugin's `rules`, in which case it will be turned into a namespace based on the plugin's `meta.name`
  - A tuple array containing a rule and a severity to specify that rule as
  - An object with string keys and rule severity values, which will be directly merged into the resultant config `rules`

Defining the `name` and `rules` from the earlier example plugin manually would look approximately like:

```js
const plugin = definePlugin({
  configs: {
    recommended: {
      name: "example/recommended",
      rules: rules.filter((rule) => rule.meta.docs?.recommended),
    },
  },
  meta,
  rules,
});
```

If any `config` value is provided as an array instead of an object, `name` and `plugins` will be added to its first element only.

#### Configs Examples

A plugin whose `meta.docs.recommended` is `'error' | 'warning'` could specify rule severities from that metadata:

```js
const plugin = definePlugin({
  configs: {
    recommended: {
      name: "example/recommended",
      rules: rules
        .filter((rule) => rule.meta.docs?.recommended)
        .map((rule) => [rule, rule.meta.docs.recommended]),
    },
  },
  meta,
  rules,
});
```

A plugin that defines additional properties such as `languageOptions` would also define them in its `configs.recommended`:

```js
definePlugin({
  configs: {
    recommended: {
      languageOptions: {
        globals: {
          myGlobal: "readonly",
        },
      },
      rules: rules.filter((rule) => rule.meta.docs?.recommended),
    },
  },
  meta,
  rules,
});
```

A plugin akin to `eslint-plugin-markdown` with multiple entries in its `recommended` config could override the `name` and `rules` of the first entry, leaving `rules` for the last:

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

A plugin equivalent to `eslint-plugin-storybook` that references rules from other plugins:

```js
definePlugin({
  configs: {
    recommended: {
      rules: [
        ...rules: rules.filter((rule) => rule.meta.docs?.recommended),
        {
          'import/no-anonymous-default-export': 'off',
          'react-hooks/rules-of-hooks': 'off',
        }
      ]
    }
  },
  meta,
  rules,
});
```

A plugin akin to `typescript-eslint` with `recommended`, `strict`, and `*TypeChecked` configs could define those four configs as functions:

```js
definePlugin({
  configs: {
    recommended: {
      rules: rules.filter((rule) => rule.meta.docs.recommended),
    },
    recommendedTypeChecked: {
      rules: rules.filter(
        (rule) =>
          rule.meta.docs.recommended && rule.meta.docs.requiresTypeChecking
      ),
    },
    strict: {
      rules: rules.filter((rule) => rule.meta.docs.recommended === "strict"),
    },
    strictTypeChecked: {
      rules: rules.filter(
        (rule) =>
          rule.meta.docs.recommended === "strict" &&
          rule.meta.docs.requiresTypeChecking
      ),
    },
  },
  meta,
  rules,
});
```

### Lazy-Loading Rules

[vuejs/eslint-plugin-vue#2732](https://github.com/vuejs/eslint-plugin-vue/issues/2732) and [typescript-eslint/typescript-eslint#11029 Enhancement: Support Lazy Loading Rules](https://github.com/typescript-eslint/typescript-eslint/issues/11029) ask that plugins lazy-load their rules the way the core ESLint configs are lazy-loaded.
Doing so would allow plugins with many dozens or more rules to avoid the cost of loading those rules unnecessarily.

`definePlugin` can enable lazy-loaded rules by allowing elements in `rules` arrays to each be provided as either:

- A rule itself
- A function that synchronously returns a rule

However, lazy-loading rules conflicts with auto-generating `configs`.
Knowing which rules are to be included in a config requires loading the rule itself.
`definePlugin` will throw an error if any rule function is provided and `configs` is not specified.

A plugin definition that uses lazy-loaded rules would look like:

```js
import { definePlugin } from "@eslint/plugin-kit";
import { createRequire } from "node:module";

import packageData from "../package.json" with { type: "json" };

const require = createRequire(import.meta.url);

export const plugin = definePlugin({
  configs: {
    recommended: [
      "a",
      "b",
      "c",
      // ...
    ],
  },
  meta: { name: packageData.name, version: packageData.version },
  rules: [
    () => require("./rules/a"),
    () => require("./rules/b"),
    () => require("./rules/c"),
    // ...
  ],
});
```

`definePlugin` would internally use a modified version of [ESLint core's `LazyLoadingRuleMap`](https://github.com/eslint/eslint/blob/129882d2fdb4e7f597ed78eeadd86377f3d6b078/lib/rules/utils/lazy-loading-rule-map.js) to only load rules as necessary.
For reference: [ESLint core implements lazy-loaded configs with a `LazyLoadingRuleMap` and `Proxy`](https://github.com/eslint/eslint/blob/129882d2fdb4e7f597ed78eeadd86377f3d6b078/lib/config/default-config.js#L46):

```js
exports.defaultConfig = Object.freeze([
  {
    plugins: {
      "@": {
        languages: {
          js: require("../languages/js"),
        },

        /*
         * Because we try to delay loading rules until absolutely
         * necessary, a proxy allows us to hook into the lazy-loading
         * aspect of the rules map while still keeping all of the
         * relevant configuration inside of the config array.
         */
        rules: new Proxy(
          {},
          {
            get(target, property) {
              return Rules.get(property);
            },

            has(target, property) {
              return Rules.has(property);
            },
          }
        ),
      },
    },
    // ...
  },
  // ...
]);
```

ESLint core today is implemented in CommonJS, which allows it to [define core plugin rules with the global `require()`](https://github.com/eslint/eslint/blob/129882d2fdb4e7f597ed78eeadd86377f3d6b078/lib/rules/index.js#L14-L16):

```js
module.exports = new LazyLoadingRuleMap(
  Object.entries({
    "accessor-pairs": () => require("./accessor-pairs"),
    // ...
  })
);
```

`definePlugin` will use similar strategies internally.

An `eslint-plugin-eslint-plugin` lint rule will also be created that enforces lazy-loading rules.
It will come with an auto-fixer that:

- Adds the requisite `createRequire()` and `require()` calls to the `definePlugin` file
- Keeps `configs` in sync with rule `meta.docs.recommended` values corresponding to existing common community conventions:
  - falsy: only in an `all` config, if it exists
  - `string | string[]`: in config(s) of the same name(s)
  - `true`: in a `recommended` config

Note that this lint rule would need to inspect each rule file to determine rules' `meta.docs.recommended` values.

Running the lint rule with `--fix` would accomplish the same automation work a many common community "update" scripts under [Design Pattern Analysis](#design-pattern-analysis).

## Documentation

- The README.md for [`@eslint/plugin-kit`](https://www.npmjs.com/package/@eslint/plugin-kit) will have a section added for `definePlugin`
- The existing [Create Plugins](https://eslint.org/docs/latest/extend/plugins) page will be updated to use the new helper
  - It can also link to the `@eslint/plugin-kit` README.md section for expanded documentation
  - It will also explicitly note that the helper is optional

## Drawbacks

Adding one more item to learn for plugins increases the barrier to entry to authoring plugins.
Developers may not want a helper that they need to learn alongside the existing concepts.
Some might find that obfuscating plugin definitions in this way is conceptual overhead costlier than the current act of writing portions manually.

Adding a first-party standardized config also runs the risk of forcing the community into patterns it does not want.
Authors may find the `definePlugin`-style `configs` cumbersome, for example, and might opt not to use the helper factory as a result.
If a super-majority of plugin authors don't onboard to `definePlugin` then this RFC might lead to an increase in plugin definition styles.

This RFC believes that the net benefits of a first-party `definePlugin` outweigh the risks.
Given how many existing plugins seem to adhere to similar patterns, it is likely that this RFC will net reduce, not increase, implementation divergence.

## Backwards Compatibility Analysis

Fully backward compatible.
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

> Note: this summary table was hand-authored, although it was double-checked, it is susceptible to human error.
> This RFC believes slight inaccuracies in the table do not invalidate its general conclusion: that a most plugins would benefit from a `definePlugin`.

## Alternatives

It might be preferable to release `definePlugin` as a community-authored plugin before standardizing it in `@eslint/plugin-kit`.
Doing so could allow for more rapid changes as community plugins adopt it.
This RFC prefers making the function first-party for visibility and to encourage confidence in plugin author adoption.

## Open Questions

### Config Arrays

`eslint-plugin-markdown` defines an unusual `plugin.configs.recommended`:

- An array of config objects instead of just one
- `name` with `/plugin` at the end
- `rules` in the last config object rather than the first

Is it acceptable to change `eslint-plugin-markdown`'s implementation to what's suggested under _[Configs Examples](#configs-examples)_?

### Legacy Configs

Many plugins include `legacy-*` configs as suggested in [Backwards Compatibility for Legacy Configs](https://eslint.org/docs/latest/extend/plugins#backwards-compatibility-for-legacy-configs).
However, configs that don't want adopting `definePlugin` to be a breaking change would need those configs to stay in existence.

Plugin developers can work around that lack of support by manually adding those config properties:

```js
const plugin = definePlugin({
  meta,
  rules,
});

// TODO: Remove when we drop support for ESLint <=9 / eslintrc
plugin.configs["legacy-recommended"] = {
  plugin: ["example"],
  rules: Object.fromEntries(
    Object.entries(rules)
      .filter(([, rule]) => rule.meta.docs?.recommended)
      .map(([name]) => ["example/" + name, "error"])
  ),
};
```

Is this an acceptable workaround strategy?

### Rule Loading

Rules are allowed to be directly provided or lazy-loaded.
However, knowing that lazy-loading is possible -and determining when to make the switch- is an extra challenge.

This RFC expects:

- Allowing directly passing rules benefits plugin authors by making the initial scaffolding for plugins as simple as possible
- Most plugins will start small and not benefit from lazy-loading
- The plugins that scale to the point of benefiting from lazy-loading are likely to enable `eslint-plugin-eslint-plugin`

Should plugins be required to always provide lazy-loading rule functions?

### Rule Creators

This RFC's scope intentionally does include a "rule creator" factory akin to [`@typescript-eslint/rule-creator`](https://typescript-eslint.io/developers/custom-rules/#rulecreator).
Creating such a factory in ESLint could be useful but is not required for any of the problems `definePlugin` aims to solve.

## Help Needed

I would love to implement this RFC.

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
As a result, it's also a more complex system to model in type system types for `definePlugin`.
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

It's used to generate keys in `configs`.

### `rules`: Could plugins be forced to enable lazy-loading?

Or: why allow `rules` values to be either the rule or a function, rather than always the function form?

Requiring getter functions is mildly inconvenient.
Not being able to automatically generate `configs` is more inconvenient.

Most community plugins load fewer than a dozen rules and enable most or all of those rules in their `recommended` configs.
For those plugins, lazy-loading rules would be added authoring overhead for little-to-no-gain.

This RFC believes that allowing automatic `configs` generation and the more succinct inline `rules` is worth the complexity and performance tradeoffs.

### `rules`: Could rules be provided as string paths to be loaded?

Or: instead of `() => require('./rules/a')`, could just `'./rules/a'` be provided?

This could technically be supported.
However:

- `configs` would still need to load the rules to be automatically generated, so string rule paths would not allow automatic configs with lazy-loaded rules
- Third-party tools such as bundlers would not be able to easily perform analyzing techniques such as tree-shaking on those "magical" string paths

## Related Discussions

- [eslint#14862 Support Lazy Loading Rules from 3rd Party Plugins](https://github.com/eslint/eslint/issues/14862)
- [typescript-eslint/typescript-eslint#11029 Enhancement: Support Lazy Loading Rules](https://github.com/typescript-eslint/typescript-eslint/issues/11029)
- [vuejs/eslint-plugin-vue#2732](https://github.com/vuejs/eslint-plugin-vue/issues/2732)
- [typescript-eslint/typescript-eslint#10383 Enhancement: Move RuleCreator into its own package with fewer dependencies than utils](https://github.com/typescript-eslint/typescript-eslint/issues/10383)
