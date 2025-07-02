- Repo: eslint/eslint
- Start Date: 2025-06-11
- RFC PR: (leave this empty, to be filled in later)
- Authors: Nicholas C. Zakas

# Allow rules to specify the languages/dialects they work on

## Summary

This RFC proposes adding metadata to ESLint rules that indicate which programming languages and language dialects they support. This will enable better documentation and runtime capabilities for ESLint when working with multiple languages.

## Motivation

Currently, an ESLint rule has no way to indicate which languages or language dialects it is designed to work with. This limitation creates two significant problems:

1. **Documentation purposes** - Users have no easy way to determine which JavaScript rules have been updated to support TypeScript syntax, for example. This makes it harder to understand which rules can be safely enabled when linting TypeScript code.

2. **Runtime purposes** - ESLint cannot automatically disable rules that don't apply to a given language. For instance, when linting CSS files, JavaScript-specific rules should ideally be automatically turned off, but currently there's no mechanism to do this.

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

1. Update the `RuleMeta` interface in `@eslint/core` to accept `languages`. Deprecate the `language` and `dialects` properties.
2. Update the `validateRulesConfig()` function in `lib/config/config.js` to validate each rule's `languages` property against the language specified in the `Config` instance. When a rule doesn't match the language, set it's severity to `0`. Normalize `"js/js"` to `"@/js"`.

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

2. We could throw an error when a rule doesn't match the language instead of turning it off. This would give the user more information about what's happening but could also lead to frustration if they can't figure out where a rule is configured.

## Open Questions

1. What should we do about `meta.dialects`? Assuming we still want some way to programmatically know which rules support which languages for documentation purposes, should we just move this into `meta.docs`?


## Help Needed

None.

## Frequently Asked Questions

**Will this feature automatically fix compatibility issues between rules and languages?**  

No, this feature only provides metadata about compatibility; it doesn't make incompatible rules work with new languages. Rule authors will still need to update their rule implementations to support additional languages.

**How will this affect existing ESLint configs?**  

Existing configs will continue to work as before. However, users may notice that some rules are automatically disabled when linting files with language plugins that those rules don't support.

**How are language identifiers determined?**  

Language identifiers follow the standardized format `"plugin/language"`. The validation will first look for a direct string match. If one isn't found, then it will search through the registered plugins and look for a `meta.namespace` that matches the first part of the language string. This ensures that users who use a different namespace in their config can still have languages match.

## Related Discussions

- [#19462: Allow rules to specify the languages/dialects they work on](https://github.com/eslint/eslint/issues/19462)
