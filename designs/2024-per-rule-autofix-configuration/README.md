- Repo: <https://github.com/eslint/eslint>
- Start Date: 2024-10-22
- RFC PR:
- Authors: [Samuel Therrien](https://github.com/Samuel-Therrien-Beslogic) (aka [@Avasam](https://github.com/Avasam))

# Per-rule autofix configuration

## Summary

<!-- One-paragraph explanation of the feature. -->
This feature aims to make it possible to control autofixes through shareable configuration on a per-rule basis.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->
Some rules provide autofixing, which is great, but can sometimes be broken or otherwise simply unwanted for various reasons.
Unsafe autofixes should be suggestions, and broken fixes should be reported, *but* ESLint is a large ecosystem where some very useful plugins are not always actively maintained. Even then, wanting to disable an autofix for project-specific or personal reasons could still happen.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

Similar to how Ruff (<https://docs.astral.sh/ruff/settings/#lint_unfixable>) does it, a top-level key to specify which rules to not autofix would be in my opinion the least disruptive and forward/backwards compatible. It should be overridable in per-file configurations, and picked up when extending a configuration.

Concretely, it could look like this:

```js
export default [
  {
    disableAutofixes: {
      // We don't want this to autofix, as a rule suddenly not failing should require human attention
      "@eslint-community/eslint-comments/no-unused-disable": true,
    },
    rules: {
      '@eslint-community/eslint-comments/no-unused-disable': 'error',
    }
  },
  {
    files: ["*.spec.js"],
    disableAutofixes: {
      // Let's pretend we want this to be autofixed in tests, for the sake of the RFC
      "@eslint-community/eslint-comments/no-unused-disable": false,
    },
  },
]
```

The fix should still exist as a suggestion. Only autofixing (when running `eslint --fix` or editor action on save) should be disabled.

The chosen key name `disableAutofixes` aims to remove the concern about "turning on" an autofix that doesn't exist. Disabling autofixes for a rule that doesn't have any or doesn't exist should be a no-op. Just like turning `off` a rule that doesn't exist. The reasoning being that this allows much more flexible shareable configurations.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
I think that "Configuring autofixes" or "Disabling autofixes" could be documented as a subsection of [Configuring Rules](https://eslint.org/docs/latest/use/configure/rules). Or as a section on the same level (between "Configuring Rules" and "Configuring Plugins")

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
A potential drawback I could see is that the configuration for autofixing a rule is not directly related with the rule itself. As a counter, I'd say this is already the case for plenty of rule-related settings, environment and parser configurations, etc. It's also less of a drawback than [Alternatives - Configure in the rule itself](#configure-in-the-rule-itself).

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->
Given that this proposal adds a new optional configuration section, this feature should be fully backwards compatible. Users that don't want to use this feature should stay completely unaffected. (see [Alternatives - Configure in the rule itself](#configure-in-the-rule-itself))

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

### Configure in the rule itself

Another approach I can think of is to encode that in the rule config itself. Something like `"my-plugin/my-rule": "[{severity: "error", autofix: False}, {...otherConfigs}]"` but it's harder to commit to such a change, and means that any config extension needs to reconfigure the rule correctly just to disable autofixing (which is already an issue when someone wants to set a pre-configured rule as warning for example)

### Use of a 3rd-party plugin

<https://www.npmjs.com/package/eslint-plugin-no-autofix> is a tool that exists to currently work around this limitation of ESLint, but it is not perfect.

1. It is an extra third-party dependency, with its own potential maintenance issues (having to keep up with ESLint, separate dependencies that can fall out of date, obsolete, unsecure, etc.)
2. It may not work in all environments. For example, pre-commit.ci: <https://github.com/aladdin-add/eslint-plugin/issues/98>
3. It may not work correctly with all third-party rules: <https://github.com/eslint-community/eslint-plugin-eslint-comments/issues/234>

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->
- Where exactly should the documentation go ?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->
My knowledge of ESLint's internals isn't that great. Whilst I think it's above the average user due to reading and configuring a lot, I haven't yet even learned how to write a plugin, and haven't migrated any project to ESLint 9 yet.
My free time both at work and personal, is currently also very limited (see how long it too me to just get to writing this RFC).
So I unfortunately don't think I can implement this feature myself, due to both a lack of time, personal motivation (I won't be able to use it for a while, but will push us towards ESLint 9 once implemented), and experience.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

Q: Could `disableAutofixes` be an array of autofixes to disable?
A: `disableAutofixes` is a record to allow re-enabling autofixes in downstream configurations and on a per-file basis. We could allow a shorthand to `disableAutofixes` to accept an array of rules to disable the autofix for, but that would result in additional complexity on the implementation side with marginal benefits to the user.

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
- <https://github.com/eslint/eslint/issues/18696>
- <https://github.com/aladdin-add/eslint-plugin/issues/98>
- <https://github.com/eslint-community/eslint-plugin-eslint-comments/issues/234>
