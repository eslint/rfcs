- Repo: eslint/eslint
- Start Date: 2023-03-22
- RFC PR: (leave this empty, to be filled in later)
- Authors: [Mattstir](https://github.com/Mattstir)

# Api support for the timing or general statistics information

## Summary

<!-- One-paragraph explanation of the feature. -->

Add possibility to get the [TimingInformation](https://eslint.org/docs/latest/extend/custom-rules#profile-rule-performance) also via the [node.js api](https://eslint.org/docs/latest/integrate/nodejs-api).

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

In big pipelines the node.js api is used as it can be integrated in a custom build system. As this is used for big projects, keeping an eye on the performance of rules (also custom ones) is a good thing to do.
As the [TimingInformation](https://eslint.org/docs/latest/extend/custom-rules#profile-rule-performance) currently is only logged its hard cumbersome to automatically report this information to some KPI tool.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

As stated in [this comment](https://github.com/eslint/eslint/issues/16521#issuecomment-1312220087) it might be the best option to add new functions. That way its no breaking change and it shouldn't break any usages of people.

Currently there are `eslint.lintFiles` & `eslint.lintText`, these return `Promise<ESLint.LintResult[]>`.

The idea would be to add:
* `eslint.lintFilesWithTiming` or `eslint.lintFilesWithStatistics`
* `eslint.lintTextWithTiming` or `eslint.lintTextWithStatistics`

These return 
```ts
Promise<{
    results: ESLint.LintResult[]
    <timings|statistics>: {
        performance: {
            // name of the rule, e.g.: "no-multi-spaces"
            rule: string;
            // the time the rule consumed in ms, e.g.: 52.472
            time: number;
            // the relative time in % of the total time, e.g.: 6.1
            timeRelative: number;
        }[]
        ... other statistics information that people want to add in the future
    }
}>
```

As you can see I added the example with either only Timing or a more general approach that would be futureproof to also enable more things like stated in https://github.com/eslint/eslint/issues/14597.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

There should be a new section added to https://eslint.org/docs/latest/extend/custom-rules#profile-rule-performance 

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
Slightly bigger package size for something that not everyone will use.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

With the adding of new functions there shouldn't be any backward compatibility problems.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->
For custom implemented rules someone could add a custom "performance capture" logic to get information about these rules. This of course only works for custom ones then and would leave a lot of things in the dark.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->
Should we do a timing specific implementation or a generic one that can get easily extended in the future?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->
I could implement this RFC on my own. Just quite new to eslint internals...

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

...

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

* Origin eslint issue: https://github.com/eslint/eslint/issues/16521
* Related issue that also wants to report details: https://github.com/eslint/eslint/issues/14597 

