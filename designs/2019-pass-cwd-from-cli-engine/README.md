- Start Date: 2019-07-25
- RFC PR: https://github.com/eslint/eslint/pull/12021
- Authors: Eric Wang

# Pass cwd from cli engine

## Summary
Make the CLIEngine `cwd` option available to rules to avoid the misalignment between it and process.cwd()

see: https://github.com/eslint/eslint/issues/11218

## Motivation
To avoid the misalignment between it and process.cwd().

Use case:
Say, the project is supposed to be open in `web` folder.
But as the project grows, prople tends to open some subfolder under the `web` to minimize sidebar.
`eslint` plugin in VSCode will then run the `eslint` from the subfolder which cause the `process.cwd` not to return the expected value (e.g. the `web` folder)

Expected:
It is expected that the rule has access to the `cwd` in the option of the CLI engine.

## Detailed Design
It could be easier to look at the code directly, as it has 15 LOC
https://github.com/eslint/eslint/pull/12021

## Documentation
I don't think it's a big change.
Adding the `cwd` into the `context` Api should be enough.

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
