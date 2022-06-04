- Repo: No ESLint repos, only the ones mentioned below (owned by private
  persons)
- Start Date: 2022-05-31
- RFC PR: <https://github.com/eslint/rfcs/pull/91>
- Authors: [@aladdin-add](https://github.com/aladdin-add),
  [@MichaelDeBoey](https://github.com/MichaelDeBoey) &
  [@ota-meshi](https://github.com/ota-meshi)

# `@eslint-community` GitHub organization

## Summary

<!-- One-paragraph explanation of the feature. -->

Facilitate a GitHub organization (eg.
[@eslint-community](https://github.com/eslint-community)) where community
members can help ensure widely depended upon ESLint related packages stay up to
date with newer ESLint releases and doesn't hold the wider community back.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

There are some widely depended upon ESLint related packages that aren't kept up
to date with eg. new ESLint releases. The community would like a place to
collectively step in and ensure that these doesn't hold the wider community
back.

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
`@eslint-community` org would be a centralized home for ESLint related packages
that don't belong in the main org or that are widely depended upon, aren't
maintained anymore and are given back to the community.

> Community repos for ESLint related projects

The benefit of having the org in place would be that the community isn't
dependent on one person's GitHub/`npm` account.  
The added benefit is that the people maintaining the packages in the new org
_could_ create a structure to streamline the packages more. They _could_ for
instance decide things like using the same tools, supporting the same
Node/ESLint versions, if we want to release all "rescued" packages under the
`@eslint-community` `npm` scope or not, ...

As a reference, `@jest-community` has the following goal & intention:

> The goal of `@jest-community` is to have a loose but centralized home for
> stuff `jest` core thinks is neat and worth having but doesn't belong in the
> core itself. The org is a way to help ensure that these projects are not
> dependent on one person's GitHub account in case they move on.

The beauty of having a shared org is that the release process is centralised, so
no need to talk to `npm` about an ownership transfer and no need to fork and
republish under a new name. Just appoint new maintainers on GitHub and they can
work to restore continuity of service - a process which would be invisible to
users (i.e. users might see a pause in updates but not have to make changes -
the updates would just resume).

### Which projects should be involved?

I think all of [@aladdin-add](https://github.com/aladdin-add),
[@ota-meshi](https://github.com/ota-meshi),
[@voxpelli](https://github.com/voxpelli) & myself would be in for helping to
maintain at least the following packages:
[`@mysticatea/eslint-plugin`](https://github.com/mysticatea/eslint-plugin),
[`eslint-plugin-eslint-comments`](https://github.com/mysticatea/eslint-plugin-eslint-comments),
[`eslint-plugin-es`](https://github.com/mysticatea/eslint-plugin-es),
[`eslint-plugin-node`](https://github.com/mysticatea/eslint-plugin-node),
[`eslint-utils`](https://github.com/mysticatea/eslint-utils) &
[`regexpp`](https://github.com/mysticatea/regexpp) (all from
[@mysticatea](https://github.com/mysticatea)),
[`eslint-plugin-eslint-plugin`](https://github.com/not-an-aardvark/eslint-plugin-eslint-plugin)
(which is a recommended plugin for linting custom plugins & was already
mentioned by [@not-an-aardvark](https://github.com/not-an-aardvark) —the current
maintainer— to `@aladdin-add` that he wanted to transfer the repo to a better
home) &
[`eslint-plugin-promise`](https://github.com/xjamundx/eslint-plugin-promise)
(which is used a lot as well and was mentioned by the current maintainer
—[@xjamundx](https://github.com/xjamundx)— that he would like to transfer it to
the community org if one was going to exist).

### What would be the acceptance criteria?

I personally think that the acceptance criteria for a package to go under the
`@eslint-community` org would be:

- Is it a package that is ESLint-related?  
  Mostly this will be ESLint plugins, but I can see (unmaintained) dependencies
  of such packages, closely related packages or packages split from the main
  ESLint repo (like
  [`eslint-formatter-codeframe`](https://github.com/fregante/eslint-formatter-codeframe)
  or
  [`eslint-formatter-table`](https://github.com/fregante/eslint-formatter-table),
  which are currently maintained by [@fregante](https://github.com/fregante)) or
  used by the main repo (like
  [`eslint-utils`](https://github.com/mysticatea/eslint-utils) &
  [`regexpp`](https://github.com/mysticatea/regexpp)) to be in the
  `@eslint-community` org as well.  
  It also shouldn't matter if the package is purely TypeScript-related or not,
  as refusing these packages and having a separate
  `@typescript-eslint-community` org would only fragmentize the ESLint community
  more without any real benefit.

- Is it widely depended upon throughout the community?  
  Don't have an exact number in mind, but all mentioned packages have at least
  3M downloads/week.  
  Only exceptions are `eslint-plugin-eslint-comments` (1M downloads/week) &
  `@mysticatea/eslint-plugin` (1.3K downloads/week), which is used as a
  devDependency for all other mentioned packages by `@mysticatea`.

- It didn't receive an update in the last 12(?) months  
  I think we can make an exception for packages that did get a PR merged which
  was created by a member of the community core team (mostly for updating
  compatibility with newer versions of ESLint).

  We can also make an exception for packages that meet the other criteria, but
  where the owner want to transfer ownership to the new org.  
  I can for instance imagine that people like [@gajus](https://github.com/gajus)
  at some point would want to transfer ownership of plugins like
  [`eslint-plugin-flowtype`](https://github.com/gajus/eslint-plugin-flowtype)
  and/or [`eslint-plugin-jsdoc`](https://github.com/gajus/eslint-plugin-jsdoc)
  if they would feel it would be better maintained by the community instead of
  putting all workload on their shoulders. I can see the same happening for
  [`eslint-plugin-security`](https://github.com/nodesecurity/eslint-plugin-security),
  which is currently maintained by the
  [@nodesecurity](https://github.com/nodesecurity) team.

- It should agree to the code of conduct of the org
- It should have an OSI-approved license
- `eslintbot` should be added as an owner to the original npm package

If a package is meeting these criteria, but we can't get into contact with the
owner of the repo to transfer ownership over to the new org, we could
temporarily fork the repo into the new org. However, the ultimate goal would
always be to transfer the original repo to the new org.  
If we had a temporary fork, we should create PRs to the original repo to bring
it up to date with the fork before continuing with other tasks.

### What permissions should maintainers have?

I think it would be best to have some sort of a community core team (of at least
3(?) people) that's making the decisions in terms of acceptance to the repo,
which would (in addition to ESLint TSC) all be admins of the org.

We can make a team (with sub-teams, if we want) for each package, where we give
maintainer rights on the repo for that specific package, but keep admin rights
to ESLint TSC & the community core team. This way we prevent that the repo would
be destroyed from the org.

We'll create a setup where we publish new versions of the package with
`eslintbot`. That way we can keep publishing new versions if someone would
dissapear.

The community core team would help maintain all repos, whereas people in
specific teams would only help maintain these repositories.

The community core team would be responsible of administer the Code of Conduct
of the organization.

The community core team would also be responsible to contact package maintainers
(outside of GitHub) if they appear inactive. If the package maintainers don't
respond in a reasonable time frame (npm gives you 14(?) days to respond to a
transfer request email), they could be considered non-responsive and the
community core team can step in to appoint a new maintainer for the repository.

**Bonus:** I think having our own place in the Discord server where we can
discuss some things would be a good idea as well.  
This way the community core team has an open place to discuss some common things
like extracting common functionality into a separate package, minimum Node &
ESLint target, how to handle disputes between maintainers, ...  
Having a channel for each separate package would maybe be a good idea as well,
so the community has a chance to ask questions to the maintainers as well.

### What should be the role of the ESLint team?

The idea is that the ESLint team is the ultimate caretakers, they assign a
community core team to take care of the effort the new org makes and let them
ran the day to day business so that the ESLint team rarely has to involve
themselves.

Next to that, I think most important role would be to facilitate at least the
contact with `@mysticatea`, so we can transfer his packages over to the
`@eslint-community` org & gain `npm` publish rights for the packages.

I can also see the ESLint team talk to the community core team of the new org
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
- `eslint-utils` & `regexpp` These packages are both used by the main ESLint
  repo & currently maintained by `@mysticatea`  
  <https://github.com/eslint/eslint/blob/ce035e5fac632ba8d4f1860f92465f22d6b44d42/package.json#L66>
  <https://github.com/eslint/eslint/blob/ce035e5fac632ba8d4f1860f92465f22d6b44d42/package.json#L87>

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

A formal announcement on the ESLint blog would be preferred to let the community
know about this new org and about which forks we're maintaining.

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

One of the drawbacks could be that this new `@eslint-community` org could give
the impression that the packages owned by this new org are recommended by the
core team.

Secondly, It could also put pressure on packages from other ESLint related
organizations/maintainers to transfer their project to this new org, as it could
be seen as the "correct" place for popular packages.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

It won't affect existing ESLint users, as package could be released under the
same name on `npm`.  
Even when we would release a package under the `@eslint-community` npm scope
(for instance because we can't get the rights to publish to the existing npm
package), people could still use the old package without any extra problems
(except for the ones they already had) if they wanted to.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

- Private persons could create a fork of the above mentioned packages.  
  This could however create several well maintained forks, which would cause
  fragmentation in the community.

  More importantly, this could cause the "problem" of the fork being reliant on
  that specific individual, which could put us in the same position
  (abandoned/unmaintained package) all over again at some point in time.
  Possibly having a new fork each X amount of time, would also have the burden
  of needing to update all existing dependants each time.

- The community could independently create multiple more focused organizations
  to maintain a subset of ESLint related packages.  
  Some examples of already existing organizations like that are
  [@import-js](https://github.com/import-js) (which maintains
  [`eslint-plugin-import`](https://github.com/import-js/eslint-plugin-import)),
  [@jsx-eslint](https://github.com/jsx-eslint) (which maintains
  [`eslint-plugin-jsx-a11y`](https://github.com/jsx-eslint/eslint-plugin-jsx-a11y),
  [`eslint-plugin-react`](https://github.com/jsx-eslint/eslint-plugin-react) &
  [`jsx-ast-utils`](https://github.com/jsx-eslint/jsx-ast-utils)) &
  [@standard](https://github.com/standard) (which maintains a couple of
  [`standard`](https://github.com/standard/standard) related ESLint configs)

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
- What's the main place of communication for the community core team?
- Would projects be allowed to accept sponsorships?
- How does this relate to stuff in eg. `@standard`?  
  Would there be an expectancy from the wider community or the
  `@eslint-community` org itself to transfer some stuff from these places?
- What about projects like
  [`eslint-plugin-unicorn`](https://github.com/sindresorhus/eslint-plugin-unicorn)?  
  Will it be considered a problem that such a plugin is under
  [@sindresorhus](https://github.com/sindresorhus)' ownership rather than the
  `@eslint-community` org?
- How about something like
  [`eslint-config-airbnb-base`](https://github.com/airbnb/javascript/tree/master/packages/eslint-config-airbnb-base)?  
  Could a ruleset like that be eligible for this new org if abandoned at some
  point?  
  Would there be a push by people to transfer such a module here?  
  Will that make it look like an officially endorsed configuration?  
  How would that affect organizations like `@standard`?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

We would need to get access to the `@eslint-community` org & be brought in
contact with `@mysticatea` to transfer his repos to the new org & get `npm`
publish rights on these repos.

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

- https://github.com/eslint/eslint/issues/15383
- https://github.com/eslint/eslint/discussions/15929
- https://github.com/mysticatea/eslint-plugin/pull/29#issuecomment-1139495904
- https://github.com/standard/eslint-config-standard/issues/192#issuecomment-1000279972
- https://github.com/weiran-zsd/eslint-plugin-node/issues/8
