- Repo: [eslint/eslint](https://github.com/eslint/eslint)
- Start Date: 2020-06-29
- RFC PR: 
- Authors: Aniketh Saha ([anikethsaha](https://github.com/anikethsaha))

# Move --init flag into a separate utility

## Summary

Move the `init` command from main repo ([eslint](https://github.com/eslint/eslint)) to a new repo named `@eslint/create`

## Motivation

Currently the whole `init` command is being shipped with the cli in the main repo. Though its not that much of size (roughly ~0.6MB for the [eslint/lib/init](https://github.com/eslint/eslint/tree/master/lib/init)), this command is not a type of command that we need everyday or everytime running eslint. It is mainly used when creating a new project or adding eslint to a project for the first time. So if we move this to a separate package that is meant to use in the command line, 
We will make use of tools like `npx` to simply run `npx @eslint/create` or using `npm` to run `npm init @eslint` or using `yarn` to run `yarn create @eslint` single time for a project instead of having it in the core project. 

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

The implementation is mainly migrating the code from main repo to `@eslint/create`. As far as the modules are concerned, all shared modules are being shipped currently with the eslint so we can use those modules through eslint itself. 

The approach would be following these steps

- Move the `eslint/lib/init/*` to the new repo's `src/*` directory
- Add the following `dependencies` in `package.json`
  - `enquirer`
  - `progress`
  - `semver`
  - `espree` (peer)
  - `lodash`
  - `json-stable-stringify-without-jsonify`
  - `cross-spawn`
  - `eslint` (peer)
  - `debug` (peer)
- for all those modules coming inside `../**/*`, it would be replaced using `eslint/**/*`
- For `tests`, move the `tests/lib/init` to new repo's `tests/init`
- Add the following `devDependencies` in `package.json`
  - `chai`
  - `sinon`
  - `js-yaml`
  - `proxyquire`
  - `shelljs`
- create `tests/utils` and add the following functions 
  - [`defineInMemoryFs`](https://github.com/eslint/eslint/blob/6677180495e16a02d150d0552e7e5d5f6b77fcc5/tests/_utils/in-memory-fs.js#L250)
- for `conf/*`, it is not a part of the public API and it cant be accessed from `eslint` module, we need a keep these synced with the main repo (`eslint`). We can choose from the following approach to solve this : 
  - using the `eslint-github-bot`, whenever there is a PR opened that changes the file for the `conf` directory in `eslint` repo, create an issue in the `@eslint/create` repo to have a look in the PR and if required submit a change in `@eslint/create`.
  - OR, while doing the release for both `eslint` and `@eslint/create` repos, the release script can be configured to copy-parse the `conf` folder from `eslint` to `@eslint/create` before the release just like the `website` repo is being updated currently. 
- After completion of this repo, whenever someone uses the `--init` flag, an error will be thrown stating the correct step to do the migration i.e using `npx @eslint/create` or `npm @eslint` or `yarn @eslint`. I am not suggesting to use run the `@eslint/create` module whenever `--init` command is being called because this would require to use the `@eslint/create` module as a dependency in `eslint` and in `@eslint/create` we are already being using `eslint` as a dependency, so there would be circular dependencies.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
Documentation for `@eslint/create` will be created with proper usage and each prompt's details. In the main documentation, a link to `@eslint/create` will be given [here](https://github.com/eslint/eslint/blob/master/docs/user-guide/command-line-interface.md#--init) and basic usage and the package details will be documented below as a deprecated notice and migration guide. 
Also in the cli command options, [here](https://github.com/eslint/eslint/blob/master/docs/user-guide/command-line-interface.md#options), for `--init` we need to change the description for this with deprecated details as the command is still active just that there will be error that will recommend to use `@eslint/create` package instead.
Yes as it will be a major change for the `eslint` repo itself as it is being removed from the main package/repo, I guess a blog can be a good solution to solve initial queries. 

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
- This might increase the maintenance burden as this will add one more repo in the organization. 
- We may need to face challenges like re-directing of issues from `@eslint/create` repo to `eslint` and vice-versa. 
- If we do not add proper tooling for handling the `conf` directory in `eslint` to `@eslint/create` repo as mentioned in the [detailed design](@detailed_design)'s 7th point, there will be an overhead work on looking through each PR/change particularly to find out whether there is a need for updating the `conf` directory in `@eslint/create` or not. 


## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->
- Releasing with a proper message which the command is being run from `eslint` with something like - `"this command has been moved from here, use 'npx @eslint/create'"` might be a good way to address backward compatibility. 
- Or whenever `--init` command is being used, show a warning and run `npx @eslint/create` using `child_process` of the native node modules. 
- Otherwise we can keep the feature in the main repo as well for starting few days/weeks with a warning and then removing it completing with an error message but this would lead to circular dependencies and that is not an ideal case for managing dependencies, so I think we should remove this feature completely with an error message as mentioned above. 

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
