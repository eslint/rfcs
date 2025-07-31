- Repo: eslint/css
- Start Date: 2025-06-24
- RFC PR:
- Authors: Nicholas C. Zakas

# CSS Custom Property Tracking in SourceCode

## Summary

This RFC proposes to add capabilities to `CSSSourceCode` to track CSS custom properties (also known as CSS variables). This includes identifying where custom properties are defined and where they are used, as well as providing a way to retrieve the value of a custom property at a specific location in the code.

## Motivation

CSS custom properties are a foundational part of modern CSS development. They allow for more modular and maintainable stylesheets. For a CSS linter to provide accurate and helpful rules, it needs to be able to understand how custom properties are being used. Currently, any rule that needs to validate property values must implement its own logic for tracking custom properties, leading to duplicated effort and potential inconsistencies.

By building this functionality directly into `CSSSourceCode`, we can provide a consistent and reliable way for all rules to access information about custom properties. Uses of this information include:

*   `no-invalid-properties` rule
*   `font-family-fallbacks` rule
*   Detecting references to undefined custom properties.
*   Future: Detecting unused custom properties.

This change is based on the discussion in [eslint/css#160](https://github.com/eslint/css/issues/160).

**Note:** The logic described in this RFC is already implemented in the [`no-invalid-properties`](https://github.com/eslint/css/blob/main/docs/rules/no-invalid-properties.md) rule. This proposal wants to standardize the logic and make it available to all rules.

## Detailed Design

The proposed changes are implemented in the [`CSSSourceCode`](https://github.com/eslint/css/blob/main/src/languages/css-source-code.js) class.

### `customProperties` Map

```ts
interface CustomPropertyUses {
    declarations: Array<DeclarationPlain>;
    definitions: Array<AtrulePlain>;
    references: Array<FunctionNode>;
}

interface CSSSourceCode {
    #customProperties: Map<string, CustomPropertyUses>
}
```

A new private property, `#customProperties`, will be added to `CSSSourceCode`. This will be a `Map` where the keys are the custom property names (e.g., `--my-color`) and the values are `CustomPropertyUses` objects. The `CustomPropertyUses` class will have three properties:

*   `declarations`: An array of `DeclarationPlain` nodes where the custom property value is declared.
*   `definitions`: Array of `AtrulePlain` nodes where the custom property is defined using an [`@property`](https://developer.mozilla.org/en-US/docs/Web/CSS/@property) rule.
*   `references`: An array of `FunctionNode` nodes (specifically `var()` functions) where the custom property is used.

### `getDeclarationVariables()` Method

```ts
interface CSSSourceCode {
    getDeclarationVariables(declaration: DeclarationPlain): Array<Function>;
}
```

A new public method, `getDeclarationVariables(declaration)`, will be added to `CSSSourceCode`. This method will take a `Declaration` node as an argument and return an array of `Function` nodes representing the `var()` functions used in that declaration's value.

### `getClosestVariableValue()` Method

```ts
interface CSSSourceCode {
    getClosestVariableValue(func: FunctionNode): Raw;
}
```

A new public method, `getClosestVariableValue(node)`, will be added to `CSSSourceCode`. This method will take a `var()` `Function` node as an argument and return the computed value of the custom property (which is currently always a `Raw` node). It will do this by searching for the last declaration of the custom property that appears before the given `Function` node in the source code. This also leaves open the possibility that we could change how this value is calculated to be more accurate in the future.

### `getVariableValues()` Method

```ts
interface CSSSourceCode {
    getVariableValues(func: FunctionNode): Array<Raw>;
}
```

A new public method, `getVariableValues(func)`, will be added to `CSSSourceCode`. This method will take a `var()` `Function` node as an argument and return an array of `Raw` nodes representing the declared values of the custom property. The fallback value, if specified in the `FunctionNode`, is returned as the last element of the array. 

### Initialization

The `traverse()` method in `CSSSourceCode` will be updated to populate the `#customProperties` map and the internal data structure (`WeakMap<DeclarationPlain, Array<FunctionNode>>`) used by `getDeclarationVariables()`. During traversal, it will identify `Declaration` nodes that define custom properties and `Function` nodes that are `var()` calls.

## Documentation

Because we don't provide documentation for `CSSSourceCode`, we will rely primarily on TypeScript types to inform rule developers as to these new class members.

## Drawbacks

The primary drawback of this approach is that it only tracks custom properties within a single file. It cannot resolve custom properties that are defined in other files or via mechanisms like inline styles on HTML elements. For the initial implementation, this is considered an acceptable tradeoff.

## Backwards Compatibility Analysis

This is a new feature and does not change any existing APIs. It is fully backwards compatible.

## Alternatives

The primary alternative is to have each rule implement its own custom property tracking logic. This is the current state of affairs and is what this RFC aims to improve upon. A centralized approach is more efficient and less error-prone.

## Open Questions

n/a

## Help Needed

No help is needed to implement this RFC.

## Frequently Asked Questions

**Why does `getVariableValue()` return a `Raw` instead of a string?**

The `getVariableValue()` method returns a `Raw` node instead of a string to preserve the original source information and maintain consistency with the AST structure. A `Raw` node contains not only the text value but also the location, which is valuable for rules that need to report issues or apply fixes at specific locations in the source code. Additionally, returning the actual AST node allows for future extensibility. If we later need to return more complex computed values or support different node types, the API won't need to change.

## Related Discussions

- [Change Request: Track variables on SourceCode #160](https.github.com/eslint/css/issues/160)
