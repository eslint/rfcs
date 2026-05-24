- Repo: eslint/eslint
- Start Date: 2026-05-23
- RFC PR: https://github.com/eslint/rfcs/pull/148
- Authors: Kuldeep2822k

# Add `getSelectorClassNodeTypes` to the Language Interface

## Summary

Add an optional `getSelectorClassNodeTypes(className)` method to the Language interface that allows languages to declare which AST node types could match a given pseudo-class selector (e.g., `:function`). This removes hardcoded JavaScript-specific logic from the core selector analysis in `lib/linter/esquery.js` and enables any language plugin to participate in selector static analysis.

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

2. **Incomplete selector-class extension API.** The current Language interface allows plugins to define runtime pseudo-class behavior through `matchesSelectorClass()`, but provides no mechanism for participating in selector static analysis. This means custom language plugins cannot integrate fully with ESLint's traversal optimization pipeline even when their pseudo-classes map to a finite set of node types. Without a static analysis companion, plugin authors would need to request core changes for every optimizable pseudo-class, and selector semantics would not be fully owned by the language implementation — contradicting the language-agnostic architecture.

3. **Maintenance concern.** The existing `TODO` comment explicitly acknowledges this code doesn't belong in the core. As more languages are added, having JS-specific logic in the core linter becomes increasingly confusing for contributors.

Selector analysis depends on selector semantics, and selector semantics are already delegated to language implementations through `matchesSelectorClass()`. Static analysis therefore belongs to the same abstraction boundary. The proposed `getSelectorClassNodeTypes()` completes this symmetry, providing a full, language-agnostic pseudo-class system where both runtime matching and static analysis are owned by the language implementation.

## Detailed Design

This proposal consists of the following changes:

1. Add an optional `getSelectorClassNodeTypes()` method to the `Language` interface (in `@eslint/core`).
2. Move the JS-specific `:function` mapping into `lib/languages/js/index.js`.
3. Update `lib/linter/esquery.js` to call the language method instead of using hardcoded logic.
4. Make the selector cache language-aware.

### The `getSelectorClassNodeTypes()` Method

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
    getSelectorClassNodeTypes?(className: string): string[] | null;
}
```

**Return value semantics:**

- **`string[]`** — Only nodes of these types could match the pseudo-class. The traverser will only invoke selector matching for these node types, skipping all others.
- **`null`** — Any node type could match. The traverser must check every node (this is the safe fallback).

**If a language does not implement this method**, the core falls back to returning `null` for all class selectors, which is functionally equivalent to the current behavior for all pseudo-classes _except_ `:function` in JS. Existing language implementations remain unaffected.

**Semantic invariant:** For correctness, the set of node types returned by `getSelectorClassNodeTypes()` MUST be a superset of the types that actually match through `matchesSelectorClass()`. Returning extra types is allowed and only reduces optimization quality — the traverser checks more nodes than necessary, but never misses a match.

**Why `null` instead of `undefined`:** `null` is used intentionally to distinguish "all node types may match" (an explicit statement about the pseudo-class) from "method not implemented" (which is `undefined` via optional chaining). This prevents collapsing semantics between an explicitly non-optimizable pseudo-class and a missing method.

### JavaScript Language Implementation

The JS language object in [`lib/languages/js/index.js`](https://github.com/eslint/eslint/blob/main/lib/languages/js/index.js) would implement this method alongside the existing `matchesSelectorClass()`:

```js
getSelectorClassNodeTypes(className) {
    if (className.toLowerCase() === "function") {
        return [
            "FunctionDeclaration",
            "FunctionExpression",
            "ArrowFunctionExpression",
        ];
    }
    return null;
}
```

This is a direct extraction of the logic currently hardcoded in `esquery.js` (line 210: `if (selector.name === "function")`). Note that the existing hardcoded check in `esquery.js` uses a case-sensitive comparison (`===`), but the runtime `matchesSelectorClass()` (line 191) uses `className.toLowerCase()` for case-insensitive matching. The proposed implementation uses `.toLowerCase()` to remain consistent with `matchesSelectorClass()`, since esquery's parser preserves the raw casing of `selector.name` (e.g., `:Function` → `"Function"`). The case-sensitive hardcoded check in `esquery.js` only works because selectors are conventionally lowercase; `getSelectorClassNodeTypes()` should handle edge cases consistently with the runtime method.

The existing `matchesSelectorClass()` in the JS language already handles runtime matching for classes like `statement`, `declaration`, `pattern`, `expression`, and `function` (see [lines 163-229](https://github.com/eslint/eslint/blob/main/lib/languages/js/index.js#L163-L229)). `getSelectorClassNodeTypes()` provides the static type analysis companion.

Note: The other pseudo-classes (`statement`, `declaration`, `pattern`, `expression`) return `null` from the static analysis (meaning "any type could match") because they match based on suffix patterns (e.g., any type ending in `"Statement"`) or complex conditions involving ancestry. These pseudo-classes are defined by structural or naming conventions rather than a fixed closed set of node types, making enumeration language-specific and potentially brittle. Only `:function`, which maps to a fixed closed set of node types, benefits from this optimization.

### Example: CSS Language Plugin

To illustrate the extensibility value, consider a CSS language plugin that exposes a `:rule` pseudo-class matching only rule-type nodes:

```js
getSelectorClassNodeTypes(className) {
    if (className === "rule") {
        return ["StyleRule", "AtRule"];
    }
    return null;
}
```

Without this interface method, the CSS plugin would need to request a core change to add its own hardcoded mapping — exactly the coupling this proposal eliminates.

### Core `esquery.js` Changes

The `analyzeParsedSelector()` function (currently at [line 148](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js#L148)) is updated to accept a `getSelectorClassNodeTypes` parameter, which is passed into the inner `analyzeSelector()` closure:

```diff
-function analyzeParsedSelector(parsedSelector) {
+function analyzeParsedSelector(parsedSelector, getSelectorClassNodeTypes) {
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
+            return getSelectorClassNodeTypes?.(selector.name) ?? null;
     }
 }
```

The optional chaining (`?.`) ensures that if `getSelectorClassNodeTypes` is `undefined` (i.e. the method wasn't provided), the result is `null` — meaning "any node type could match," which is the safe fallback.

The exported `parse()` function (currently at [line 288](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js#L288)) is updated to accept an optional `getSelectorClassNodeTypes` parameter and pass it through:

```diff
-function parse(source) {
-    if (selectorCache.has(source)) {
-        return selectorCache.get(source);
+function parse(source, getSelectorClassNodeTypes) {
+    const cache = getMethodCache(getSelectorClassNodeTypes);
+
+    if (cache.has(source)) {
+        return cache.get(source);
     }

     const cleanSource = source.replace(/:exit$/u, "");
     const parsedSelector =
         trySimpleParseSelector(cleanSource) ?? tryParseSelector(cleanSource);
     const { nodeTypes, attributeCount, identifierCount } =
-        analyzeParsedSelector(parsedSelector);
+        analyzeParsedSelector(parsedSelector, getSelectorClassNodeTypes);

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

### Method-Keyed Selector Cache

Currently, `esquery.js` uses a single global `Map` ([line 114](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js#L114): `const selectorCache = new Map()`) to cache parsed selectors. Since `getSelectorClassNodeTypes()` can return different results for different languages, the same selector string (e.g., `:function`) may produce different `nodeTypes` arrays depending on which language is active.

The cache is replaced with a `WeakMap` keyed by the language's `getSelectorClassNodeTypes` method reference, so each language gets its own `Map<string, ESQueryParsedSelector>`:

```js
// Replaces: const selectorCache = new Map();
const selectorCacheByMethod = new WeakMap();
const noMethodCache = new Map();

function getMethodCache(getSelectorClassNodeTypes) {
    if (!getSelectorClassNodeTypes) {
        return noMethodCache;
    }

    let cache = selectorCacheByMethod.get(getSelectorClassNodeTypes);

    if (!cache) {
        cache = new Map();
        selectorCacheByMethod.set(getSelectorClassNodeTypes, cache);
    }

    return cache;
}
```

ESLint languages are exported as singleton object literals (e.g., `lib/languages/js/index.js` exports its language as a `module.exports` object literal), so the method reference is stable for the lifetime of the process. Keying on the method therefore gives each language its own cache bucket without requiring a language identifier on the `Language` interface.

Using `WeakMap` ensures that if a language (and hence its method) is no longer reachable, it can be garbage collected, preventing memory leaks. The `noMethodCache` provides backwards compatibility for any code that calls `parse(source)` without providing a method.

Note: In practice, ESLint currently only uses one language per file (see [source-code-traverser.js line 269-275](https://github.com/eslint/eslint/blob/main/lib/linter/source-code-traverser.js#L269-L275)), but a multi-language lint run will process different files with different languages sequentially, so the method-keyed cache prevents stale optimization data from being reused across languages.

### Source Code Traverser Changes

The [`SourceCodeTraverser`](https://github.com/eslint/eslint/blob/main/lib/linter/source-code-traverser.js) already stores the language as a private field `#language` (set in the constructor at [line 249](https://github.com/eslint/eslint/blob/main/lib/linter/source-code-traverser.js#L249)). The only changes needed are:

1. Pass the method directly through the `esqueryOptions` object when constructing `ESQueryHelper`, consistent with how `matchesSelectorClass`, `visitorKeys`, and `nodeTypeKey` are already passed individually:

```diff
 // In SourceCodeTraverser.traverseSync() (line 269)
 traverseSync(sourceCode, visitor, { steps } = {}) {
     const esquery = new ESQueryHelper(visitor, {
         visitorKeys: sourceCode.visitorKeys ?? this.#language.visitorKeys,
         fallback: vk.getKeys,
         matchClass: this.#language.matchesSelectorClass ?? (() => false),
         nodeTypeKey: this.#language.nodeTypeKey,
+        getSelectorClassNodeTypes: this.#language.getSelectorClassNodeTypes,
     });
```

2. In the `ESQueryHelper` constructor, store the method and pass it to `parse()`:

```diff
 // In ESQueryHelper constructor (line 52)
 constructor(visitor, esqueryOptions) {
+    this.getSelectorClassNodeTypes = esqueryOptions.getSelectorClassNodeTypes;
     // ...
     visitor.forEachName(rawSelector => {
-        const selector = parse(rawSelector);
+        const selector = parse(rawSelector, this.getSelectorClassNodeTypes);
         // ... rest unchanged
     });
 }
```

## Drawbacks

1. **Increased Language interface surface.** Adding a new optional method increases the size of the Language contract. However, it is optional with a safe fallback, so existing language implementations are unaffected.

2. **Potential for inconsistency.** A language could implement `matchesSelectorClass()` and `getSelectorClassNodeTypes()` inconsistently (e.g., `matchesSelectorClass` handles `:function` but `getSelectorClassNodeTypes` doesn't list the right types). The consequence of such inconsistency is only reduced optimization (more nodes checked than necessary), not incorrect behavior. This is a major design strength: static analysis is advisory only, and runtime matching via `matchesSelectorClass()` remains authoritative. Under this design, plugin authors cannot cause false negatives through an imprecise `getSelectorClassNodeTypes()` implementation. At worst, the traverser checks more nodes than strictly necessary. The only way incorrect behavior could arise is if ESLint core itself misused the static analysis results to skip runtime matching, which this design explicitly avoids.

3. **Cache complexity.** The cache changes from a simple global `Map` to a `WeakMap` of `Map`s. This is slightly more complex, but the added complexity is minimal and localized to `esquery.js`.

4. **Small scope.** The current optimization affects a relatively small part of the selector pipeline, so the primary benefit of this proposal is architectural consistency and extensibility rather than a large performance gain.

## Backwards Compatibility

This change is **fully backwards compatible**:

1. **`getSelectorClassNodeTypes()` is optional.** If a language does not implement it, the core falls back to `null` (match any type). This is the current behavior for all pseudo-classes except `:function` in JS.

2. **JS behavior is preserved.** The JavaScript language object will implement `getSelectorClassNodeTypes()` with the exact same mapping currently hardcoded in `esquery.js`. End users will see no behavioral difference.

3. **`parse()` API is backwards compatible.** The `getSelectorClassNodeTypes` parameter is optional. Calling `parse(source)` without it continues to work, using the `noMethodCache` with `null` node types for class selectors.

4. **No breaking changes for language plugins.** Existing language plugins (CSS, Markdown, JSON, HTML, YAML) do not need to implement this method. They will simply not benefit from the optimization until they choose to.

5. **Type definitions.** The method is added as an optional property (`getSelectorClassNodeTypes?`) in `@eslint/core`'s `Language` interface, so existing TypeScript users are unaffected.

## Alternatives

### 1. Move Logic Without a Language Interface Method

Move the hardcoded logic into a helper function that checks the language identity (e.g., `if (language === jsLanguage)`) rather than adding a generic interface method. This removes the hardcoding from `esquery.js` but doesn't provide a generic solution for other languages and still couples the core to the JS language.

### 2. Use `matchesSelectorClass()` for Static Analysis

Instead of adding a new method, try to infer possible types by calling `matchesSelectorClass()` against a set of known node types. This is impractical because:
- ESLint core has no canonical registry of all node types for arbitrary languages, so exhaustive inference would require language-specific enumeration anyway.
- It would require instantiating dummy nodes for every possible type.
- It conflates runtime matching with static analysis.

### 3. Extend `matchesSelectorClass()` to Return Type Information

Instead of a separate method, modify `matchesSelectorClass()` to optionally return type information. This was rejected because it changes the semantics of an existing method and makes the return type complex (boolean vs. type array).


## Open Questions

1. **Method naming.** The proposed name `getSelectorClassNodeTypes` was chosen for conciseness and alignment with ESLint core conventions. It is directly tied to selector classes, avoids the unnecessary "possible" qualifier (static analysis inherently implies possibility, not certainty), and keeps the return semantics clear from the documented contract. The team may prefer a different name.

2. **Should language authors be encouraged to implement this for all their pseudo-classes?** For pseudo-classes defined by structural or naming conventions (like `:statement` matching `*Statement`), returning `null` is appropriate since the matching set is open-ended. Should the documentation explicitly guide authors on when to return `null` vs. an explicit list?

3. **Cache eviction strategy.** The `WeakMap` approach means caches are evicted when a language's `getSelectorClassNodeTypes` method becomes unreachable. In practice, language objects (and their methods) are long-lived (often singletons). Should we consider adding a size limit to per-method caches, or is unbounded growth acceptable given the finite number of selectors in a typical lint run?

## Help Needed

I am willing to submit a pull request with the reference implementation for this RFC once it is accepted. The change should be performance-neutral outside selector parsing and traversal narrowing.

## Frequently Asked Questions

**Q: Why not wait until there's a concrete non-JS language that needs this?**

A: While the immediate benefit is architectural, making this change now is low-risk (fully backwards compatible, optional method) and avoids accumulating more JS-specific debt in the core. It also sends a clear signal to language plugin authors that the optimization path exists.

**Q: Does this change affect how `matchesSelectorClass()` works?**

A: No. `matchesSelectorClass()` continues to handle runtime matching exactly as before. `getSelectorClassNodeTypes()` is a separate, complementary method used only during selector parsing for static analysis.

## Related Discussions

- [Discussion #20856](https://github.com/eslint/eslint/discussions/20856) — Original discussion: "Should we move JS-specific ESQuery analysis logic into the Language interface?"
- [RFC #99](https://github.com/eslint/rfcs/pull/99) — ESLint Language Plugins (introduced the Language interface)
- [esquery External class resolve PR](https://github.com/estools/esquery/pull/140) — The mechanism that enabled `matchesSelectorClass()`
