- Repo: [eslint/eslint](https://github.com/eslint/eslint)
- Start Date: 2026-01-14
- RFC PR: [eslint/rfcs#141](https://github.com/eslint/rfcs/pull/141)
- Authors: [Kyle Hensel](https://github.com/k-yle)

# Support Localisation of Rule Messages

## 1. Summary

<!-- One-paragraph explanation of the feature. -->

This RFC proposes adding localisation/internationalisation support for rule messages. This allows rules to define messages in multiple langauges, not just English.
Translations could be defined by plugin authors, or in a third-party package, similar to [the `@types/*` ecosystem](http://github.com/definitelyTyped/DefinitelyTyped).

For example, instead of:

```handlebars
{{left}} can be removed because it is already included by {{right}}
```

Someone who prefers Japanese would see:

<div lang="ja">

```handlebars
{{left}}は既に{{right}}に含まれているため、不要です。
```

</div>

<sub>(example from [eslint-plugin-regexp](https://ota-meshi.github.io/eslint-plugin-regexp/rules/optimal-quantifier-concatenation.html))</sub>

## 2. Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Internationalisation would make Eslint more accessible to non-English speakers.
We're already seeing [eslint plugins using non-English languages](https://github.com/LLLaiYoung/eslint-plugin-aegis/blob/9780cd87240e2bbe0c134b97b8776782fe9016e1/lib/rules/no-implicit-complex-object.js#L28) in messages, and there have already been attempts to [translate core rules](https://github.com/mysticatea/eslint-plugin-ja).

In some regions, learning English is [seen as](https://www.hanselman.com/blog/do-you-have-to-know-english-to-be-a-programmer) a [prerequisite](https://www.youtube.com/watch?v=fsn6buk-BtE) for [learning](https://www.reddit.com/r/learnprogramming/comments/tpyvn3/comment/i2go8sc/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button) to [code](https://antirez.com/news/61).
In other regions, it's [considered](https://www.quora.com/Is-it-true-that-Chinese-and-Russian-programmers-generally-dont-speak-English-How-do-they-then-name-variables-and-write-comments-given-that-in-most-compilers-they-have-to-do-it-in-Latin-alphabet) normal [to write](https://temochka.com/blog/posts/2017/06/28/the-language-of-programming.html) code [with only](https://www.dywt.com.cn) a [minimal](https://pg.ucsd.edu/publications/non-native-english-speakers-learning-programming_CHI-2018.pdf) understanding of English.

It it possible to build JavaScript products with minimal English knowledge, since many tools are already translated, including [VSCode](https://code.visualstudio.com/docs/configure/locales), [browser de](https://learn.microsoft.com/en-us/microsoft-edge/devtools/customize/localization)[v-tools](https://developer.chrome.com/docs/devtools/customize#language), the [TypeScript compiler](https://github.com/microsoft/TypeScript/blob/dba6c9b824852d3333f2314eca7ad995935227f4/src/loc/lcl/deu/diagnosticMessages/diagnosticMessages.generated.json.lcl#L37), etc.

[Vue](https://github.com/vuejs) is a great example where [translated docs](https://cn.vuejs.org) (amoung other things) have helped Vue to [foster a community](https://www.reddit.com/r/vuejs/comments/8dhguz/why_are_there_so_many_chinese_vue_ui_frameworks/) of non-English users, and [gain traction](https://www.quora.com/Why-is-Vue-js-widely-used-in-Asia-but-not-in-the-United-States) in places [like China](https://jiyinyiyong.medium.com/the-chinese-part-of-react-community-f783c5e2d0f9), clearly showing the benefit of internationalisation.

## 3. Detailed Design

It would be possible to translate messages for core rules (like `@eslint/js`) and rules from plugins.

There would be two ways of defining translations, inspired by the [DefinitelyTyped (`@types/*`)](http://github.com/DefinitelyTyped/DefinitelyTyped) ecosystem.

### 3.1. Case 1: The plugin wants to maintain its own translations

If a plugin author wants to maintain translations themselves, they can define these translations in the rule's code, alongside the existing English messages:

```diff
  module.exports = {
    meta: {
      messages: {
        avoidName: 'Avoid using variables named {{name}}',
      },
+     messageTranslations: {
+       'es': {
+         avoidName: 'Evite utilizar variables llamadas {{name}}',
+       },
+       'zh-Hans': {
+         avoidName: '避免使用名为{{name}}的变量',
+       },
+     },
    },
    create: (context) => ({
      Identifier(node) {
        if (node.name == 'foo') {
          context.report({ node, messageId: 'avoidName', data: { name: node.name } });
        }
      },
    }),
  };
```

This would work for messages used by `context.report()`, and for messages used by [suggestions](https://eslint.org/docs/latest/extend/custom-rules#suggestion-messageids).

### 3.2. Case 2: The plugin does NOT want to maintain translations

Plugin authors may not want to maintain translations themselves.
This is quite reasonable, maintaining translations increases the workload for maintainers, and can be difficult to review without other experts in each language.

This is a similar situation to JavaScript libraries which don't want to maintain TypeScript definitions.
In this case, contributors can maintain the types in [DefinitelyTyped](http://github.com/DefinitelyTyped/DefinitelyTyped).

This RFC proposes an equivalent monorepo, tentatively called "_eslint-translations_".
It would be setup very similarly to [DefinitelyTyped](http://github.com/DefinitelyTyped/DefinitelyTyped). Specifically:

- A folder for each library, like `eslint-plugin-example`.
- Anyone could submit a PR to create or update translations
- A bot would automatically publish packages to npm, perhaps under a scope like `@eslint-translations/*`.
  This repository would be heavily automated, see [this flowchart](https://github.com/microsoft/DefinitelyTyped-tools/blob/main/packages/mergebot/docs/dt-mergebot-lifecycle.svg) from [DefinitelyTyped](http://github.com/DefinitelyTyped/DefinitelyTyped) for inspiration.

Potential repository structure:

```ini
├── packages
|  ├── eslint-plugin-example1
|  |   ├── de.json
|  |   └── zh-Hant.json
|  └── my-org__eslint-plugin    # for @my-org/eslint-plugin
|      ├── pt-BR.json
|      └── fr.json
# only showing interesting files for brevity
```

The files could be JSON or JS, and would export an object with the following structure:

```js
// es.json or es.js:
export default {
  example_rule_name: {
    avoidName: 'Evite utilizar variables llamadas {{name}}',
  },
};
```

### 3.3. Implementation for _eslint core_

#### 3.3.1. Determining the user's locale

Eslint would use the OS' locale. Obtaining the locale used to be a [complicated, platform-specific process](https://github.com/sindresorhus/os-locale/blob/main/index.js), that required a lot of [hardcoded data](https://github.com/sindresorhus/lcid/blob/main/lcid.json). Thanks to [`Intl`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl), it's now possible with 1 line of code (requires [NodeJS v13 or newer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/resolvedOptions#browser_compatibility)):

```js
Intl.DateTimeFormat().resolvedOptions().locale; // -> 'es-419'
```

Eslint could also support overriding the locale with a CLI argument like `--locale=de-DE` (further discussion in [§10.3](#10-3)).
This CLI argument would be a comma-separated string, where each value is a valid [BCP 47 language tag](https://developer.mozilla.org/en-US/docs/Glossary/BCP_47_language_tag).
If specified, the CLI argument will be the preferred locale, falling back to the OS' locale, and using the matching strategy in [§3.3.2](#332-matching-the-users-locale-to-available-translations).

The locale would be exposed to rules via [`context`](https://eslint.org/docs/latest/extend/custom-rules#the-context-object)`.locale`, in case they need it for their own formatting (see [§8.1](#81-limitations-of-message-placeholders) for an example).
This attribute would be an array of strings, to match [`navigator.languages`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/languages).
Using [`context.settings`](https://eslint.org/docs/latest/use/configure/configuration-files#configuring-shared-settings)`.locale` is an alternative, but it is not safe, because plugins might already be using this attribute for other purposes.

#### 3.3.2. Matching the user's locale to available translations

If the user's device reports the locale as [`zh-CN`](https://r12a.github.io/app-subtags/index?lookup=zh-CN), we should first call [`Intl.Locale#maximize`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/maximize) to expand the locale to include [likely subtags](https://www.unicode.org/reports/tr35/#Likely_Subtags).
This would give us [`zh-Hans-CN`](https://r12a.github.io/app-subtags/index?lookup=zh-Hans-CN) (or [`en-Latn-US`](https://r12a.github.io/app-subtags/index?lookup=en-Latn-US), [`ru-Cyrl-RU`](https://r12a.github.io/app-subtags/index?lookup=ru-Cyrl-RU), etc.).

We would then follow the [lookup algorithm from RFC-4647](https://datatracker.ietf.org/doc/html/rfc4647#section-3.4) to match the expanded code ([`zh-Hans-CN`](https://r12a.github.io/app-subtags/index?lookup=zh-Hans-CN)) to the most appropriate language for which we have translations.

#### 3.3.3. Resolving translations

> **Assumption:** The default language of [`meta.messages`](https://eslint.org/docs/latest/extend/custom-rules#placeholders-in-suggestion-messages) is English. If the user's language is [`en-*`](https://r12a.github.io/app-subtags/index?lookup=en), translations will not take effect.
> This ensures that eslint is not slowed down by this feature in CI environments, nor for users who expect English to be used.
> This assumption is not always correct ([counterexample: `messages` in Chinese](https://github.com/LLLaiYoung/eslint-plugin-aegis/blob/9780cd87240e2bbe0c134b97b8776782fe9016e1/lib/rules/no-implicit-complex-object.js#L28)).

**Case 1:** Resolving translations from the rule's source is trivial. [`computeMessageFromDescriptor()`](https://github.com/eslint/eslint/blob/23490b266276792896a0b7b43c49a1ce87bf8568/lib/linter/file-report.js#L440) would simply check `messageTranslations[locale][messageId]` before falling back to `messages[messageId]`.

**Case 2:** If translations cannot be found in the rule itself, we need to:

- check if `@eslint-translations/{packageName}` exists with [`require.resolve`](https://nodejs.org/api/modules.html#requireresolverequest-options), similar to the method used [to load formatters](https://github.com/eslint/eslint/blob/b34b93852d014ebbcf3538d892b55e0216cdf681/lib/eslint/eslint.js#L1197).
- if yes, read that package and check if it contains a matching language
- return the `messages` from that package.

### 3.4. Prior Art

- [microsoft/vscode-l10n](https://github.com/microsoft/vscode-l10n) is used for localising vscode extensions. Translations can contain placeholders like `Hello, {0}`, but [only literals are allowed](https://github.com/microsoft/vscode-l10n/blob/3fc1f7b7abfa6fc16e9ec9398ec3f2108112796b/l10n/src/main.ts#L161) (`string | number | boolean`).
- The TypeScript compiler [has translations](https://github.com/microsoft/TypeScript/tree/dba6c9b824852d3333f2314eca7ad995935227f4/src/loc/lcl) for all their [diagnostics messages](https://github.com/microsoft/TypeScript/blob/f5ccf4345d6ac01f47be84db99fde50b98accbb5/src/compiler/diagnosticMessages.json#L3268).

## 4. Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

Blog announcement.
Update documentation and provide examples for:

- a plugin that wants to maintain translations themselves
- a plugin where translations are maintained in a third-party package (like `@eslint-translations/*`).

## 5. Drawbacks

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think
    about any opposing viewpoints that reviewers might bring up.

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

1. Increased maintenance burden, if translators choose to maintain translations themselves.
2. Increased download size, if translators choose to maintain translations themselves.
3. Slower execution time when translation is enabled.
   For each plugin without 1st-party translations, _eslint core_ needs to check `node_modules` if 3rd-party translations are available (from a package called `@eslint-translations/{pluginName}`)
   <!-- TODO: this limitation could be resolved if 3rd-party translations are explicitly defined in eslint.config.js -->
4. The presence of translations may subconsciously discourage maintainers from tweaking the wording or structure of a rule message.
   For example, renaming a `{{placeholder}}` in a message would be a breaking change, because 3rd-party translations would also need to rename the placeholder.

## 6. Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

- For core- and plugin- maintainers: `messageId`s are now a public API, since they could be referenced in 3rd-party translations.
  The names of `{{placeholders}}` in messages are also public.
  Changing either `messageId`s or placeholders would be a breaking change if the package has 3rd-party translations.
- Some tools which [hardcode eslint error messages](https://github.com/microsoft/vscode-eslint/blob/fd82f23df1d0e2b7e5040ad3532444d0f66cf8dd/server/src/eslint.ts#L184) will break.
- No impact on the minimum supported NodeJS version; [the relev](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/resolvedOptions#browser_compatibility)[ant `Intl` APIs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/maximize#browser_compatibility) have been supported since Node v13
- `context.locale` is a new property.
  Rules must expect this property to be undefined in older eslint versions.
- `--locale` is a new CLI argument.
  Using `--locale` with an older eslint version will cause eslint to exit with a non-zero status code, like any other unsupported CLI argument.

## 7. Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

1. Maintain the status-quo; do not suppport localisation.
   Those who use AI tools in their IDE could continue to ask the LLM to translate the error.
   Everyone else would be required to read the error in English.
   Disadvantageous because natural language comprehension may be slower in English compared to people's native language.

2. Implement translation completely in userland, no official support from eslint.
   Anyone who wants translations would need to use a patched version of eslint (which [has been attempted](https://github.com/mysticatea/eslint-plugin-ja)). Or alternatively, patch `rule.messages` yourself in your `eslint.config.js` file.
   This is obviously less reliable and error prone.

## 8. Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

### 8.1. Limitations of message placeholders

Currently, Eslint [requires](https://github.com/eslint/rewrite/blob/7abc05147e2b6d29cb5170867c2172d25c563454/packages/core/src/types.ts#L166) message placeholders to be a literal.
Message placeholders also cannot reference other messages.

This creates several limitations:

1. Some rules need to include an array as a placeholder, so they must convert the array into a string themselves. [For example, using `array.join(' and ')`](https://github.com/typescript-eslint/typescript-eslint/blob/dd90da8f442a651ab0dbaead80735f9259260569/packages/eslint-plugin/src/util/misc.ts#L187).
2. Some rules [include English text in message placeholders](https://github.com/typescript-eslint/typescript-eslint/blob/dd90da8f442a651ab0dbaead80735f9259260569/packages/eslint-plugin/src/rules/no-unnecessary-type-conversion.ts#L284-L287), or even [entire sentances](https://github.com/ota-meshi/eslint-plugin-regexp/blob/2524e3e6e57d4e852245a16938e021846f43f25b/lib/rules/optimal-quantifier-concatenation.ts#L654-L668), because it's more concise than creating many `messageId`s.
3. Some major plugins like `eslint-plugin-react-hooks` do [not use `messageId`s](https://github.com/facebook/react/blob/5aec1b2a8d1bcafb43a7ddb64dc37eacad081bea/packages/eslint-plugin-react-hooks/src/rules/ExhaustiveDeps.ts#L627-L632), so translations would not be possible. Suggestions also don't use `messageId`s.
4. Some rules produce messages that are [far too co](https://github.com/facebook/react/blob/5aec1b2a8d1bcafb43a7ddb64dc37eacad081bea/packages/eslint-plugin-react-hooks/src/rules/ExhaustiveDeps.ts#L924-L938)[mplicated](https://github.com/facebook/react/blob/5aec1b2a8d1bcafb43a7ddb64dc37eacad081bea/packages/eslint-plugin-react-hooks/src/rules/ExhaustiveDeps.ts#L1052-L1053) to model with `messageId`s

In all cases, the result is that there are English words scattered across the code, which can't be translated.

Issue #1 is solvable: the plugin should replace their hardcoded `array.join(' and ')` with [`Intl.ListFormat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/ListFormat) (available since NodeJS v13).
This would allow an English speaker to see `"A", "B", and "C"`, while a Chinese speaker would see `A、B和C`.

The other issues can't be trivially solved, and would require eslint to support a more complex syntax for message placeholders.
There are several options:

1. Wait for the [TC39 proposal for `Intl.MessageFormat`](https://github.com/tc39/proposal-intl-messageformat), which is currently [stuck](https://github.com/tc39/proposal-intl-messageformat/issues/49) in Stage-1.
   Once supported natively without a polyfill, allow messages to use the [ICU Message Syntax](https://unicode-org.github.io/icu/userguide/format_parse/messages/).
   This would be a breaking change, but conveniently, the ICU Message Syntax uses single curly braces (`{ }`) for placeholders, while eslint currently uses double curly braces (`{{ }}`), so there is no syntax clash.
2. Support the [ICU Message Syntax](https://unicode-org.github.io/icu/userguide/format_parse/messages/) with an npm dependency (like [this one](https://github.com/messageformat/messageformat)), at least until the [TC39 proposal for `Intl.MessageFormat`](https://github.com/tc39/proposal-intl-messageformat) is natively supported.
   This increases the bundle size of eslint for very minimal benefit.
3. Support **only** recursively nested messages, no other complex syntax. For example:
   ```js
   messages: {
     banned: '{{name}} is not allowed{{extra}}',
     suffix1: ' because it could be confusing.',
     suffix2: ' because it is a reserved word.',
   }
   // …
   context.report({
    messageId: 'banned',
    data: { name: 'foo', extra: '{{suffix1}}' }
   });
   ```
4. Do nothing, make no change to the format of message placeholders.
   Accept that some translations will be imperfect, and will contain a mix of English and the other language.

### 8.2. governance of the `eslint-translations` monorepo

- How would this be maintained, would it be an [eslint-community](https://github.com/eslint-community) project?
- Does it make sense to publish an npm package for each plugin (like `@eslint-translations/eslint-plugin-example`), or a single npm package with translations for every plugin?
- Would IDE extensions like [vscode-eslint](https://github.com/microsoft/vscode-eslint) be willing to automatically download `@eslint-translations/*` packages, even if they're not specified in package.json?
  This is [what the TypeScript Language Server does](https://github.com/microsoft/vscode-docs/blob/71c6970d906cf2a9174d0c4223d759cf2c64a7bf/docs/nodejs/working-with-javascript.md?plain=1#L30) in VSCode, for `@types/*` packages (in `.js` files only)

## 9. Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

- Opinions from people who speak other languages are always valuable.
- Opinions from eslint maintainers, and maintainers of plugins and eslint IDE integrations.
- Opinions from plugin maintainers – is there a general preference for maintaining translations in the repo itself, or in a different translations project?

## 10. Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

> <a name="10-1"></a>10.1. Should translations be version-controlled or stored in a Translation Management System (TMS)?

I would suggest **not** using a Translation Management System because:

1. Version-controlling translations using JSON files is significantly simpler than integrating with a third-party commercial tool.
2. Translators are almost certainly software developers themselves, so storing translations in a git repository would not be a barrier.
3. Open-source tools exist to make it more convenient to manage translations in git (like [i18n-ally](https://github.com/antfu/i18n-ally))

---

> <a name="10-2"></a>10.2. What about translations for the docs and internal strings (such as [messages](https://github.com/eslint/eslint/tree/main/messages), [services](https://github.com/eslint/eslint/blob/43677de07ebd6e14bfac40a46ad749ba783c45f2/lib/services/warning-service.js#L33), and [inline configuration directives](https://github.com/eslint/eslint/blob/43677de07ebd6e14bfac40a46ad749ba783c45f2/lib/linter/linter.js#L1074))?

This is considered out of scope.
Docs translations are tracked by [eslint/eslint#11512](https://github.com/eslint/eslint/issues/11512), and internal strings are not exposed as a public API, so translations can be implemented much more easily.

---

> <a name="10-3"></a>10.3. Should it be possible to define your preferred language in `eslint.config.js`?

I believe **no**. Conceptually, it doesn't make sense to store your preferred language in version-control.
TypeScript also only allows [`--locale` as a CLI argument](https://www.typescriptlang.org/docs/handbook/compiler-options.html#:~:text=locale), it can't be defined in [tsconfig.json](https://typescriptlang.org/tsconfig).

## 11. Related Discussions

- Rule message translation:
  - 2026: [eslint/rfcs#141](https://github.com/eslint/rfcs/pull/141) (this RFC)
  - 2024: [eslint/rfcs#128](https://github.com/eslint/rfcs/pull/128)
  - 2024: [eslint/eslint#19210](https://github.com/eslint/eslint/issues/19210)
  - 2018: [eslint/eslint#9870](https://github.com/eslint/eslint/issues/9870)
- Docs translations:
  - 2025: [eslint/zh-hans.docs.eslint.org#130](https://github.com/eslint/zh-hans.docs.eslint.org/issues/130)
  - 2019: [eslint/eslint#11512](https://github.com/eslint/eslint/issues/11512)
  - 2015: [eslint/archive-website#161](https://github.com/eslint/archive-website/issues/161)
