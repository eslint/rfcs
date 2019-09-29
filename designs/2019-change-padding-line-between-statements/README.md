- Start Date: 2019-09-29
- RFC PR: (leave this empty, to be filled in later)
- Authors: Igor Novozhilov [@IgorNovozhilov]

# Change rule: padding-line-between-statements

## Summary

<!-- One-paragraph explanation of the feature. -->
* Changes to the `padding-line-between-statements` rule to correct the issue [#11129](https://github.com/eslint/eslint/issues/11129):
* Rename `padding-line-between-statements` to `padding-line-around-statements`
* Add new types to `STATEMENT_TYPE`

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->
* This will combine the checks of rules `lines-around-comment` and `no-multiple-empty-lines` for empty lines into one rule
* Fix this ptoblem: [#11129](https://github.com/eslint/eslint/issues/11129)
* The rule `padding-line-around-statements` will cover all aspects of having blank lines in the file,
    but the maximum number of blank lines allowed will remain in the rule `no-multiple-empty-lines`

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->
Copy code of `padding-line-between-statements` to `padding-line-around-statements`.
Mark `padding-line-between-statements` as deprecated.

New types for `STATEMENT_TYPE`:
* `multiline-comment`
* `singleline-comment`
* `beginning-of-file`
* `end-of-file`

Old and new types will interact according to the existing rules.
Currently checking comments is absorbed in the analysis of the spacing between statements.
After the changes, the indentation between comments and statements will be set in the configuration.

Rule `lines-around-comment` mark as `deprecated`. Rule options `allow*` no longer support.

Options `maxEOF`, `maxBOF` of rule `no-multiple-empty-lines` mark as deprecated.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
* Add documentation for the new rule `padding-line-around-statements` (copy of `padding-line-between-statements`, with modifications)
* Add `deprecated` notes to `padding-line-between-statements`, `lines-around-comment`, `no-multiple-empty-lines`

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
Lose some features of rule `lines-around-comment`. Options:
* `allowBlockStart`
* `allowBlockEnd`
* `allowObjectStart`
* `allowObjectEnd`
* `allowArrayStart`
* `allowArrayEnd`
* `allowClassStart`
* `allowClassEnd`
* `applyDefaultIgnorePatterns`
* `ignorePattern`

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->
These are major changes.

You should prepare the migration manual from `padding-line-between-statements` to `padding-line-around-statements`.
With recommendations on new features of accounting comments in the code.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

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
