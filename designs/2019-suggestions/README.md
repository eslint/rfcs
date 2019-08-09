- Start Date: 2019-07-04
- RFC PR: https://github.com/eslint/rfcs/pull/30
- Authors: Ilya Volodin ([@ilyavolodin](https://github.com/ilyavolodin)), Dan Abramov ([@gaearon](https://github.com/gaearon))

# Code Suggestions

## Summary

This RTC proposes introducing a new mechanism for providing users with suggestions. It's very similar to fixes mechanism that already exists in ESLint, but will provide ability to create multiple suggestions per rule, suggestions will not be automatically applied by using `--fix` flag, and will require user interaction. This feature will only be exposed through the ESLint API, and will not be available on the CLI.

## Motivation

This has been suggested multiple times before, but the main driving force behind this proposal is ability to provide user with options on how to automatically refactor/fix their code, without a possibility of "stealthily" breaking it. Examples of such suggestions include ability to suggest adding `import` statement if certain code is added to a file by the user, ability to provide refactoring suggestions like extracting repeated value into a variable, reformatting code to extract repeated statements into reusable function, etc. I do not envision ESLint team to initially create any rules that would provide suggestions, so this would be a feature that would enable plugin authors to do so, but it might be something that ESLint team will eventually do. This feature is geared towards code editor integrations, as such, it will only be available via NodeJS API.

## Detailed Design

Suggestion will be applied as a stand-alone change, without triggering multipass fixes. Each suggestion should focus on a singular change in the code and should not try to confirm to user defined styles. For example, if suggestion is adding a new statement into the codebase, it should not try to match correct indentation, or confirm to user preferences on presence/absence of semicolumns. All of those things will be corrected by multipass autofix when the user triggers it. It should be suggested that if a user wants to use suggestions, he/she should also enable auto-fix on save in their editor of choice.

* Modify `context.report` function parameter to include new property called `suggest` of type array of objects. Each object will contain a description of the suggestion and a function that will be able to modify the code of the file to implement suggestion. Syntax is going to be exactly the same as for `fix` functions. Example of new signature below:
```js
context.report({
    node,
    loc,
    messageId: 'messageId',
    fix(fixer) { },
    suggest: [
        { desc: 'Description of the suggestion', fix: (fixer) => { }},
        { desc: 'Description of the suggestion', fix: (fixer) => { }}
    ]
});
```
* Change `_verifyWithoutProcessors` and `_verifyWithProcessor` functions in linter.js to modify output of those functions to include `suggestions` array returned by the rule. After modification the signature of the return object should look like this:
```js
interface LintMessage {
    //...(existing props)...
    suggestions: Array<{ desc: string, fix: { range: [number, number] } }>
}
```
* Modify output of `CLIEngine` `executeOnFiles` and `executeOnText` to include `suggestions` array returned by the rule. If `options.disableFixes` are passed into those functions, return result should not include any suggestions return by the rules.
* Update `docs` section of rule metadata to include `suggestion: true|false` to make it easier to identify rules that can return suggestions.
* Unlike fixes, suggestion will not expose a new API function on `CLIEngine` to output selected suggestion to a file. Instead, integration would be expected to apply suggestion in memory and let the user save the file on their own.

## Documentation

This would need to be documented as part of the public API.
* [Node.js API](https://eslint.org/docs/developer-guide/nodejs-api) page needs to be updated with corresponding changes to `linter` and `CLIEngine`.
* [Working with Rules](https://eslint.org/docs/developer-guide/working-with-rules) page needs to be updated to show examples of returning suggestions. It would also need to include changes to metadata to identify rules that return suggestions.
* Update rule documentation template to clearly show rules that return suggestions.

## Drawbacks

* This might potentially be a bit confusing, since we do not have many functions that we provide through API only and not through CLI.
* Suggestions might lead to user created rules that try to do too much modifications to the user's code. For example suggesting large refactoring that might introduce a lot of breaking changes. To mitigate this, we should add documentation that would warn rule creators against doing that.

## Backwards Compatibility Analysis

Since `suggestions` property in `context.report` object is optional, this should be fully backwards compatible.

## Alternatives

This functionality is basically the same as existing `autofix` functionality already included in ESLint. The only difference is there might be multiple suggestions per error and they are not applied automatically by ESLint.

## Open Questions

Should we consider instead providing new type of `rules` that would not report any issues, but only provide suggestions?

## Related Discussions

https://github.com/eslint/eslint/issues/7873
https://github.com/eslint/eslint/issues/6733#issuecomment-234656842
