- Repo: eslint/eslint
- Start Date: 2021/1/12
- RFC PR: https://github.com/eslint/rfcs/pull/75
- Authors: [A-Katopodis](https://github.com/A-Katopodis)

# (Break on parsing errors)

## Summary

The suggest change is ESLint support a parameter that will break the ESLint run when it finds configuration errors. The option will be an opt-in argument called `--break-on-error`.

## Motivation

We met with a couple of cases where we assumed a succesfull run of ESLint in CI enviroments. When ESLint wouldn't be able to read the tsconfig.json or had a wrongly configured source type the command would report all errors in each file and also exit with a 1 exit code.

According to the eslint docs about exit codes:

- 0: Linting was successful and there are no linting errors. If the --max-warnings flag is set to n, the number of linting warnings is at most n.
- 1: Linting was successful and there is at least one linting error, or there are more linting warnings than allowed by the --max-warnings option.
- 2: Linting was unsuccessful due to a configuration problem or an internal error.

Being able to exit with code `2` instead of `1` it will allow for CI pipelines to better understand the results.

## Detailed Design
Adding a new option in ESLint called `--break-on-error` which will be  which will report and nonzero exit code on case of parsing errors. The most current similar option is `--max-warning`.

A command example:

`eslint **.js --break-on-error`

If this command finds any kind of file which may use ECMAScript modules it will report a non-zero exit code (since the default for `sourceType` is script.)

Without `--break-on-error` (the current behavior) a user of ESLint would have to look at each and every result to distinquish if there were any kind of missconfiguration/parsing errors as having a normal error in a file is reported the same way as a parsing error. This validation becomes harder for CI pipelines who want to ensure that ESLint reports any rules correctly and was run sucessfully.

The expected behavior of the command should gather be able to read the reported errors that eslint reports. If at least one missconfiguration is found it exits with a non-zero exit code, 2 so we can distinguish it from the exit code 1.

## Documentation
It may be a good idea on why we are introducing a option that changes the way ESLint behaves. Will leave the choice to the ESLint team.


## Drawbacks
Some users who may want to enable this new feature may find themselves needing to reconfigure their process. It can change some CI pipelines if used. But since its optional the impact is minimal and it may allow them to discover issues.


## Backwards Compatibility Analysis
There are no expected backward compatibility issues. The parameter will be disabled by default and only users who will use this new parameter will experience any kind of change.


## Open Questions
Do we want to still report all errors or only failed linting errors when `break-on-lint-error` is passed?

Assume that for some files ESLint successfully lints them and reports a rule but for some others it doesnt.
If ESLint finds a single file that has a parsing error should it report just that file or every rule as 
well?

## Related Discussions
https://github.com/eslint/eslint/issues/13711
