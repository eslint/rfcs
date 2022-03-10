- Repo: eslint/eslint
- Start Date: (2022-02-28)
- RFC PR: (leave this empty, to be filled in later)
- Authors: aladdin-add(weiran.zsd@outlook.com)

# convert eslint codebase to esm

## Summary

<!-- One-paragraph explanation of the feature. -->
将eslint仓库中的代码转换为ESM，并且发布ESM版本。

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->
长久以来，Node.js社区都是采用CJS规范来编写代码，ESLint也是一样。但ES6中引入了esm，Node.js v12 也已经正式支持了它，因此越来越多的项目也逐渐在转向ESM。不久前，我们已经将ESLint组织下的一些仓库转换为了ESM：
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
主要有2种方式来实现：
1. 编写esm，发布esm
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
2. 编写esm，编译为cjs，发布esm+cjs
```js
    "type": "module",
    "exports": {
        "./package.json": "./package.json",
        ".": {
            "import": "./lib/api.js",
            "require": "./dist/api.cjs"
        },
        "./use-at-your-own-risk": {
            "import": "./lib/unsupported-api.js",
            "require": "./dist/unsupported-api.cjs",
        }
    }
```

Node.js 的esm有一个限制：在cjs中不能import esm。方式2对于现有依赖ESLint的项目来说提供了良好的兼容性，而不必要求使用者也采用ESM。在以前的几个仓库中，主要都采用方式2（除了`@eslint/create-config`采用方式1）。

但是这种方式也有很多不足：
1. 依赖升级。一些eslint依赖的新版本使用了ESM-only，这意味着如果要继续支持cjs，我们将不能升级它们。对于新的依赖，如果是esm-only，我们也不能够使用。
2. 更慢的安装速度&更大的安装体积，即使大多数用户只用到了ESM，仍然需要下载CJS模块。
3. 更高的维护成本。需要增加编译流程，测试和发布esm/cjs代码。
4. CLI 不支持 dual mode。

因此，我建议最终我们可以采用esm-only，但为了给用户更多的升级时间，作为一个中间状态，我们可以在ESLint v9中采用 dual-mode：

* eslint v9:
    - ESLint CLI: esm only
    - Node.js API: dual mode, 
* eslint v10:
    - ESLint CLI: esm only
    - Node.js API: esm only 

## Toolings esm supports

* mocha ✅
* proxyquire ❌
* nyc ❌
* eslint-plugin-node ❌(不完全支持，但我们可以使用一个fork: eslint-plugin-n)

## 实现步骤
1. ESLint v9
- package.json + "type": "module"
- package.json exports
- package.json engines >= Node.js v12
- update docs to use esm
- remove 'use strict'
- convert './xxx' to './xxx/index.js'
- convert './xxx' to './xxx.js'
- esm toolings(nyc -> c8, proxyquire -> esmock)
- cjs supports(build dist/api.js using rollup)

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
是的。在eslint v9发布时，推荐用户升级到esm，cjs将在v10中不再支持。

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
1. 版本兼容性问题。

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->
1. 对于 ESLint CLI的使用者，不会有影响，用户可以使用 ESM/CJS 编写配置文件（如果）。
2. 对于 Node.js API的使用者：
* ESLint v9: 将支持 ESM/CJS，因此对使用者来说，不会有兼容性问题。
* ESLint v10: 对于ESLint Node.js API的使用者，将不再支持CJS。此时开发者可以：
a. 切换至ESM。
```diff
- var { ESLint } = require("eslint");
+ import { ESLint } from "eslint";
```
b. 改为异步代码
```diff
- var { ESLint } = require("eslint");
+ var { ESLint } = await import("eslint");
```
c. 用 babel/tsc/... 将 ESLint编译为CJS。
d. 停留在ESLint v9。

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->
1. esm-only in eslint v9？

2. dual-mode??

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->
1. 我们是否想要打包esm/cjs，还是仅仅编译esm=>cjs?
与espree等其它仓库一样，对于esm不打包，cjs打包到dist/。

2. eslint有一些devdeps也在依赖eslint，然而它们还没有支持esm，这将导致测试报错？？

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

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
* https://github.com/eslint/eslint/issues/15560
* https://github.com/eslint/rfcs/pull/72
