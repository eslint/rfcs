- Start Date: 2019-11-28
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Update Default Ignore Patterns

## Summary

This proposal changes the default ignore patterns to include `node_modules` in subdirectories and to exclude `.eslintrc.js`.

## Motivation

- In monorepo style development, there are `node_modules`s in subdirectories. But the current default ignore patterns don't include it, so we have to configure ESLint with `node_modules` to ignore it.
- Because ESLint ignores dotfiles, it doesn't lint `.eslintrc.js` by default. We have to configure ESLint with `!.eslintrc.js` to lint it.

## Detailed Design

This proposal changes the default ignore patterns:

- (as-is) `.*`
- (new) `!.eslintrc.js`
- (change) `/node_modules/*` → `/**/node_modules/*`
- (as-is) `/bower_components/*`

Therefore, ESLint lints `.eslintrc.js` and ignores `node_modules` in subdirectories.

### How about the config files of other tools?

For example, Babel has `.babelrc.js` and VuePress has `.vuepress/config.js`. ESLint ignores such files as dotfiles by default.

This proposal doesn't change this behavior because we don't want to have allowlist/denylist for all tools in the world. Instead, we can have unignoring patterns in the `ignorePatterns` property of shareable configs for your tools.

## Documentation

- It needs an entry in the migration guide for 7.0.0 (or another major release) because of breaking changes.
- It needs to update the "[.eslintignore](https://eslint.org/docs/user-guide/configuring#eslintignore)" section to explain the new default ignore patterns.

## Drawbacks

This proposal doesn't increase maintenance cost because it just updates a pattern list.

If somebody wants to lint `node_modules` in subdirectories, they have to configure ESLint to unignore those. But I guess that it's rarer than wanting to ignore `node_modules` in subdirectories.

## Backwards Compatibility Analysis

This is a breaking change.

- ESLint may report more warnings on `.eslintrc.js`.
- ESLint may stop with fatal errors (all files are ignored) in the `node_modules` of subdirectories.

## Alternatives

We can use the `ignorePatterns` of shareable configs to configure and reuse this kind of matter.

## Related Discussions

- https://github.com/eslint/eslint/issues/1163 - ignore node_modules folder by default
- https://github.com/eslint/eslint/issues/10089 - .babelrc.js should not be ignored by default
- https://github.com/eslint/eslint/issues/10341 - do not ignore files started with `.` by default
