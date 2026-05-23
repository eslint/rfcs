- Repo: eslint/eslint
- Start Date: 2026-05-23
- RFC PR: (fill in after PR is created)
- Authors: [Kuldeep2822k](https://github.com/Kuldeep2822k)

# Add `getPossibleTypesForSelectorClass` to the Language Interface

## Summary

Add an optional `getPossibleTypesForSelectorClass(className)` method to the `Language` interface that allows languages to declare which AST node types could match a given pseudo-class selector (e.g., `:function`). This removes hardcoded JavaScript-specific logic from the core selector analysis in `lib/linter/esquery.js` and enables any language plugin to provide the same optimization.

## Motivation

ESLint's core selector engine ([`lib/linter/esquery.js`](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js)) performs static analysis on parsed selectors to determine which AST node types could possibly trigger them. This is a performance optimization: if a selector like `:function` can only match `FunctionDeclaration`, `FunctionExpression`, and `ArrowFunctionExpression` nodes, the traverser can skip invoking the selector's matching logic on all other node types.

However, there is currently a hardcoded JavaScript-specific mapping for the `:function` pseudo-class:

```js
// lib/linter/esquery.js, lines 208-217
case "class":
    // TODO: abstract into JSLanguage somehow
    if (selector.name === "function") {
        return [
            "FunctionDeclaration",
            "FunctionExpression",
            "ArrowFunctionExpression",
        ];
    }
    return null;
```

This presents several problems:

1. **Language-agnostic roadmap conflict.** The Language interface was designed ([RFC #99](https://github.com/eslint/rfcs/pull/99)) to make ESLint language-agnostic. Having JS-specific logic in the core selector engine contradicts this goal.

2. **Missing optimization for other languages.** The Language interface already provides `matchesSelectorClass()` for _runtime_ matching of nodes against pseudo-classes. However, there is no companion method for _static analysis_ — determining upfront which node types a pseudo-class could possibly match. Languages like CSS, Markdown, JSON, HTML, and YAML that implement custom pseudo-classes cannot benefit from this traversal optimization.

3. **Maintenance concern.** The existing `TODO` comment explicitly acknowledges this code doesn't belong in the core. As more languages are added, having JS-specific logic in the core linter becomes increasingly confusing for contributors.

The Language interface already has a precedent for this pattern: `matchesSelectorClass()` handles runtime matching, and the proposed `getPossibleTypesForSelectorClass()` would handle static analysis. Together, they provide a complete, language-agnostic pseudo-class system.

## Detailed Design

This proposal consists of the following changes:

1. Add an optional `getPossibleTypesForSelectorClass()` method to the `Language` interface (in `@eslint/core`).
2. Move the JS-specific `:function` mapping into `lib/languages/js/index.js`.
3. Update `lib/linter/esquery.js` to call the language method instead of using hardcoded logic.
4. Make the selector cache language-aware.

### The `getPossibleTypesForSelectorClass()` Method

A new optional method is added to the `Language` interface:

```ts
interface Language {
    // ... existing properties ...

    /**
     * Returns the AST node types that could possibly match a given
     * pseudo-class selector. Used for static analysis of selectors
     * to optimize traversal by narrowing which node types need to
     * be checked.
     *
     * @param className The name of the pseudo-class (e.g., "function").
     * @returns An array of node type strings that could match, or
     *          `null` if all node types could potentially match.
     */
    getPossibleTypesForSelectorClass?(className: string): string[] | null;
}
```

**Return value semantics:**

- **`string[]`** — Only nodes of these types could match the pseudo-class. The traverser will only check these node types, skipping all others.
- **`null`** — Any node type could match. The traverser must check every node (this is the safe fallback).

**If a language does not implement this method**, the core falls back to returning `null` for all class selectors, which is functionally equivalent to the current behavior for all pseudo-classes _except_ `:function` in JS. This means no existing language is negatively affected.

### JavaScript Language Implementation

The JS language object in [`lib/languages/js/index.js`](https://github.com/eslint/eslint/blob/main/lib/languages/js/index.js) would implement this method alongside the existing `matchesSelectorClass()`:

```js
getPossibleTypesForSelectorClass(className) {
    if (className === "function") {
        return [
            "FunctionDeclaration",
            "FunctionExpression",
            "ArrowFunctionExpression",
        ];
    }
    return null;
}
```

This is a direct extraction of the logic currently hardcoded in `esquery.js` (line 210: `if (selector.name === "function")`). Note that the existing hardcoded check uses a case-sensitive comparison (`===`), and so does this implementation to maintain the same behavior. The `className` parameter receives the raw `selector.name` value from the esquery parsed AST, which preserves the casing as written in the selector string (e.g., `:function` → `"function"`).

The existing `matchesSelectorClass()` in the JS language already handles runtime matching for classes like `statement`, `declaration`, `pattern`, `expression`, and `function` (see [lines 163-229](https://github.com/eslint/eslint/blob/main/lib/languages/js/index.js#L163-L229)). This new method provides the static type analysis companion.

Note: The other pseudo-classes (`statement`, `declaration`, `pattern`, `expression`) return `null` from the static analysis (meaning "any type could match") because they match based on suffix patterns (e.g., any type ending in `"Statement"`) or complex conditions involving ancestry. We cannot statically enumerate all possible node types for these classes. Only `:function`, which matches a fixed set of three specific types, benefits from this optimization.

### Core `esquery.js` Changes

The `analyzeParsedSelector()` function (currently at [line 148](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js#L148)) is updated to accept a `language` parameter, which is passed into the inner `analyzeSelector()` closure:

```diff
-function analyzeParsedSelector(parsedSelector) {
+function analyzeParsedSelector(parsedSelector, language) {
     // ...
     function analyzeSelector(selector) {
         // ...
         case "class":
-            // TODO: abstract into JSLanguage somehow
-            if (selector.name === "function") {
-                return [
-                    "FunctionDeclaration",
-                    "FunctionExpression",
-                    "ArrowFunctionExpression",
-                ];
-            }
-            return null;
+            return language?.getPossibleTypesForSelectorClass?.(selector.name) ?? null;
     }
 }
```

The optional chaining (`?.`) ensures that if `language` is `undefined` or doesn't implement the method, the result is `null` — meaning "any node type could match," which is the safe fallback.

The exported `parse()` function (currently at [line 288](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js#L288)) is updated to accept an optional `language` parameter and pass it through:

```diff
-function parse(source) {
-    if (selectorCache.has(source)) {
-        return selectorCache.get(source);
+function parse(source, language) {
+    const cache = getLanguageCache(language);
+
+    if (cache.has(source)) {
+        return cache.get(source);
     }

     const cleanSource = source.replace(/:exit$/u, "");
     const parsedSelector =
         trySimpleParseSelector(cleanSource) ?? tryParseSelector(cleanSource);
     const { nodeTypes, attributeCount, identifierCount } =
-        analyzeParsedSelector(parsedSelector);
+        analyzeParsedSelector(parsedSelector, language);

     const result = new ESQueryParsedSelector(
         source,
         source.endsWith(":exit"),
         parsedSelector,
         nodeTypes,
         attributeCount,
         identifierCount,
     );

-    selectorCache.set(source, result);
+    cache.set(source, result);
     return result;
 }
```

### Language-Aware Selector Cache

Currently, `esquery.js` uses a single global `Map` ([line 114](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js#L114): `const selectorCache = new Map()`) to cache parsed selectors. Since `getPossibleTypesForSelectorClass()` can return different results for different languages, the same selector string (e.g., `:function`) may produce different `nodeTypes` arrays depending on which language is active.

The cache is replaced with a `WeakMap` keyed by language object, so each language gets its own `Map<string, ESQueryParsedSelector>`:

```js
// Replaces: const selectorCache = new Map();
const selectorCacheByLanguage = new WeakMap();
const noLanguageCache = new Map();

function getLanguageCache(language) {
    if (!language) {
        return noLanguageCache;
    }

    let cache = selectorCacheByLanguage.get(language);

    if (!cache) {
        cache = new Map();
        selectorCacheByLanguage.set(language, cache);
    }

    return cache;
}
```

Using `WeakMap` ensures that language objects can be garbage collected when they're no longer in use, preventing memory leaks. The `noLanguageCache` provides backwards compatibility for any code that calls `parse()` without a language argument.

Note: In practice, ESLint currently only uses one language per file (see [source-code-traverser.js line 269-275](https://github.com/eslint/eslint/blob/main/lib/linter/source-code-traverser.js#L269-L275)), but a multi-language lint run will process different files with different languages sequentially, so the per-language cache prevents stale optimization data from being reused across languages.

### Source Code Traverser Changes

The [`SourceCodeTraverser`](https://github.com/eslint/eslint/blob/main/lib/linter/source-code-traverser.js) already stores the language as a private field `#language` (set in the constructor at [line 249](https://github.com/eslint/eslint/blob/main/lib/linter/source-code-traverser.js#L249)). The only changes needed are:

1. Pass the language through the `esqueryOptions` object when constructing `ESQueryHelper`:

```diff
 // In SourceCodeTraverser.traverseSync() (line 269)
 traverseSync(sourceCode, visitor, { steps } = {}) {
     const esquery = new ESQueryHelper(visitor, {
         visitorKeys: sourceCode.visitorKeys ?? this.#language.visitorKeys,
         fallback: vk.getKeys,
         matchClass: this.#language.matchesSelectorClass ?? (() => false),
         nodeTypeKey: this.#language.nodeTypeKey,
+        language: this.#language,
     });
```

2. In the `ESQueryHelper` constructor, store the language and pass it to `parse()`:

```diff
 // In ESQueryHelper constructor (line 52)
 constructor(visitor, esqueryOptions) {
+    this.language = esqueryOptions.language;
     // ...
     visitor.forEachName(rawSelector => {
-        const selector = parse(rawSelector);
+        const selector = parse(rawSelector, this.language);
         // ... rest unchanged
     });
 }
```

## Drawbacks

1. **Increased Language interface surface.** Adding a new optional method increases the size of the Language contract. However, it is optional with a safe fallback, so existing language implementations are unaffected.

2. **Potential for inconsistency.** A language could implement `matchesSelectorClass()` and `getPossibleTypesForSelectorClass()` inconsistently (e.g., `matchesSelectorClass` handles `:function` but `getPossibleTypesForSelectorClass` doesn't list the right types). However, the consequence of inconsistency is only reduced optimization (more nodes checked than necessary), not incorrect behavior, because `matchesSelectorClass` is still the source of truth for actual matching.

3. **Cache complexity.** The cache changes from a simple global `Map` to a `WeakMap` of `Map`s. This is slightly more complex, but the added complexity is minimal and localized to `esquery.js`.

4. **Small scope.** As noted by the ESLint maintainer, this is "a tiny corner of the codebase that doesn't get hit very often." The practical benefit of this change is primarily architectural cleanliness rather than a significant feature or performance win.

## Backwards Compatibility

This change is **fully backwards compatible**:

1. **`getPossibleTypesForSelectorClass()` is optional.** If a language does not implement it, the core falls back to `null` (match any type). This is the current behavior for all pseudo-classes except `:function` in JS.

2. **JS behavior is preserved.** The JavaScript language object will implement `getPossibleTypesForSelectorClass()` with the exact same mapping currently hardcoded in `esquery.js`. End users will see no behavioral difference.

3. **`parse()` API is backwards compatible.** The `language` parameter is optional. Calling `parse(source)` without a language continues to work, using the `noLanguageCache` with `null` node types for class selectors.

4. **No breaking changes for language plugins.** Existing language plugins (CSS, Markdown, JSON, HTML, YAML) do not need to implement this method. They will simply not benefit from the optimization until they choose to.

5. **Type definitions.** The method is added as an optional property (`getPossibleTypesForSelectorClass?`) in `@eslint/core`'s `Language` interface, so existing TypeScript users are unaffected.

## Alternatives

### 1. Do Nothing (Status Quo)

Leave the hardcoded JS logic in `esquery.js` with the `TODO` comment. This is the simplest approach, and the maintainer has acknowledged the current state is acceptable. However, it leaves a known architectural debt and prevents other languages from benefiting from the same optimization.

### 2. Move Logic Without a Language Interface Method

Move the hardcoded logic into a helper function that checks the language identity (e.g., `if (language === jsLanguage)`) rather than adding a generic interface method. This removes the hardcoding from `esquery.js` but doesn't provide a generic solution for other languages and still couples the core to the JS language.

### 3. Use `matchesSelectorClass()` for Static Analysis

Instead of adding a new method, try to infer possible types by calling `matchesSelectorClass()` against a set of known node types. This is impractical because:
- The set of possible node types is unbounded (any language can define any node types).
- It would require instantiating dummy nodes for every possible type.
- It conflates runtime matching with static analysis.

### 4. Extend `matchesSelectorClass()` to Return Type Information

Instead of a separate method, modify `matchesSelectorClass()` to optionally return type information. This was rejected because it changes the semantics of an existing method and makes the return type complex (boolean vs. type array).

## Open Questions

1. **Method naming.** The proposed name `getPossibleTypesForSelectorClass` is descriptive but verbose. Alternatives considered:
   - `getNodeTypesForClass(className)` — shorter, but less clear about "possible" semantics
   - `selectorClassNodeTypes(className)` — property-style naming
   
   The longer name was chosen for clarity, matching the style of `matchesSelectorClass`. The team may prefer a different name.

2. **Should language authors be encouraged to implement this for all their pseudo-classes?** For pseudo-classes that match based on type-name patterns (like `:statement` matching `*Statement`), returning `null` is correct since we can't enumerate all types. Should the documentation explicitly guide authors on when to return `null` vs. an explicit list?

3. **Cache eviction strategy.** The `WeakMap` approach means caches are evicted when a language object is garbage collected. In practice, language objects are long-lived (often singletons). Should we consider adding a size limit to per-language caches, or is unbounded growth acceptable given the finite number of selectors in a typical lint run?

## Help Needed

I am willing to submit a pull request with the reference implementation for this RFC once it is accepted. Performance benchmarking (using `npm run test:performance`) will be done to confirm there is no regression.

## Frequently Asked Questions

**Q: Why not wait until there's a concrete non-JS language that needs this?**

A: While the immediate benefit is architectural, making this change now is low-risk (fully backwards compatible, optional method) and avoids accumulating more JS-specific debt in the core. It also sends a clear signal to language plugin authors that the optimization path exists.

**Q: Does this change affect how `matchesSelectorClass()` works?**

A: No. `matchesSelectorClass()` continues to handle runtime matching exactly as before. `getPossibleTypesForSelectorClass()` is a separate, complementary method used only during selector parsing for static analysis.

## Related Discussions

- [Discussion #20856](https://github.com/eslint/eslint/discussions/20856) — Original discussion: "Should we move JS-specific ESQuery analysis logic into the Language interface?"
- [RFC #99](https://github.com/eslint/rfcs/pull/99) — ESLint Language Plugins (introduced the Language interface)
- [esquery External class resolve PR](https://github.com/estools/esquery/pull/140) — The mechanism that enabled `matchesSelectorClass()`
