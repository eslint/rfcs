- Repo: eslint/eslint
- Start Date: 2022-02-28
- RFC PR:
- Authors: Brian Bartels

# ESLintRC Option To Forbid Disable For Specific Rules

## Summary

Allow users to forbid users from disabling certain rules. This is similar to `noInlineConfig` where the primary difference is that `noInlineConfig` is set for all rules

## Motivation

When working with complex lint configs. Some rules are more important than other. For this reason it can be inconsequential to disable certain rules. But incredibly bad to disable others.

EG:
```
//eslint-disable-next-line @government/required-SECURITY-rule
```
Is a much worse rule to disable than
```
//eslint-disable-next-line noExtraSemi
```

`noInlineConfig` works well for some. But in reality, many users disable rules at least somewhat frequently for valid reasons. And in large repos, there are going to be exceptions that need inline configuration. 

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->
in `linter.js` I will adjust the `getDirectiveComments` function so that I reference a list of forbidden rules. If someone is disabling a rule that is forbidden. I will add a warning (error?) similar to `warnInlineConfig`

In `eslintrc` I will add another option for rules. The new state can be
```
warn
error
forbid
```
where forbid is always an error and cannot be overridden by a directive


## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
We will want to update the documentation to reflect the new changes

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

this adds complexity to the already complex eslintrc experience

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->
This should integrate into the planned eslintrc changes and therefore will not be backwards compatible.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

instead of adding a state to `error/warn` we could instead add another optional field to the rule config. This would enable a rule to be `warn` and have directives disabled. However, I believe that this actually adds more bloat to eslintrc which is why I chose the above design

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

- How Can I gracefully integrate into the eslintrc updates

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
