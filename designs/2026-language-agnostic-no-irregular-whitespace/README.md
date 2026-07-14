- Repo: eslint/eslint
- Start Date: 2026-07-14
- RFC PR: https://github.com/eslint/rfcs/pull/150
- Authors: xbinaryx

# Make no-irregular-whitespace Language-Agnostic

## Summary

Make the `no-irregular-whitespace` rule language-agnostic to prevent crashes on non-JavaScript files (e.g., CSS, JSON, Markdown), while accurately reporting irregular whitespace where applicable.

## Motivation

The introduction of Language Plugins enables ESLint to natively lint non-JavaScript languages such as CSS, JSON, and Markdown. However, running the core `no-irregular-whitespace` rule on files parsed by these plugins causes ESLint to crash with:
`TypeError: Error while loading rule 'no-irregular-whitespace': sourceCode.getAllComments is not a function`.

This crash occurs because the rule assumes that the `SourceCode` object will always expose a `getAllComments()` method, which is JavaScript-specific and not implemented by the `@eslint/css`, `@eslint/json`, or `@eslint/markdown` language plugins. Furthermore, the rule assumes specific JS-specific AST node types (like `Literal`, `TemplateElement`, `JSXText`) to filter out whitespace inside strings and templates, and assumes the root AST node is always `Program`.

The goal is to make this rule language-agnostic in its basic operation, ensuring it checks for irregular whitespace characters across any language without crashing. This requires unifying how rules access comment nodes by standardizing on a `comments` property across all ESLint language plugins.

## Detailed Design

The proposed change is to standardize comment extraction and handle the varying implementations of ASTs across different language plugins.

Specifically:

1. **Standardize `comments` property on `SourceCode`:** - A `comments` property is to be added to the JavaScript `SourceCode` class. The `CSSSourceCode` and `JSONSourceCode` classes already expose this property. `MarkdownSourceCode` does not expose it, as Markdown does not have a direct equivalent to comments.

2. **Update `no-irregular-whitespace` Comment Extraction:** - The rule is to be updated to access the `sourceCode.comments` property, with a fallback to an empty array for languages like Markdown that lack comments.

```js
const commentNodes = sourceCode.comments || [];
```

Additionally, to make `skipComments` logic work reliably across all languages, the check will be updated to use `sourceCode.getText(node)` instead of `node.value`, as comment node shapes vary (e.g., JSON comment nodes from `@eslint/json` do not expose a `.value` property).

```js
function removeInvalidNodeErrorsInComment(node) {
	if (ALL_IRREGULARS.test(sourceCode.getText(node))) {
		removeWhitespaceError(node);
	}
}
```

3. **Generic Syntax Exclusion via `skipNodes` Option:** - A new configuration option, `skipNodes`, will be added. This option will accept an array of ESQuery selectors. The rule will dynamically register traversal listeners for these selectors and strip any irregular whitespace errors found within their boundaries. For example, a user linting CSS or JSON could set `skipNodes: ["String"]`.

```js
// In schema:
// skipNodes: { type: "array", items: { type: "string" } }

const [{ skipNodes }] = context.options;

function removeInvalidNodeErrorsInSelector(node) {
	const rawText = sourceCode.getText(node);
	if (ALL_IRREGULARS.test(rawText)) {
		removeWhitespaceError(node);
	}
}

skipNodes.forEach(selector => {
	nodes[selector] = removeInvalidNodeErrorsInSelector;
});
```

The existing JS-specific options (`skipStrings`, `skipRegExps`, etc.) will be retained and continue to function alongside `skipNodes` for backwards compatibility.

4. **Root Node Listener:** - The rule is to be updated to attach the main traversal to the root node type of the current AST, rather than the hardcoded `Program` node. This will accommodate different language plugins that use different root nodes (e.g., `StyleSheet` for CSS, `Document` for JSON, and `root` for Markdown).

```js
const rootNodeType = sourceCode.ast.type;

nodes[rootNodeType] = function (node) {
	checkForIrregularWhitespace(node);
	checkForIrregularLineTerminators(node);
};

nodes[`${rootNodeType}:exit`] = function () {
	// Handle comment stripping and reporting errors
};
```

## Documentation

- [Custom Rules documentation](https://eslint.org/docs/latest/extend/custom-rules) should be updated to document the new `comments` property on the `SourceCode` object.
- [`no-irregular-whitespace` rule documentation](https://eslint.org/docs/latest/rules/no-irregular-whitespace) should be updated to document the new `skipNodes` option, explaining how to use it with ESQuery selectors, and to note that the rule can now safely be used on non-JavaScript files.

## Drawbacks

The existing JS-specific options (`skipStrings`, `skipTemplates`, `skipRegExps`, `skipJSXText`) will have no effect when the rule is used on non-JS files, since the corresponding AST node types (`Literal`, `TemplateElement`, `JSXText`) do not exist in CSS, JSON, or Markdown ASTs. The listeners are registered but never triggered. The new `skipNodes` option is the intended mechanism for non-JS exclusions.

## Backwards Compatibility Analysis

This change is fully backwards compatible. The JS `SourceCode` object will continue to expose the existing `getAllComments()` method alongside the new `comments` property, ensuring no disruption to ecosystem plugins that rely on the older API. The existing boolean options for skipping specific JS nodes (`skipStrings`, `skipTemplates`, etc.) will also continue to behave exactly as they do today.

## Alternatives

Separate rules, such as `css/no-irregular-whitespace` and `json/no-irregular-whitespace`, could be created. However, checking text for irregular whitespace is a text-based operation that applies to almost any language, making a unified language-agnostic core rule more maintainable.

## Open Questions

With the addition of the `comments` property to the JavaScript `SourceCode` class, its existing `getAllComments()` method becomes redundant. Should a formal deprecation (via JSDoc `@deprecated` tag and documentation updates) be included in the scope of this RFC, or should it be deferred to avoid immediate ecosystem churn?

## Help Needed

None.

## Frequently Asked Questions

None.

## Related Discussions

- eslint/eslint#19805
