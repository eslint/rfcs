- Repo: eslint/eslint
- Start Date: 2025-06-11
- RFC PR: https://github.com/eslint/rfcs/pull/135
- Authors: Nicholas C. Zakas

# Allow rules to specify the languages/dialects they work on

## Summary

This RFC proposes adding metadata to ESLint rules that indicate which programming languages and language dialects they support. This will enable better documentation and runtime capabilities for ESLint when working with multiple languages.

## Motivation

Currently, an ESLint rule has no way to indicate which languages or language dialects it is designed to work with. This limitation creates two significant problems:

1. **Documentation purposes** - Users have no easy way to determine which JavaScript rules have been updated to support TypeScript syntax, for example. This makes it harder to understand which rules can be safely enabled when linting TypeScript code.

2. **Runtime purposes** - Currently, rules that don't apply to a given language crash in unpredictable ways, causing confusion for users. Ideally, ESLint would throw an error with a descriptive message stating that these rules cannot be used with the given language.

As ESLint's ecosystem expands to support more languages beyond JavaScript (such as TypeScript, CSS, and potentially others), having a standardized way to specify language compatibility becomes increasingly important for both users and maintainers.

## Detailed Design

I propose adding a `languages` array to the rule's `meta` object. Example:

```js
// Rule definition example
module.exports = {
  meta: {
    type: "suggestion",
    docs: {
      description: "Disallow something",
      recommended: true
    },
    
    // new property
    languages: ["markdown/gfm", "markdown/commonmark"] // Languages the rule supports
  },
  create(context) {
    // Rule implementation
    return {};
  }
};
```

The `languages` array will contain strings that identify the specific language plugins the rule is designed to work with. Each string follows the standardized format `"plugin/language"` to uniquely identify the language plugin. Special syntax:

- To specify that a rule works with any language in a plugin, the format of `"plugin/*"` is used.
- To specify that a rule works for any language, `"*"` is used. 

For backward compatibility, if `languages` is not specified, the rule will be assumed to work with all languages. (Effectively, the same as `languages: ["*"]`).

For rules meant to work only with JavaScript, the `"js/js"` string is used. In the short-term, we'll need to special case this to match `"@/js"`, which is how the JavaScript language is defined right now. In the long-term, once `@eslint/js` fully contains the language, we can remove the check.

### Implementation Approach

1. Update the `RuleMeta` interface in `@eslint/core` to accept `languages`. Deprecate the `language` and `dialects` properties. Add `meta.docs.dialects` property.
2. Update the `validateRulesConfig()` function in `lib/config/config.js` to validate each rule's `languages` property against the language specified in the `Config` instance. When a rule doesn't match the language, add to an array of invalid rules. When all validation is complete, if there is an array of invalid rules, throw an error. Normalize `"js/js"` to `"@/js"`.

## Documentation

We will need to update these pages:

* [Custom rules](https://eslint.org/docs/latest/extend/custom-rules)
* [Custom rule tutorial](https://eslint.org/docs/latest/extend/custom-rule-tutorial)

## Drawbacks

There are a few potential drawbacks to this approach:

1. **Added Complexity**: Adding another property to the rule metadata increases the complexity of rule creation and maintenance.

2. **Maintaining Accuracy**: Rule authors will need to remember to update the language metadata when they enhance a rule to support additional languages, which could lead to outdated or incorrect information.

3. **Migration Effort**: Existing rules will need to be updated to include this metadata, which represents a non-trivial effort for the core team and plugin authors.

4. **Edge Cases**: Some rules might work with multiple language plugins but have different behaviors or limitations, which might not be fully captured by a simple list of supported languages.

## Backwards Compatibility Analysis

This RFC is designed to be fully backward compatible:

1. **Default Behavior**: Rules without the `languages` property will work with all languages, which matches the current behavior with JavaScript.

2. **No API Changes**: No existing APIs are modified or removed, so current tooling will continue to work.

3. **Gradual Adoption**: Plugin authors can add this metadata to their rules at their own pace, without breaking existing functionality.

4. **No Configuration Changes**: Users will not need to update their configurations to benefit from this feature.

## Alternatives

1. We could keep the existing `language` and `dialects` properties and then add keys to plugin metadata specifying the languages and dialects they support. This has compatibility issues as two plugins could provide languages that say they support Markdown but do so in completely different ways. Thus, we can't guarantee the rule will work in both plugins if we just go by the language and dialect.

2. We could disable any rule that doesn't match the language instead of throwing an error. The downside is that this doesn't inform the user of any problem and they might wonder why a rule wasn't executed even though it was configured to do so.

## Open Questions

None.

## Help Needed

None.

## Frequently Asked Questions

**Will this feature automatically fix compatibility issues between rules and languages?**  

No, this feature only provides metadata about compatibility; it doesn't make incompatible rules work with new languages. Rule authors will still need to update their rule implementations to support additional languages.

**How will this affect existing ESLint configs?**  

Existing configs will continue to work as before.

**How are language identifiers determined?**  

Language identifiers follow the standardized format `"plugin/language"`. The validation will first look for a direct string match. If one isn't found, then it will search through the registered plugins and look for a `meta.namespace` that matches the first part of the language string. This ensures that users who use a different namespace in their config can still have languages match.

## Related Discussions

- [#19462: Allow rules to specify the languages/dialects they work on](https://github.com/eslint/eslint/issues/19462)
