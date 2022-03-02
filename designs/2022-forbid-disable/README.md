- Repo: eslint/eslint
- Start Date: 2022-02-28
- RFC PR:
- Authors: Brian Bartels

# ESLintRC Option To Enable noInlineConfig For Specific Rules

## Summary

Allow users to forbid users from configuring certain rules inline. This is an extension to `noInlineConfig` where the primary difference is that `noInlineConfig` currently is set for all rules.

## Motivation

When working with complex lint configs. Some rules are more important than other. For this reason it can be inconsequential to disable certain rules. But incredibly bad to disable others.

EG:
```
//eslint-disable-next-line @government/required-SECURITY-rule
```
Is a much worse rule to disable than
```
//eslint-disable-next-line noExtraSemi
```

`noInlineConfig` currently works well for some. But in reality, many users disable rules at least somewhat frequently for valid reasons. And in large repos, there are going to be exceptions that need inline configuration. 

## Detailed Design

I will add the option to pass `noInlineConfig` an array of rule names. This is in addition to the already supported boolean setting. This will result in those rules having inline configuration disabled.

This means that I will need to modify `config-schema.js`
```
noInlineConfig: { type: ["boolean", "array"] },
```

EG `// eslint-disable semi` will have ~~semi~~ as an error, as well as the violation that it is attempting to disable also having an error.

Both errors will have a severity of 2. See [LintMessage#severity](https://eslint.org/docs/developer-guide/nodejs-api#-lintmessage-type)

first in `getDirectiveComments` I will match any directives that affect a rule listed in the array `noInlineConfig`. If there is a match, I will create a problem on that line to alert the user that they are attempting to disable a rule that will not be affected by the inline config.

This error will **not** have a quick fix available


In `applyDirectives` (`apply-disable-directives.js`) I will reference the problem. If that problem matches a name in the array `noInlineConfig` I will not apply any directives. This will ensure that `problems.push(problem)` gets run. Note that this same logic applies to configuration comments as well.

## Documentation

We will want to update the documentation ([here](https://eslint.org/docs/user-guide/configuring/rules#disabling-inline-comments)) to reflect the new changes. The announcement can be integrated into the config update announcement.

## Drawbacks

this adds complexity to the already complex config experience.

This will have a very minor impact on perf.

This design does not alert users in the case where they user `// eslint-disable` because that directive has `ruleId: null`. However, the design will still appropriately enforce the proper rules as desired.

## Backwards Compatibility Analysis

This should integrate into the planned config changes and therefore will not be backwards compatible. However, this is acceptable since that will be a breaking change that is already approved and planned.

## Alternatives

instead of extending `noInlineConfig` we could instead add a state called `enforce` to `error/warn`. This would be a superset of `error` where the rules cannot be disabled by inline config. However, it is hard to convey to the user succinctly what this accomplishes. And it allows users to disable the rules more easily in config files

There are plugins such as [eslint-comments/no-restricted-disable](https://mysticatea.github.io/eslint-plugin-eslint-comments/rules/no-restricted-disable.html) that implement this concept. However, since these are just other rules, one can easily disable them. This results in scenarios such as

```
// eslint-disable-next-line eslint-comments/no-restricted-disable
// eslint-disable no-undef
```
In large PRs, reviewers often miss these comments. In a large repo that I work on these scenario has happened over 30 times

Another alternative that requires 0 code is to encourage users to set up a second eslintrc and run lint twice. Once normally with regular rules, and once with `no-inline-config` with just the rules that are restricted. However, **this will not integrate into vscode error reporting**. Which will result in users pushing code to PR phase only to fail a gate that checks for restricted disables. This solution also results in a lot of bloat and configuration overhead for anyone who wants this functionality.

## Open Questions

- How can I gracefully integrate into the config updates?
- How can I test runtime to ensure that this doesn't affect perf?
- What is the testing strategy that ESLint community prefers?

## Help Needed

- I can implement this independently

## Frequently Asked Questions

- Why do PR approvers approve PRs with `no-eslint-disable` rule?
    - because PR reviewers are lazy.
## Related Discussions

- [Initial issues conversation](https://github.com/eslint/eslint/issues/15631)
