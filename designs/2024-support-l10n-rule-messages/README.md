- Repo: eslint/eslint
- Start Date: 2024-12-09
- RFC PR: (leave this empty, to be filled in later)
- Authors: [gweesin](https://github.com/gweesin)

# Support Localization of Rule Messages

## Summary

<!-- One-paragraph explanation of the feature. -->

This RFC proposes adding support for localization of rule messages in ESLint. The goal is to allow users to provide translations for rule messages, enabling ESLint to display messages in different languages based on user preferences. This feature aims to improve the accessibility and usability of ESLint for non-English speaking users.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

The primary motivation for this feature is to make ESLint more accessible to a global audience. Currently, ESLint rule messages are only available in English, which can be a barrier for non-English speaking users. By allowing rule messages to be localized, we can improve the user experience for developers who prefer to work in their native language. This change will help ESLint reach a broader audience and make it easier for developers around the world to understand and fix linting issues in their code.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

To implement localization of rule messages in ESLint, we will support two ways to define the localization information:

1. Support for plugin definition messages with locale files.
   plugins can define localized messages directly within their rule definitions so that all the messages can be supported in the same repository. This approach allows plugin authors to provide translations for their rule messages without relying on external files.

   Example:

   ```js
   module.exports = {
      meta: {
         messages: {
            someMessageId: {
               en: "Default message in English",
               es: "Mensaje predeterminado en español"
            }
         }
      },
      create(context) {
         return {
            Identifier(node) {
               context.report({
                  node,
                  messageId: "someMessageId"
               });
            }
         };
      }
   };
   ```

2. Support a way using `eslint.config.js` file to configure message info by message ID.
   This way should allow users to provide some plugins to update other plugins' messages. This approach allows users to customize the messages for rules provided by plugins.

   Example:
    
   ```js
   module.exports = {
     locale: "es",
     rules: [/* ... */],
     messages: {
       "plugin/ruleId": {
         someMessageId: "Mensaje predeterminado en español"
       }
     }
   };
   ```

   and eslint will use the message info to replace the message in the rule if the message ID is found in the configuration.

for corner cases, we need to consider the following:

1. If a translation is missing for a specific language, ESLint should fall back to the default English message.
2. ESLint should support original flat rules structure and the new structure with localized messages.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

1. Update the official ESLint documentation to include information on how to create and use localization files.
2. Provide examples and guidelines for contributing translations, including third party integrations.

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

1. Increased Maintenance Burden: Adding support for localization will increase the complexity of the ESLint codebase. Maintaining translations and ensuring they are up-to-date with rule changes will require additional effort from both the core team and contributors.
2. It's better to use a module to handle the localization such as typescript using `@types/some-module` but it's not possible to use a module to handle the localization in ESLint.
3. Potential for Incomplete Translations: There is a risk that some languages may have incomplete translations, leading to a mix of localized and default English messages.
   This could confuse users and reduce the overall effectiveness of the feature.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This change should not affect existing ESLint users who do not use the localization feature. Users who do not provide translations will continue to see the default English messages. To minimize disruption, ESLint should provide clear documentation on how to create and use localization files, as well as guidelines for contributing translations.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

1. TypeScript `--locale` option: This option allows users to specify the locale for the TypeScript compiler, which affects the language service features. However, this approach is limited to the TypeScript compiler and does not provide a general solution for localization in ESLint.
   > [TypeScript compiler options](https://www.typescriptlang.org/docs/handbook/compiler-options.html#compiler-options)
   > https://github.com/eslint/eslint/issues/9870#issuecomment-359228967

   why not? because almost all the messages are provided by third party plugins, so it's not possible to handle the localization in the core.

2. Is it possible to check `@locale/module` in the same way as `@type/module`?

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I sure I can implement this RFC on my own.

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

- https://github.com/eslint/eslint/issues/19210
- https://github.com/eslint/eslint/issues/9870