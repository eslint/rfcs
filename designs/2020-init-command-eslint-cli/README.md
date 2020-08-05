- Repo: [eslint/eslint-cli](https://github.com/eslint/eslint-cli), [eslint/eslint](https://github.com/eslint/eslint)
- Start Date: 2020-06-29
- RFC PR: 
- Authors: Aniketh Saha ([anikethsaha](https://github.com/anikethsaha))

# init command in `eslint-cli`

## Summary

Move the `init` command from main repo ([eslint](https://github.com/eslint/eslint)) to a new repo named `@eslint/create`

## Motivation

Currently the whole `init` command is being shipped with the cli in main repo. Though its not that much of size (roughly ~0.6MB for the [eslint/lib/init](https://github.com/eslint/eslint/tree/master/lib/init)), this command is not a type of command that we need 
everyday or everytime running eslint. It is mainly used when creating a new project or adding eslint to a project for first time. So if we move this to a separate package that is meant to use in command line, 
We will make use of tools like `npx` to simply run `npx @eslint/create` or using `npm` to run `npm init @eslint` or using `yarn` to run `yarn create @eslint` single time for a project instead of having it in the core project. 

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

The implementation is mainly migrating the code from main repo to `eslint-cli`. As far as the modules are concerned, all shared modules are being shipped currently with the eslint so we can use those modules through eslint itself. 
As far as the `eslint-recommended` rules are concerned, it will be done as a copy paste to the `@eslint/create` repo. 

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
Documentation for `@eslint/create` will be created with proper usage and each prompt's details. In the main documentaion, a link to `@eslint/create` will be given and basic usage and the package details will be written. Like this 
Yes as it will be a major change for the `eslint` repo itself as it is being remove from the main package/repo, I guess a blog can be a good solution to solve inital queries. 

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
This might increase maintenance burden and if we are planing to have a monorepo, it might be a more burden with a requirement of proper tooling. 


## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->
Releasing with a proper message which the command is being run from `eslint` with something like - `"this command has been moved from here, use 'npx @eslint/create'"` might be a good way to address backward compatibility. 
Otherwise we can keep the feature in the main repo as well for starting few days/weeks with a warning and then removing it completing with an error message. 

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->
The current implementation in `eslint` repo is the alternative. No changes needed. 

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

Do we want to implement this as monorepo under `eslint` repo or a separate repo `@eslint/create` but github will make it as `eslint-create` or similar ?
