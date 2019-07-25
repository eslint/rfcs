- Start Date: 2019-07-25
- RFC PR: https://github.com/eslint/rfcs/pull/35
- Authors: Eric Wang

# Pass cwd from cli engine

## Summary
Make the CLIEngine `cwd` option available to rules to avoid the misalignment between it and process.cwd()

see: https://github.com/eslint/eslint/issues/11218

## Motivation
To avoid the misalignment between it and process.cwd().
There is a situation where a rule need to get the relative path of the current file to the root folder of the project.
But it's hard to guarantee that all developers will open the root folder in their IDE (VSCode with eslint plugin for example), especially when the project grows.

Use case:
Say, the project is supposed to be open in `web` folder.
But as the project grows, people tends to open some subfolder under the `web` to minimize sidebar.
`eslint` plugin in VSCode will then run the `eslint` from the subfolder which cause the `process.cwd` not to return the expected value (e.g. the `web` folder)

Expected:
It is expected that the rule has access to the `cwd` in the option of the CLI engine.

## Detailed Design
Refactor the `Linter` constructor to accept a nullable string parameter `cwd`.
Refactor the `runRules` method in `linter.js` to accept the nullable string parameter `cwd`, and pass it to the shared context so the rule can access it via `context`.
If the `cwd` is `undefined`, `process` exists, it will be `process.cwd()`.
If both of the `cwd` and the `process` are undefined, it will be undefined.

https://github.com/eslint/eslint/pull/12021

## Documentation
Adding the `cwd` into the `context` Api.
Adding the `cwd` into the `Linter` constructor Api.

## Drawbacks
Can't think of any.

## Backwards Compatibility Analysis
Those rules using `process.cwd` will have exact same behavior, so there is no compatibility issue.

## Alternatives
N/A

## Open Questions
N/A

## Help Needed
N/A

## Frequently Asked Questions
N/A

## Related Discussions
N/A
