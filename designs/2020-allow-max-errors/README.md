- Repo: [eslint/eslint](https://github.com/eslint/eslint)
- Start Date: 2020-08-25
- RFC PR: (leave this empty, to be filled in later)
- Authors: @josmo

# Allow max number of errors before cli returns error exit code

## Summary

This RFC proposes adding a similar command option to --max-warnings for errors

## Motivation

Currently, when trying to implement eslint in legacy projects which might
have significant numbers of warnings and errors we can set a max limit on
warnings to ensure we don't introduce more in the code base.  There is
currently no equivalent for errors for ensuring no new errors get introduced. 

## Detailed Design

We should update lib/options.js to include
```js
{
  option: "max-errors",
  type: "Int",
  default: "0",
  description: "Number of errors to allow before triggering nonzero exit code"
}
```
as well as update lib/cli.js to return a 0 exit code if the errors are below the max
number of errors

[This Draft PR](https://github.com/eslint/eslint/pull/13617) currently has the
implementation, documentation and tests which match the max-warnings
implementation with the following differences.
1. default is 0 - keeps default behavior with no breaking changes
1. defaults to error if no max-error set.


## Documentation

docs/user-guide/command-line-interface.md in the eslint repo needs to be updated
to reflect the change. The [Draft PR](https://github.com/eslint/eslint/pull/13617)
has the changes which would be required.


## Drawbacks

Philosophical we shouldn't allow errors in the code base. If something is an error then it should
be fixed right away and not allowed to pass.


## Backwards Compatibility Analysis

There shouldn't be any backward compatibility issues.

## Alternatives

1. Plugin to change all errors to warnings - This would work, however it doesn't give
the project a chance to have a choice of what are warnings and what are errors. Everything
would be considered a warning.
1. Manually mark exceptions and or all rules to be warnings - This doesn't allow for a
slower adoption and requires a lot of upfront work.

