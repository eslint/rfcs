- Repo: No ESLint repos, only the ones mentioned below (owned by private
  persons)
- Start Date: 2022-05-31
- RFC PR:
- Authors: [@aladdin-add](https://github.com/aladdin-add),
  [@MichaelDeBoey](https://github.com/MichaelDeBoey),
  [@ota-meshi](https://github.com/ota-meshi) &
  [@voxpelli](https://github.com/voxpelli)

# `@eslint-community` GitHub organisation

## Summary

<!-- One-paragraph explanation of the feature. -->

Facilitate a GitHub organisation
([@eslint-community](https://github.com/eslint-community)) where community
members could maintain widely used ESLint related packages that would otherwise
get abandoned.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

There a some widely used ESLint related packages that aren't actively maintained
anymore and we'd like to transfer them to a community owned GitHub organization
(`@eslint-community`), so we can give them back to the community.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

### How do you envision `@eslint-community`?

Just like the [@jest-community](https://github.com/jest-community) org, the
`@eslint-community` org would be a collection of important/widely used ESLint
plugins/packages that aren't maintained anymore and are given back to the
community.

> Community repos for ESLint related projects

The projects could be released under the `@eslint-community` scope in a later
stage.

The benefit of having the org in place would be that we could create a structure
to streamline the packages more, i.e. tools used, supported Node/ESLint
versions, ...

### Which projects should be involved?

I think all of [@aladdin-add](https://github.com/aladdin-add),
[@ota-meshi](https://github.com/ota-meshi) &
[@voxpelli](https://github.com/voxpelli) & myself would be in for helping to
maintain at least the following packages:
[`@mysticatea/eslint-plugin`](https://github.com/mysticatea/eslint-plugin),
[`eslint-plugin-eslint-comments`](https://github.com/mysticatea/eslint-plugin-eslint-comments),
[`eslint-plugin-es`](https://github.com/mysticatea/eslint-plugin-es),
[`eslint-plugin-node`](https://github.com/mysticatea/eslint-plugin-node),
[`eslint-utils`](https://github.com/mysticatea/eslint-utils) (all from
[@mysticatea](https://github.com/mysticatea)) &
[`eslint-plugin-promise`](https://github.com/xjamundx/eslint-plugin-promise)
(which is used a lot as well and was mentioned by the current maintainer
([@xjamundx](https://github.com/xjamundx)) that he would like to transfer it to
the community org if one was going to exist).

### What would be the acceptance criteria?

I personally think that the acceptance criteria for a plugin to go under the
`@eslint-community` org would be:

- Is it a package that is ESLint-related?  
  Mostly this will be ESLint configs/plugins, but I can see unmaintained
  dependencies of such packages, closely related packages or packages split from
  the main ESLint repo (like
  [`eslint-formatter-codeframe`](https://github.com/fregante/eslint-formatter-codeframe)
  or
  [`eslint-formatter-table`](https://github.com/fregante/eslint-formatter-table),
  which are currently maintained by [@fregante](https://github.com/fregante)).

- Is it widely used throughout the community?  
  Don't have an exact number in mind, but all mentioned packages have at least
  3M downloads/week.  
  Only exceptions are `eslint-plugin-eslint-comments` (1M downloads/week) &
  `@mysticatea/eslint-plugin` (1.3K downloads/week), which is used as a
  devDependency for all other mentioned packages by `@mysticatea`.

- It didn't receive an update in the last 12(?) months  
  I think we can make an exception for packages that did get a PR merged which
  was created by a member of the new org's "core" team (mostly for updating
  compatibility with newer versions of ESLint).

  We can also make an exception for packages that meet the other criteria, but
  where the owner want to transfer ownership to the new org.  
  I can for instance imagine that
  [@not-an-aardvark](https://github.com/not-an-aardvark) would want to transfer
  ownership of
  [`eslint-plugin-eslint-plugin`](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin)
  to the new org as well, as it's a recommended plugin for linting custom
  plugins.  
  <https://eslint.org/docs/developer-guide/working-with-plugins#linting>  
  I can also imagine that people like [@gajus](https://github.com/gajus) at some
  point wanting to transfer ownership of
  [`eslint-plugin-flowtype`](https://github.com/gajus/eslint-plugin-flowtype) or
  [`eslint-plugin-jsdoc`](https://github.com/gajus/eslint-plugin-jsdoc) if they
  would feel it would be better maintained by the community instead of putting
  all workload on their shoulders.

If a package is meeting these criteria, but we can't get into contact with the
owner of the repo to transfer ownership over to the new org, we could
temporarily fork the repo & publish the package as
`@eslint-community-fork/<package name>` or so.

Maybe [@G-Rath](https://github.com/G-Rath) &
[@SimenB](https://github.com/SimenB) could give some insights on the acceptance
criteria for `@jest-community` & we can learn from their findings as well.

### What permissions should maintainers have?

I think it would be best to have some sort of a "core" team (of at least 3(?)
people) that's making the decisions in terms of acceptance to the repo, which
would all be admin of the org.

We can make a team (with sub-teams if we want) for each package, where we give
admin rights on the repo for that specific package.

The core team would help maintain all repos, whereas people in specific teams
would only help maintain these repositories.

**Bonus:** I think having our own place in the Discord server where we can
discuss some things would be a good idea as well.  
This way the "core" team has an open place to discuss some common things like
extracting common functionality into a separate package, minimum Node & ESLint
target, how to handle disputes between maintainers, ...  
Having a channel for each separate package would maybe be a good idea as well,
so the community has a change to ask questions to the maintainers as well.

### What should be the role of the ESLint team?

I think most important role would be to facilitate at least the contact with
`@mysticatea`, so we can transfer his packages over to the `@eslint-community`
org & gain `npm` publish rights for the packages.

Next to that, I can see the ESLint team talk to the "core" team of the new org
regarding creating new community maintained packages that are split out from the
main ESLint repo. Examples of these kind of repos are:

- `eslint-plugin-node`  
  This plugin was recommended because of the deprecation of some rules in v7  
  <https://eslint.org/docs/user-guide/migrating-to-7.0.0#deprecate-node-rules>
- `eslint-formatter-codeframe` & `eslint-formatter-table`  
  These were both in the main ESLint repo but were removed in v8 and are
  currently maintained by [@fregante](https://github.com/fregante)  
  <https://eslint.org/docs/8.0.0/user-guide/migrating-to-8.0.0#-removed-codeframe-and-table-formatters>
- `eslint-plugin-eslint-plugin`  
  This plugin is (together with `eslint-plugin-node`) a recommended plugin for
  linting custom plugins.  
  <https://eslint.org/docs/developer-guide/working-with-plugins#linting>

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

I don't think this RFC needs a formal announcement on the ESLint blog.

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

I can't think of a reason why we would not want this.  
Once we have access to the `@eslint-community` org & have the repos transfered,
the "core" team can work out the rest of the process themselves.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

It won't affect existing ESLint users, as package could be released under the
same name on `npm`.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

Creating a fork of the above mentioned packages. This could however create
ceveral forks (which would cause fragmentation in the community), but also has
the burden of needing to update all existing dependants.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

- What exact acceptance criteria do we want to use?
- What's the main place of communication for the "core" team?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

We would need to get access to the `@eslint-community` & be brought in contact
with `@mysticatea` to transfer his repos to the new org & get `npm` publish
rights.

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
