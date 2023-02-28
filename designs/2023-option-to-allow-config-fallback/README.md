- Repo: eslint/eslint
- Start Date: 2023-02-28
- RFC PR: (leave this empty, to be filled in later)
- Authors: (the names of everyone contributing to this RFC)

# Option to allow config fallback

## Summary

A "defaultConfigFile" option to the API (ESLint/FlatESLint) that would allow an API user to specify a default config file to use if none is found through the normal lookup.

## Motivation

For those who use "vscode-eslint" extension to add rules installed globally and desire to keep working on files on the fly on their machine without having to install the rules every single time.

In case they have to work on a project with local rules, the configuration setup from "defaultConfigFile" would be replaced with the local configuration found in the project.

## Detailed Design

Currently there is an option named "overrideConfigFile" that fulfills the desire to have a global configuration with the desired rules and plugins. However, when it is required to work on a project with its own configuration, it will continue to use the configuration from "overrideConfigFile". This lead to comment out the option from vscode settings in order to be able to work on local rules and uncomment it out when it is required to work out of the project.

With a "defaultConfigFile" option, we skip the manual process of commenting out "overrideConfigFile" in vscode settings by following the next:

If any eslint config or config file is not found on root and up directories, use the file declared in "defaultConfigFile".

## Documentation

Since this is a very simple use case, this can be announced as a new option avaiable with a brief description, similar to "overrideConfigFile".

## Drawbacks

This is meant to be used on rules installed globally, so it might generate confusion to users who want to consume this option.

## Backwards Compatibility Analysis

There is no side effects on the behavior of other options. However, there should be a clear description as it might get confused with the "overrideConfigFile" option.

## Alternatives

"defaultConfigFile" looks a good option name. However, if the team consider there is another proper name having the 

## Help Needed

The team would be the best qualified to implement the feature.

## Related Discussions

- <https://github.com/eslint/eslint/issues/16828> - the issue triggering this RFC
