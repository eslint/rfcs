- Repo: eslint/eslint
- Start Date: 2020-05-23
- RFC PR:
- Authors: Shahar Dawn Or (@mightyiam)

# Do not exclude dot files by default

## Summary

ESLint excludes dot files (files whose names begin with `.`) by default. This is a proposal to remove this default exclusion.

## Motivation

Exclution of dot files by default does not seem like a reasonable default. Dot files that are JavaScript files seem common in JavaScript projects. It feels safe to assume that many projects have potential lint errors in such files, because they don't know about this default exclusion.

Removing this exclusion would effectively turn on linting for dot files in existing and future usages. This feels like reasonable default behavior.

## Detailed Design

Remove the ignore pattern for dot files from the default configuration.

## Documentation

It will be announced in the release notes of the major version it is introduced in.

## Drawbacks

See next chapter.

## Backwards Compatibility Analysis

Many existing usages are expected to fail linting, forcing them to fix and/or ignore the new lint errors in newly linted dot files. Or ignore the dot files.

No attempt to minimize disruption other than announcing the change in the release notes.

## Alternatives

Alternatives have not come to mind.

## Open Questions

Why is ESLint currently ignoring dot files by default?

Does ESLint have a test that is run against many public usages, where statistics of this change could be obtained?

## Help Needed

Where is the dot files pattern in the implementation code and which tests would have to be modified, please?

## Frequently Asked Questions

Q: Do you have real examples of JavaScript dot files in projects where the authors are under the false assumption that those files are linted, where in truth, the are not?

A: [Prettier](https://prettier.io/) is [used by ~1.7m projects](https://github.com/prettier/prettier/network/dependents). One of the [default configuration file names](https://prettier.io/docs/en/configuration.html) is `.prettierrc.js`. How many of the usages of Prettier use that file name? How many of them _do not_ explicitly include dot files? How many of them are not aware that that file is not linted? Prettier is only one example. I did not attempt to answer these questions.

## Related Discussions

Issue [#13342](https://github.com/eslint/eslint/issues/13342), which is a follow-up from issue [#10341](https://github.com/eslint/eslint/issues/10341).
