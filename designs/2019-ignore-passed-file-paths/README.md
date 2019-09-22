-   Start Date: 2019-09-22
-   RFC PR: _not yet created_
-   Authors: Fabian (fabb)

# Add CLI Option to Ignore Passed File Paths

## Summary

A new CLI option should be added that causes ESLint to respect the ignore patterns in `.eslintignore` even for file paths that are passed directly to the CLI.

## Motivation

Tools like [lint-staged](https://github.com/okonet/lint-staged) allow to run ESLint and other CLI tools on each commit on all changed files by passing the file paths directly as parameters to the tools. For example in the case the files `file1.js` and `file2.js` are changed, lint-staged will call `eslint —fix file1.js file2.js`.

The problem is that the git repo might contain files that should be ignored by ESLint via `.eslintignore`. In case such a file is changed and committed, lint-staged will pass its path to the ESLint CLI nonetheless. ESLint currently works in a way where it ignores `.eslintignore` for files explicitly passed to the ESLint CLI. That causes the file to be linted, and in case of lint-staged, will prevent the git commit to succeed if errors are found.

## Detailed Design

A new CLI option `--force-ignore` should be added to the ESLint CLI. When this option is given, ESLint should ignore files according to `.eslintignore` even when files are passed to the ESLint CLI explicitly.

## Documentation

The option should be added to the `eslint -h` output and an explanation should be added to the [user guide](https://eslint.org/docs/user-guide/command-line-interface) with a hint on the use case of lint-staged. A blog post should not be necessary.

## Drawbacks

The option might be confusing to users at first. The user guide should resolve the confusion though.

## Backwards Compatibility Analysis

This is a purely additive change, therefore completely backwards compatible.

## Alternatives

### Change the default of ESLint to ignore explicitly passed file paths when they are ignored in `.eslintignore`

This would be a non-backwards-compatible change. It could be more intuitive though. The current reason that ESLint still lints ignored files when passed explicitly is, that sometimes users might want to do that manually. In these cases, ESLint could output an info message like "Explicitly passed file1.js has been ignored by .eslintigore, if you want to lint it, use the command line option --force-lint".

## Open Questions

-   Is `--force-ignore` a good parameter name, or are there better suggestions?
-   Should rather ESLint's default be changed to ignore explicitly passed file paths when they are ignored in `.eslintignore`, instead of introducing a new option? Or Maybe change the default but provide an option for opt-in to the old behavior?

## Frequently Asked Questions

### Why not put the logic into lint-staged and make it read the `.eslintignore`?

lint-staged is tool-agnostic. It will execute any tool that users configure in their package.json on the committed files, may it be ESLint, StyleLint, prettier or even a custom script. It makes no sense if it has to know how each tool ignores files.

## Related Discussions

-   https://github.com/eslint/eslint/issues/12206
