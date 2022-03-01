- Repo: eslint/eslint
- Start Date: 2022-02-28
- RFC PR:
- Authors: Brian Bartels

# ESLintRC Option To Forbid Disable For Specific Rules

## Summary

Allow users to forbid users from disabling certain rules. This is similar to `noInlineConfig` where the primary difference is that `noInlineConfig` is set for all rules.

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

`noInlineConfig` works well for some. But in reality, many users disable rules at least somewhat frequently for valid reasons. And in large repos, there are going to be exceptions that need inline configuration. 

## Detailed Design

In `eslintrc` I will add another option for rules. The new state can be
```
"warn"
"error"
"forbid"
```
where forbid is always an error and cannot be overridden by a directive

EG `// eslint-disable semi` will have ~~semi~~ as an error, as well as the violation also having an error.

Both errors will have a severity of 2. See [LintMessage#severity](https://eslint.org/docs/developer-guide/nodejs-api#-lintmessage-type)

I will add a new array to the linter called `forbiddenRules` that will be processed on initialization of the linter.js. This will happen in the Constructor. Then in `getDirectiveComments` I will reference the rule being disabled in the directive. If that rule matches a rule in `forbiddenRules` then I will not accept that directive. And I will declare an error on that string. Note that this same logic applies to configuration comments as well.

in `linter.js` I will adjust the `getDirectiveComments` function so that I reference a list of forbidden rules. If someone is disabling a rule that is forbidden. I will add an error similar to `warnInlineConfig`

note that when disabling all rules such as `/* eslint-disable */` the directive object will have `ruleId: null`. This will need a workaround. 

This error will **not** have a quick fix available


## Documentation

We will want to update the documentation to reflect the new changes. The announcement can be integrated into the eslintrc update announcement.

## Drawbacks

this adds complexity to the already complex eslintrc experience. Users could be confused by the interaction with `forbid` and `noInlineConfig`.

This will have a very minor impact on perf.

## Backwards Compatibility Analysis

This should integrate into the planned eslintrc changes and therefore will not be backwards compatible. However, this is acceptable since that will be a breaking change that is already approved and planned.

## Alternatives

instead of adding a state to `error/warn` we could instead add another optional field to the rule config. This would enable a rule to be `warn` and have directives disabled. However, I believe that this actually adds more bloat to eslintrc which is why I chose the above design

There are plugins such as [eslint-comments/no-restricted-disable](https://mysticatea.github.io/eslint-plugin-eslint-comments/rules/no-restricted-disable.html) that implement this concept. However, since these are just other rules, one can easily disable them. This results in scenarios such as

```
// eslint-disable-next-line eslint-comments/no-restricted-disable
// eslint-disable no-undef
```
In large PRs, reviewers often miss these comments. In a large repo that I work on these scenario has happened over 30 times

Another alternative that requires 0 code is to encourage users to set up a second eslintrc and run lint twice. Once normally with regular rules, and once with `no-inline-config` with just the rules that are restricted. However, **this will not integrate into vscode error reporting**. Which will result in users pushing code to PR phase only to fail a gate that checks for restricted disables. This solution also results in a lot of bloat and configuration overhead for anyone who wants this functionality.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

- How can I gracefully integrate into the eslintrc updates?
- How can I test runtime to ensure that this doesn't affect perf?
- What is the testing strategy that ESLint community prefers?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->
- I can implement this independently
- Will the RFC still need to be open for 21 days if the main contributors all chime in?

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->
- Why do PR approvers approve PRs with `no-eslint-disable` rule?
    - because PR reviewers are lazy.
## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
- [Initial issues conversation](https://github.com/eslint/eslint/issues/15631)
