- Repo: [eslint/eslint](https://github.com/eslint/eslint)
- Start Date: 2020-06-29
- RFC PR: https://github.com/eslint/rfcs/pull/79
- Authors: Aniketh Saha ([anikethsaha](https://github.com/anikethsaha)) aladdin-add(weiran.zsd@outlook.com)

# Move --init flag into a separate utility

## Summary

- Remove the auto-config.
- Move the `init` command from main repo ([eslint](https://github.com/eslint/eslint)) to a new repo named `@eslint/create-config`.

## Motivation

Currently the whole `init` command is being shipped with the cli in the main repo. Though its not that much of size (roughly ~0.6MB for the [eslint/lib/init](https://github.com/eslint/eslint/tree/master/lib/init)), this command is not a type of command that we need everyday or everytime running eslint. It is mainly used when creating a new project or adding eslint to a project for the first time. So if we move this to a separate package that is meant to use in the command line,
We will make use of tools like `npx` to simply run `npm init @eslint/config` or using `npm` to run `npx  @eslint/create-config` or using `yarn` to run `yarn create @eslint/config` single time for a project instead of having it in the core project.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

The implementation is mainly migrating the code from main repo to `@eslint/create-config`.

The approach would be following these steps

### 1. Remove eslint auto-config

- Remove auto-config(`eslint/lib/init/autoconfig.js`), and its tests(`eslint/tests/lib/init/autoconfig.js`).
- Remove dependency `progress`, as it's no longer used in production.

### 2. Move `eslint --init` related files to a separate repo

- Move the `eslint/lib/init/*` to the new repo's `lib/*` directory
- Add the following `dependencies` in `package.json`

  - `enquirer`
  - `progress`
  - `semver`
  - `lodash`
  - `json-stable-stringify-without-jsonify`
  - `cross-spawn`
  - `debug`
  - `@eslint/eslintrc`
  - `eslint` (to be removed)
  - `espree` (to be removed)

- for all those modules coming inside `../**/*`, it would be replaced using `eslint/**/*`
- For `tests`, move the `tests/lib/init` to new repo's `tests/lib/`
- Add the following `devDependencies` in `package.json`

  - `chai`
  - `sinon`
  - `js-yaml`
  - `proxyquire`
  - `shelljs`

It was almost done: [aladdin-add/eslint-create](https://github.com/aladdin-add/eslint-create).
I will make a PR once the official repo is created later.

### 3. Remove the usage of eslint & espree

eslint was used:
https://github.com/aladdin-add/eslint-create/blob/c27509cbbaea7c846fd9d7a373feece5254ac8d8/lib/config-file.js#L86

It can be removed by using the local installed eslint, and switch to the new `Linter` api.

espree was used:
https://github.com/aladdin-add/eslint-create/blob/c27509cbbaea7c846fd9d7a373feece5254ac8d8/lib/config-initializer.js#L161

It can be removed after https://github.com/eslint/espree/issues/495 get landed.

### 4. Release the new package

### 5. Update eslint repo

- Remove `eslint --init` related files.
- the following dependencies can be removed in eslint repo:
  - enquirer
  - semver
- Whenever `--init` command is being used, show a warning(e.g. "You can also run this command directly using 'npm init @eslint/config'") and run `npm init @eslint/config` using `child_process` of the native node modules.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
- [get-started](https://eslint.org/docs/user-guide/getting-started).
- Documentation for `@eslint/create-config` will be created with proper usage and each prompt's details. In the main documentation, a link to `@eslint/create-config` will be given [here](https://github.com/eslint/eslint/blob/master/docs/user-guide/command-line-interface.md#--init) and basic usage and the package details will be documented.
  Also in the cli command options, [here](https://github.com/eslint/eslint/blob/master/docs/user-guide/command-line-interface.md#options), for `--init` we need to change the description.
- A formal announcement in not needed, as it is not a breaking change.

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
- We may need to face challenges like re-directing of issues from `@eslint/create-config` repo to `eslint` and vice-versa.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

- Users can use `eslint --init`, which is as the same as current.
- Users can no longer use auto-config.

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

- Do we want to implement this as monorepo under `eslint` repo or a separate repo `@eslint/create-config` but github will make it as `create-config` or similar ?

  We decided not to go in the monorepo direction for this.
