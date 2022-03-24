- Repo: eslint/eslint
- Start Date: (2022-02-28)
- RFC PR: https://github.com/eslint/rfcs/pull/88
- Authors: aladdin-add(weiran.zsd@outlook.com)

# convert eslint codebase to esm

## Summary

<!-- One-paragraph explanation of the feature. -->
Convert the ESLint codebase to ESM and publish the ESM version.
## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->
The Node.js community has been written commonjs for a long time, the same goes for ESLint. But with ESM introduced in ES6 and officially supported by Node.js v12, more and more projects are moving to ESM. Not long ago, we had converted a few repos under the ESLint organization to ESM:
* @eslint/eslintrc
* espree
* eslint-scope
* eslint-visitor-keys
* @eslint/create-config

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->
There are two main ways to achieve this:
1. esm-only: write esm, publish esm
```js
{
    "type": "module",
    "exports": {
        "./package.json": "./package.json",
        ".": "./lib/api.js",
        "./use-at-your-own-risk": "./lib/unsupported-api.js"
    }
}
```
2. dual-mode: compile esm to cjs, publish esm+cjs
```js
    "type": "module",
    "exports": {
        "./package.json": "./package.json",
        ".": {
            "import": "./lib/api.js",
            "require": "./dist/api.cjs",
            "default": "./dist/api.cjs"
        },
        "./use-at-your-own-risk": {
            "import": "./lib/unsupported-api.js",
            "require": "./dist/unsupported-api.cjs",
        }
    }
```

Node.js ESM has a limitation: ESM cannot be required. Approach 2 provides a good compatibility for existing projects that rely on ESLint without requiring users to adopt ESM as well. In previous repos, dual-mode was mostly used (except for '@eslint/create-config').

However, dual-mode also has a few downsides:
1. dependencies. Some ESLint's dependencies have been (or will be) esm-only, which means we won't be able to upgrade them if we continue to support CJS. For new dependencies, we cannot use them if they are esm-only.
2. Slower installation & larger volume usage, even though most users only use ESM, they still need to download CJS modules.
3. Higher maintenance costs. You need to add a build process, test and publish ESM/CJS code.
4. The CLIs do not support dual-mode. If the eslint bin is esm, it has no use compiling to cjs - as no one would require it.

In the (very) long run, I would suggest that we eventually go with esm-only, but to give users more time to upgrade, as an intermediate state, we could use dual-mode in ESLint v9:

* eslint v9:
    - ESLint CLI: esm-only(`./bin/eslint.js` will be esm, and won't be compiled to cjs.)
    - Node.js API: dual mode
* eslint 1x (at the point when the majority of the ecosystem is ready to support it, likely to be v10/v11/...):
    - ESLint CLI: esm-only
    - Node.js API: esm-only

## Toolings esm supports(ESLint's dependencies)

* mocha ✅
* proxyquire ❌
* nyc ❌
* eslint-plugin-node ❗️(Not fully supported, but we could use a fork: `eslint-plugin-n`)

## Implementations
1. ESLint v9
* package.json + `"type": "module"`
* package.json exports
* package.json engines >= Node.js v12 (has been shipped in ESLint v8.0)
* remove `"use strict"`
* convert `"./foo"` to `"./foo/index.js"`
* convert `"./foo"` to `"./foo.js"`
* convert top-level `require/module.exports` => `import/export`
* convert non-top-level `require/module.exports` => `import()` (TODO: it requires the wrapping functions to be async, need to check if any public APIs need to be changed).
* convert `__dirname/__filename` => `import.meta.url`. There are ~75 usages of __dirname that will need to be dealt with. [#what-do-i-use-instead-of-__dirname-and-__filename](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c#what-do-i-use-instead-of-__dirname-and-__filename)
* esm toolings(`nyc` -> `c8`, `proxyquire` -> `esmock`)
* cjs supports(build `dist/api.cjs` and `dist/use-at-your-own-risk.cjs` using rollup)
* update docs to use esm

2. ESLint 1x
* remove package.json exports.cjs (use esm-only)
* remove build process
* upgrade blocked deps.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
Yes. When ESLint v9 is released, it is recommended to announce that users should upgrade to ESM, CJS may be no longer supported in future major versions.

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
1. Compatibility issues, see below.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

1. For users of the ESLint CLI: there is no impact and users can write configuration files using ESM/CJS.
2. For users of the Node.js API:
* ESLint v9: Will support ESM/CJS, so there will be no compatibility issues for users.
* ESLint 1x: CJS will no longer be supported. At this point developers can:

a. convert to esm.
```diff
- var { ESLint } = require("eslint");
+ import { ESLint } from "eslint";
```

b. move to async.
```diff
- var { ESLint } = require("eslint");
+ var { ESLint } = await import("eslint");
```

c. compile ESLint to CJS.

d. stay previous version util they can convert to esm.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->
1. esm-only in eslint v9？
TBD.

2. cjs only??
TBD.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->
1. Do we want to bundle ESM/CJS, or just compile ESM => CJS?

To be consistent with other eslint repos, cjs will be bundled to `dist/`, while esm will not.

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->
1. helpful toolings for the converting:
* `'import/extensions': ['error', 'always']`, (enforce file extensions) https://github.com/import-js/eslint-plugin-import/blob/main/docs/rules/extensions.md
* `'unicorn/prefer-module': 'error'` (enforces a lot of ESM syntax / disallows CJS syntax) https://github.com/sindresorhus/eslint-plugin-unicorn/blob/main/docs/rules/prefer-module.md


2. helpful (but opinionated) definitive guides for converting to esm-only:
* https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c
## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
* https://github.com/eslint/eslint/issues/15560
* https://github.com/eslint/rfcs/pull/72
