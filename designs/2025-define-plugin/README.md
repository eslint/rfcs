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
`definePlugin` would provide three benefits:

- Throwing an error when expected fields do not exist or are an invalid type
- Populating the `plugins` field on configs, so that a plugin can be defined within a single object literal
- Types-based editing support in editors that support types

As with `defineConfig`, `definePlugin` would be a purely optional, additive change.
Existing and future plugins would continue to work as-is without a `definePlugin` call.

## Detailed Design

The `@eslint/plugin-kit` package would export a `definePlugin` function that takes in a single required options object.
The options object would allow authors to specify all the same properties as described in [Create Plugins > Creating a plugin](https://eslint.org/docs/latest/extend/plugins#creating-a-plugin).
Out of those provided properties:

- `configs` will be modified as described in [`Configs`](#configs).
- All other properties will be passed through without modification.

The only required property will be `meta.name`, which must be set to a `string`.

The following code block creates a standard plugin with metadata and rules:

```js
import { definePlugin } from "@eslint/plugin-kit";

import ruleA from "./ruleA.js";
import ruleB from "./ruleB.js";

export const plugin = definePlugin({
  meta: {
    name: "example",
    version: 0.1.2,
  },
  rules: {
    "a": ruleA,
    "b": ruleB,
  },
});
```

That definition would be the equivalent of:

```js
// (equivalent to the previous snippet)
import ruleA from "./ruleA.js";
import ruleB from "./ruleB.js";

export const plugin = {
  meta: {
    name: "example",
    version: 0.1.2,
  },
  rules: {
    "a": ruleA,
    "b": ruleB,
  },
};
```

Note that there is no difference between the object literal provided to `definePlugin` in the first snippet compared to the equivalent object literal set to `plugin` in the second snippet.

### Configs

An optional `configs` object may be provided to `definePlugin`.
If provided, each property on it be merged with a `plugins` object specifying the current plugin by its `meta.name`.

For example, given this common definition style of a plugin with configs, metadata, and rules:

```js
import { definePlugin } from "@eslint/plugin-kit";

import ruleA from "./ruleA.js";
import ruleB from "./ruleB.js";

export const plugin = definePlugin({
  configs: {
    recommended: {
      rules: {
        "example/a": "error",
        "example/b": "error",
      },
    },
  },
  meta: {
    name: "example",
  },
  rules: {
    a: ruleA,
    b: ruleB,
  },
});
```

That definition would be the equivalent of:

```js
// (equivalent to the previous snippet)
import ruleA from "./ruleA.js";
import ruleB from "./ruleB.js";

export const plugin = {
  meta: {
    name: "example",
    version: 0.1.2,
  },
  rules: {
    "a": ruleA,
    "b": ruleB,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      plugins: {
        example: plugin,
      },
      rules: {
        "example/a": ruleA,
        "example/b": ruleA,
      },
    },
  ],
});
```

#### Configs with Arrays

If a plugin provides multiple objects in a `configs` array, only the first object will have `name` and `plugins` added in as defaults.
For example, this plugin is set up akin to the current `eslint-plugin-markdown`.
It provides multiple entries in its `recommended` config and only specifies `rules` in the last element:

```js
import { definePlugin } from "@eslint/plugin-kit";

import ruleA from "./ruleA.js";
import ruleB from "./ruleB.js";

export const plugin = definePlugin({
  configs: {
    recommended: [
      {
        name: "example/recommended/plugin",
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
          "example/a": "error",
          "example/b": "error",
        },
      },
    ],
  },
  meta: {
    name: "example",
  },
  processors: {
    markdown: processor,
  },
  rules: {
    a: ruleA,
    b: ruleB,
  },
});
```

The equivalent generated plugin object would be:

```js
// (equivalent to the previous snippet)
import ruleA from "./ruleA.js";
import ruleB from "./ruleB.js";

const plugin = {
  configs: {},
  meta: {
    name: "example",
  },
  processors: {
    markdown: processor,
  },
  rules: {
    a: ruleA,
    b: ruleB,
  },
};

Object.assign(plugin.configs, {
  recommended: [
    {
      name: "example/recommended/plugin",
      plugins: {
        example: plugin,
      },
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
        "example/a": "error",
        "example/b": "error",
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
Authors may find the `definePlugin` function cumbersome, for example, and might opt not to use the helper factory as a result.
If a super-majority of plugin authors don't onboard to `definePlugin` then this RFC might lead to an increase in plugin definition styles.

This RFC believes that the net benefits of a first-party `definePlugin` outweigh the risks.
Given how many existing plugins seem to adhere to similar patterns, it is likely that this RFC will net reduce, not increase, implementation divergence.

## Backwards Compatibility Analysis

Fully backward compatible with no breaking changes.
The new `definePlugin` would be opt-in and optional.

## Alternatives

It might be preferable to release `definePlugin` as a community-authored plugin before standardizing it in `@eslint/plugin-kit`.
Doing so could allow for more rapid changes as community plugins adopt it.
This RFC prefers making the function first-party for visibility and to encourage confidence in plugin author adoption.

### Config Generation

This RFC originally included significant added logic around generating `configs` from different forms of input.
The goal was to allow authors to define plugin configs without referencing the plugin name in multiple places.
For example, a plugin might be allowed to provide an array of rules for a config:

```js
export const examplePlugin = definePlugin({
  configs: {
    recommended: [ruleA, ruleB],
  },
  meta: {
    name: "example",
  },
  rules: {
    a: ruleA,
    b: ruleB,
  },
});
```

The `configs` generation feature was removed from the RFC to simplify its initial implementation.
This RFC now treats `configs` generation as out of scope.

See [112c47 > 2025-define-plugin/README.md](https://github.com/eslint/rfcs/blob/112c474a8d3af98359d5611dd7b0c8ce597f85bd/designs/2025-define-plugin/README.md?plain=1) for the latest version of the RFC that proposed `configs` generation.

### Lazy-Loading Rules

This RFC previously suggested providing features and lint rules around lazy-loading rules.
Doing so would allow plugins to reduce their startup times.
However, lazy-loading requires a non-trivial increase in design complexity.
This RFC now treats lazy-loading as out of scope.

See [5d2557 > 2025-define-plugin/README.md#L705](https://github.com/eslint/rfcs/blob/5d2557787574e85c0e9413ba8cfa75f9215673b8/designs/2025-define-plugin/README.md?plain=1#L705) for the latest version of the RFC that proposed lazy-loading rules.

## Help Needed

I would love to implement this RFC.

I can also:

- Post this RFC in relevant Discords to draw plugin author attention to it
- Work with `eslint-plugin-eslint-plugin` on feature requests for proposed lint rules as part of this RFC
- Send draft PRs to community plugins to help onboard them to the new `definePlugin`

### Plugin Testing

`definePlugin` should be tested before release on at least the plugins defined in [2024-repo-ecosystem-plugin-tests](../2024-repo-ecosystem-plugin-tests/README.md).
This will ensure coverage of at least the following cases:

- `.js` files with `.d.ts` siblings
  - Test cases should be added to ensure [eslint/markdown#402 bug: `exactOptionalPropertyTypes` causes type errors when using plugins that were not built with this option](https://github.com/eslint/markdown/issues/402) is fixed and does not regress
- `.ts` files

If other plugin authors would like to test it out, that would be appreciated and helpful.

## Frequently Asked Questions

### `meta`: Why is `meta.name` required?

It's used to populate the `plugins` object in `configs[string][0]` objects.

## Related Discussions

- [vuejs/eslint-plugin-vue#2732](https://github.com/vuejs/eslint-plugin-vue/issues/2732)
- [typescript-eslint/typescript-eslint#10383 Enhancement: Move RuleCreator into its own package with fewer dependencies than utils](https://github.com/typescript-eslint/typescript-eslint/issues/10383)
