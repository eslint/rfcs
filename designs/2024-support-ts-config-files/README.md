- Repo: eslint/eslint
- Start Date: 2024-03-09
- RFC PR: <https://github.com/eslint/rfcs/pull/117>
- Authors: [Arya Emami](https://github.com/aryaemami59)

# Add Support for TypeScript Config Files

## Summary

Add support for TypeScript config files (`eslint.config.ts`, `eslint.config.mts`, `eslint.config.cts`)

<!-- One-paragraph explanation of the feature. -->

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

The primary motivation for adding support for TypeScript configuration files to ESLint is to enhance the developer experience and accommodate the evolving JavaScript ecosystem. As TypeScript's popularity continues to grow, more projects are adopting TypeScript not only for their source code but also for their configuration files. This shift is driven by TypeScript's ability to provide compile-time type checks and IntelliSense. By supporting `eslint.config.ts`, `eslint.config.mts`, and `eslint.config.cts`, ESLint will offer first-class support to TypeScript users, allowing them to leverage these benefits directly within their ESLint configuration.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

The goal is to seamlessly support TypeScript configuration files in ESLint. To achieve this, ESLint will need to recognize and parse TypeScript configuration files in the same way it does for JavaScript configuration files. This will involve creating the configuration file resolution logic to recognize `.ts`, `.mts`, and `.cts` files as valid ESLint configuration files. We will need to treat `eslint.config.mts` files as ECMAScript Modules (ESM) and `eslint.config.cts` files as CommonJS (CJS). The module resolution for `eslint.config.ts` files will depend on the `type` field in the nearest `package.json`. The maintainers of ESLint have raised some valid concerns some of which include:

- There should not be extra overhead for JavaScript users. This means this change should not have a significant impact (if any at all) affecting users who use plain JavaScript config files.
- The external tools that are used to parse the config files written in TypeScript should not create side effects. Specifically, it is imperative that these tools do not interfere with Node.js's native module resolution system by hooking into or altering the standard `import/require` mechanisms. This means tools like [`ts-node`](https://github.com/TypeStrong/ts-node) and [`tsx`](https://github.com/privatenumber/tsx) might not be suitable for this purpose.

So far the tool that seems to be the most suitable for this purpose is [`jiti`](https://www.npmjs.com/package/jiti). It does not introduce side effects and performs well, demonstrating its reliability. It also seems to be more battle-tested given some established frameworks such as [Nuxt](https://github.com/nuxt/nuxt), [Tailwind CSS](https://github.com/tailwindlabs/tailwindcss) and [Docusaurus](https://github.com/facebook/docusaurus) have been using it to load their configuration files.

## Examples

with `eslint.config.mts` file:

```ts
import eslint from "@eslint/js"
import type { Linter } from "eslint"

const config: Linter.FlatConfig[] = [
  eslint.configs.recommended,
  {
    rules: {
      "no-console": [0],
    },
  },
]

export default config
```

with `eslint.config.cts` file:

```ts
import type { Linter } from "eslint"
const eslint = require("@eslint/js")

const config: Linter.FlatConfig[] = [
  eslint.configs.recommended,
  {
    rules: {
      "no-console": [0],
    },
  },
]

module.exports = config
```

with `eslint.config.ts` file:

```ts
import eslint from "@eslint/js"
import type { Linter } from "eslint"

const config: Linter.FlatConfig[] = [
  eslint.configs.recommended,
  {
    rules: {
      "no-console": [0],
    },
  },
]

export default config
```

It is worth noting that you can already perform some type-checking with the [`checkJs`](https://www.typescriptlang.org/tsconfig#checkJs) TypeScript option and a JavaScript config file. Here's an example:

with `eslint.config.mjs` or (`eslint.config.js` + `"type": "module"` in the nearest `package.json`):

```js
import eslint from "@eslint/js"

/** @type {import('eslint').Linter.FlatConfig[]} */
const config = [
  eslint.configs.recommended,
  {
    rules: {
      "no-console": [0],
    },
  },
]

export default config
```

with `eslint.config.cjs` or (`eslint.config.js` without `"type": "module"` in the nearest `package.json`):

```js
const eslint = require("@eslint/js")

/** @type {import('eslint').Linter.FlatConfig[]} */
const config = [
  eslint.configs.recommended,
  {
    rules: {
      "no-console": [0],
    },
  },
]

module.exports = config
```

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The documentation for this feature will be added to the [ESLint User Guide](https://eslint.org/docs/user-guide/configuring) page. The documentation will explain how to use TypeScript configuration files and the differences between JavaScript and TypeScript configuration files.

## Drawbacks

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think
    about any opposing viewpoints that reviewers might bring up.

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

This change will most likely require at least one external tool as either a dependency or a peer dependency.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This goal is to minimize the disruption to existing users. The primary focus is to ensure that the existing JavaScript configuration files continue to work as expected. The changes will only affect TypeScript users who are using TypeScript configuration files. The changes will not affect the existing JavaScript configuration files.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

While developing this feature, we considered the following alternatives:

1. Using [`ts-node`](https://github.com/TypeStrong/ts-node) to parse TypeScript configuration files. This approach proved to be problematic because [`ts-node`](https://github.com/TypeStrong/ts-node) hooks into Node.js's native module resolution system, which could potentially cause side effects.

2. Using [`tsx`](https://github.com/privatenumber/tsx) to parse TypeScript configuration files. This approach also proved to be problematic because [`tsx`](https://github.com/privatenumber/tsx) hooks into Node.js's native module resolution system, which could potentially cause side effects.

3. Using [TypeScript's `transpileModule()`](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API#a-simple-transform-function) to parse TypeScript configuration files. This approach proved to be problematic because it requires a significant amount of overhead and is not suitable for this purpose.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

1. How is caching going to work with TypeScript config files? We only cache the computed result of loading a config file, so I don't think this should be a problem.
2. Should we look at the nearest `tsconfig.json` file to determine the module resolution for `eslint.config.ts` files? Most likely not, but it's worth considering.
3. Should we allow some sort of interoperability between JavaScript and TypeScript configuration files? For example, should we allow a TypeScript configuration file to extend a JavaScript configuration file and vice versa?
4. Should we allow `eslint.config.ts` to be able to use `export default` as well as `module.exports` (might be related to [TypeScript's automatic Module Detection](https://www.typescriptlang.org/tsconfig#moduleDetection))?
5. Tools like [Vitest](https://github.com/vitest-dev/vitest) export a [`defineConfig`](https://vitest.dev/config/file.html#managing-vitest-config-file) function to make it easier to write configuration files in TypeScript. Should we consider doing something similar for ESLint?
6. How does the feature interact with the [CLI option](https://eslint.org/docs/latest/use/command-line-interface#options) [`--config`](https://eslint.org/docs/latest/use/command-line-interface#-c---config) for specifying a config file? It doesn't behave any differently, same as before. You can do `eslint . --config=eslint.config.ts` or `eslint . -c eslint.config.ts` and they just work. Same as with a `eslint.config.js` file.

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I will be implementing this feature. I will need help from the team to review the code and provide feedback.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- [This PR](https://github.com/eslint/eslint/pull/18134) is related to this RFC.
- [Prior Discussion](https://github.com/eslint/rfcs/pull/50) related to supporting `.eslintrc.ts` files.
- [Prior Issue](https://github.com/eslint/eslint/issues/12078) related to supporting `.eslintrc.ts` files.

## External References

- [`jiti` on NPM](https://www.npmjs.com/package/jiti)
- [`jiti` on Github](https://github.com/unjs/jiti)
- [`tsx` on NPM](https://www.npmjs.com/package/tsx)
- [`tsx` on Github](https://github.com/privatenumber/tsx)
- [`ts-node` on NPM](https://www.npmjs.com/package/ts-node)
- [`ts-node` on Github](https://github.com/TypeStrong/ts-node)
- [`ts-node` Docs](https://typestrong.org/ts-node)
- [TypeScript on NPM](https://www.npmjs.com/package/typescript)
- [TypeScript on Github](https://github.com/Microsoft/TypeScript)
- [TypeScript docs](https://www.typescriptlang.org)
- [TypeScript's `transpileModule()`](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API#a-simple-transform-function)
- [Vitest on Github](https://github.com/vitest-dev/vitest)
- [Vitest on NPM](https://www.npmjs.com/package/vitest)
- [Vitest Docs](https://vitest.dev)
- [Docusaurus on Github](https://github.com/facebook/docusaurus)
- [Docusaurus on NPM](https://www.npmjs.com/package/docusaurus)
- [Docusaurus Docs](https://docusaurus.io)