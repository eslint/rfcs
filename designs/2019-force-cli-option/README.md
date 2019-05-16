- Start Date: 2019-05-16
- RFC PR: 
- Authors: Tit Kovalenko


# Add force CLI option

## Summary

<!-- One-paragraph explanation of the feature. -->

Adding new `--force` CLI option which will force linting process to exit with `0` error code even if errors or warnings appeared.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Such a functionality can be useful while using in pre-commit git hooks.
So for example each time before commit `eslint . --fix --force` can run, fixing all automatically fixable errors and then commit.
Currently if there non fixed errors left commit will not be made because linting process fails with >0 error code.
And it not possible to make intermediate commits using such a hook with console logs and debuggers while developing new functionality or investigating issues.
So the `--force` flag will help to handle the case.

Inspired by [tslint --force](https://palantir.github.io/tslint/usage/cli/)

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

Adding new boolean option named `force` to the existing list of cli options at `./lib/options.js`.

Extend condition in `./lib/cli.js:228` to return `0` error code even if errors occurred and number of warnings is too big.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

Adding option to registry seems like it will automatically will add it's description which will be displayed on `eslint --help`

Also it seems appropriate to extend CLI docs on website.

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

While investigating related to this functionality discussions [#2949](https://github.com/eslint/eslint/issues/2949) [#6882](https://github.com/eslint/eslint/issues/2949)
I didnt found the drawbacks of such an approach which is more explicit and natural for using CLI.
There were suggestions
1. Use Node api for such a purposes to wrap eslint call

_This looks as a complex solution for simple case.
Using this developer have to move from declarative description setting
description for my tool using options and config to building new layer of tool_

2. append CLI command with `|| exit 0`

_This approach ignores all of the errors but not only rules violation.
So there will be no way to know something actually went wrong_

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

My expertize regarding the project is definitely low. However I cannot se the problems with backward compatibility.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

Mentioned in drawbacks.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

Nothing at the point.

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

There is a PR with implementation in this [PR#11726](https://github.com/eslint/eslint/pull/11726)

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

[#2949](https://github.com/eslint/eslint/issues/2949)
[#6882](https://github.com/eslint/eslint/issues/2949)
[#11726](https://github.com/eslint/eslint/pull/11726)
