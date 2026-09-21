- Repo: eslint/eslint
- Start Date: 2026-05-23
- RFC PR: https://github.com/eslint/rfcs/pull/148
- Authors: Kuldeep2822k

# Add `selectorClassNodeTypes` to the Language Interface

## Summary

This RFC proposes adding an optional `selectorClassNodeTypes` property (`Map<string, Array<string>>`) to the `Language` interface, so a language can declare which AST node types a pseudo-class selector (such as `:function`) can match. This moves JavaScript-specific selector analysis out of `lib/linter/esquery.js` and lets language plugins participate in selector static analysis.

## Motivation

When rules register selectors, ESLint's selector engine ([`lib/linter/esquery.js`](https://github.com/eslint/eslint/blob/main/lib/linter/esquery.js)) statically analyzes the parsed selector to find candidate AST node types. For example, `:function` only matches `FunctionDeclaration`, `FunctionExpression`, and `ArrowFunctionExpression`, so the traverser indexes the selector under those node types and skips selector matching on other nodes.

Currently, this optimization is hardcoded for JavaScript functions in the `analyzeParsedSelector()` helper in `lib/linter/esquery.js`:

```js
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

This creates a few practical issues:

- **Language interface consistency.** RFC #99 established the `Language` interface so ESLint core remains agnostic to specific language syntax. Keeping JS-specific node names hardcoded in the core selector engine conflicts with that abstraction.
- **Plugin parity.** `matchesSelectorClass()` allows language plugins to define runtime pseudo-class matching, but there is no mechanism for plugins to declare candidate node types for static analysis. Because the `case "class"` branch is hardcoded, `:function` narrows to the JS function node types regardless of the active language, while every other class selector returns `null`, so language plugins can't declare candidate node types to narrow traversal.
- **Silently dead selectors in other languages.** In a language without `matchesSelectorClass()` (for example `@eslint/json` today), class selectors like `:statement` or `:function` silently match nothing — esquery returns `false` for every class when options provide a `nodeTypeKey` but no `matchClass`. There is no error and no match, so class selectors are unusable in these languages, and language authors currently have to hand-write runtime matching logic to fix that.
- **Direct TODO in core.** The comment in `esquery.js` explicitly notes that this mapping should be abstracted into the language implementation.

Adding `selectorClassNodeTypes` to the `Language` interface moves this mapping to the language object, where pseudo-class semantics are already defined.

## Detailed Design

I propose storing the pseudo-class-to-node-type mapping on the language object and having core read it during selector static analysis. The change touches three places: the `Language` interface, the JavaScript language object, and two spots in the selector engine.

### 1. `Language` Interface Property

Add an optional `selectorClassNodeTypes` field to the `Language` interface in `@eslint/core`:

```ts
interface Language<Options extends LanguageTypeOptions = LanguageTypeOptions> {
    // ... existing properties ...
    nodeTypeKey: string;
    visitorKeys?: Record<string, readonly string[]>;

    /**
     * Map of lowercase selector class names to the AST node types that can match them.
     * Used for static analysis to narrow candidate node types during traversal.
     */
    selectorClassNodeTypes?: Map<string, Array<string>>;

    matchesSelectorClass?(
        className: string,
        node: Options["Node"],
        ancestry: Options["Node"][],
    ): boolean;
}
```

### 2. Static Analysis vs. Runtime Matching

Static analysis and runtime matching serve different roles.

During selector parsing, `analyzeParsedSelector()` converts the class name to lowercase and looks it up with `selectorClassNodeTypes.get(selector.name.toLowerCase())`. If the class is in the map, it returns the array of node types, and the traverser indexes the selector under them (`enterSelectorsByNodeType` / `exitSelectorsByNodeType`). If the class is missing, or `selectorClassNodeTypes` is undefined, static analysis returns `null`, which places the selector in the traverser's any-type list (`anyTypeEnterSelectors` / `anyTypeExitSelectors`) and evaluates it on every node during traversal.

During traversal, `matchesSelectorClass()` performs the actual match. A pseudo-class that returned `null` from static analysis, like `:statement`, is still filtered correctly here by matching only nodes ending in `Statement` or `Declaration`. An unknown class like `:foo` throws `Error: Unknown class name: foo`, as it does today.

The rule for plugin authors follows from this: the node types in `selectorClassNodeTypes` must cover every node type `matchesSelectorClass()` can match. When a pseudo-class is open-ended or dynamic, leave it out of the map so it falls back to `null` and evaluates against all nodes. Returning an empty array `[]` instead prunes the selector from traversal completely.

### 3. JavaScript Language Implementation

In `lib/languages/js/index.js`, export `selectorClassNodeTypes` on the JS language object:

```js
module.exports = {
    fileType: "text",
    lineStart: 1,
    columnStart: 0,
    nodeTypeKey: "type",
    visitorKeys: evk.KEYS,

    selectorClassNodeTypes: new Map([
        [
            "function",
            [
                "FunctionDeclaration",
                "FunctionExpression",
                "ArrowFunctionExpression",
            ],
        ],
    ]),

    matchesSelectorClass(className, node, ancestry) {
        // ... existing implementation unchanged ...
    },
};
```

Pseudo-classes with open-ended or pattern-based matching (`statement`, `declaration`, `pattern`, `expression`) are omitted from `selectorClassNodeTypes`, safely returning `null` during static analysis and relying on runtime matching.

### 4. Example: CSS Language Plugin

A CSS language plugin could declare candidate node types for its pseudo-classes:

```js
export const cssLanguage = {
    fileType: "text",
    nodeTypeKey: "type",
    visitorKeys: {
        StyleSheet: ["children"],
        Rule: ["prelude", "block"],
        Atrule: ["prelude", "block"],
        Declaration: ["property", "value"],
        Block: ["children"],
    },

    selectorClassNodeTypes: new Map([
        ["rule", ["Rule", "Atrule"]],
        ["at-rule", ["Atrule"]],
        ["atrule", ["Atrule"]],
        ["declaration", ["Declaration"]],
    ]),

    matchesSelectorClass(className, node, ancestry) {
        switch (className.toLowerCase()) {
            case "rule":
                return node.type === "Rule" || node.type === "Atrule";
            case "at-rule":
            case "atrule":
                return node.type === "Atrule";
            case "declaration":
                return node.type === "Declaration";
            default:
                return false;
        }
    },
};
```

### 5. Default implementation in `@eslint/plugin-kit`

Rather than requiring every language plugin to hand-write both the map and a `matchesSelectorClass()` that duplicates it, `@eslint/plugin-kit` can provide a default runtime implementation that mirrors the map. A language that declares `selectorClassNodeTypes` and does not need custom matching logic can omit `matchesSelectorClass()` entirely and get this behavior automatically:

```js
const nodeTypes = this.selectorClassNodeTypes?.get(className.toLowerCase());
if (nodeTypes) {
    return nodeTypes.includes(node.type);
}
return false;
```

This gives map-backed classes (`:function` in the JS language, `:rule` in the CSS example above) working runtime matching for free. Languages that need richer matching — like the JS language's suffix-based classes — keep their own `matchesSelectorClass()` implementation, which takes precedence.

Note: the JS language itself needs its custom implementation to keep `:statement`, `:declaration`, `:pattern`, and `:expression` working, because those classes match by node-type *suffix* rather than an enumerable list. That asymmetry is discussed under Open Questions.

### 6. Core `esquery.js` Changes

In `lib/linter/esquery.js`, `analyzeParsedSelector()` accepts `selectorClassNodeTypes`:

```diff
-function analyzeParsedSelector(parsedSelector) {
+function analyzeParsedSelector(parsedSelector, selectorClassNodeTypes) {
     let attributeCount = 0;
     let identifierCount = 0;

     function analyzeSelector(selector) {
         switch (selector.type) {
             // ...
             case "class":
-                // TODO: abstract into JSLanguage somehow
-                if (selector.name === "function") {
-                    return [
-                        "FunctionDeclaration",
-                        "FunctionExpression",
-                        "ArrowFunctionExpression",
-                    ];
-                }
-                return null;
+                return selectorClassNodeTypes?.get(selector.name.toLowerCase()) ?? null;
         }
     }
```

`parse()` accepts `selectorClassNodeTypes` and keys the cache by the map reference:

```diff
+const DEFAULT_SELECTOR_CACHE = new Map();
+const selectorCacheByMap = new WeakMap();
+
+function getMapCache(selectorClassNodeTypes) {
+    if (!selectorClassNodeTypes) {
+        return DEFAULT_SELECTOR_CACHE;
+    }
+    let cache = selectorCacheByMap.get(selectorClassNodeTypes);
+    if (!cache) {
+        cache = new Map();
+        selectorCacheByMap.set(selectorClassNodeTypes, cache);
+    }
+    return cache;
+}

-function parse(source) {
-    if (selectorCache.has(source)) {
-        return selectorCache.get(source);
-    }
+function parse(source, selectorClassNodeTypes) {
+    const cache = getMapCache(selectorClassNodeTypes);
+
+    if (cache.has(source)) {
+        return cache.get(source);
+    }

     const cleanSource = source.replace(/:exit$/u, "");
     const parsedSelector =
         trySimpleParseSelector(cleanSource) ?? tryParseSelector(cleanSource);
     const { nodeTypes, attributeCount, identifierCount } =
-        analyzeParsedSelector(parsedSelector);
+        analyzeParsedSelector(parsedSelector, selectorClassNodeTypes);

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

Keying the cache by the `Map` reference assumes `selectorClassNodeTypes` is immutable after the language is created, the same assumption already made for `visitorKeys`. If a language mutated the map after a selector was cached, the cached analysis would go stale, so languages should treat it as a fixed, read-only property.

### 7. `SourceCodeTraverser` Changes

In `lib/linter/source-code-traverser.js`, pass `selectorClassNodeTypes` through `esqueryOptions`:

```diff
 // In SourceCodeTraverser.prototype.traverseSync()
 traverseSync(sourceCode, visitor, { steps } = {}) {
     const esquery = new ESQueryHelper(visitor, {
         visitorKeys: sourceCode.visitorKeys ?? this.#language.visitorKeys,
         fallback: vk.getKeys,
         matchClass: this.#language.matchesSelectorClass ?? (() => false),
         nodeTypeKey: this.#language.nodeTypeKey,
+        selectorClassNodeTypes: this.#language.selectorClassNodeTypes,
     });
```

```diff
 // In the ESQueryHelper constructor
 constructor(visitor, esqueryOptions) {
     this.visitor = visitor;
     this.esqueryOptions = esqueryOptions;
+    this.selectorClassNodeTypes = esqueryOptions?.selectorClassNodeTypes;
     // ...
     visitor.forEachName(rawSelector => {
-        const selector = parse(rawSelector);
+        const selector = parse(rawSelector, this.selectorClassNodeTypes);
         // ... rest unchanged
     });
 }
```

## Documentation

This feature requires updating the following:

* The `Language` interface type definitions in [`@eslint/core`](https://github.com/eslint/rewrite/tree/main/packages/core), to add the optional `selectorClassNodeTypes` property.
* The [Languages](https://eslint.org/docs/latest/extend/languages) documentation, which lists the properties and methods of the `Language` object. `selectorClassNodeTypes` should be documented alongside `matchesSelectorClass()`, noting that the node types it declares must cover every node type `matchesSelectorClass()` can match for the same class.

No blog announcement is needed. This is an additive, opt-in extension point for language plugin authors.

## Drawbacks

- **Desynchronization risk:** If a language plugin updates `matchesSelectorClass()` to match a new AST node type but forgets to update `selectorClassNodeTypes`, the traverser skips dispatching the selector on those nodes, and rules silently fail to execute with no error or warning.
- **Limited to enumerable types:** Dynamic or pattern-based pseudo-classes like `:statement` or `:expression` cannot be statically narrowed and must return `null` to evaluate against all visited nodes.

## Backwards Compatibility Analysis

1. **`selectorClassNodeTypes` is optional.** If omitted, static analysis returns `null` for class selectors, meaning candidate nodes are not narrowed during parsing and all nodes are evaluated during traversal. This matches the current behavior for all non-`:function` pseudo-classes.
2. **JavaScript behavior is unchanged.** The JS language object declares the same three function node types currently in `esquery.js`.
3. **`parse(source)` without options.** Calling `parse` without a second argument falls back to `DEFAULT_SELECTOR_CACHE` and returns `null` for all class selectors. This is a change from today, where `parse(":function")` returns the three JS function node types from the hardcoded branch. It is internal-only: the traverser always passes the active language's `selectorClassNodeTypes`, so the JS language still resolves `:function` to those node types in practice.
4. **Existing plugins.** Language plugins without this property continue to function without modification.
5. **Unknown class names: asymmetric by language.** On the JS language, an unknown class like `:foo` throws `Error: Unknown class name` from `matchesSelectorClass()`, exactly as it does today. On languages without `matchesSelectorClass()` (e.g. `@eslint/json`), unknown classes already silently match nothing — esquery's `matchClass` returns `false`. This RFC preserves both behaviors unchanged. The plugin-kit default must return `false` (not throw) to avoid introducing a crash into languages that currently silently no-op, and to match core's existing fallback at `source-code-traverser.js` (`matchClass: this.#language.matchesSelectorClass ?? (() => false)`).

## Alternatives

### 1. Method: `getSelectorClassNodeTypes(className)`

The initial draft proposed a method on `Language`. A plain `Map` property was chosen instead based on maintainer feedback: mapping pseudo-class names to static node type arrays is fixed data rather than dynamic behavior, so a property map is simpler and avoids introducing an unnecessary method to the interface.

### 2. Check JS Language Identity in Core Helpers

Move the `:function` logic to a helper in core that checks whether the active language is the JS language. This would remove the inline `case "class"` branch from `esquery.js` but keeps the core coupled to JS and does not provide an extension point for language plugins.

### 3. Infer Node Types via `matchesSelectorClass()` Probing

Call `matchesSelectorClass()` against mock nodes at parse time to infer types. This is not feasible because ESLint core does not maintain a comprehensive registry of all AST node types for arbitrary languages, and creating dummy nodes for every type would be slow and fragile.

## Open Questions

### 1. Pruning vs any-type fallback for undeclared classes

Should a class selector that isn't declared in `selectorClassNodeTypes` match all nodes, or should it be pruned from traversal entirely?

Today the two code paths disagree, and this RFC preserves that behavior:

- Static analysis returns `null` for an undeclared class, which places the selector in the any-type list so it is evaluated on every node during traversal.
- At runtime, `matchesSelectorClass()` decides the actual match. For a truly unknown class (one the language doesn't define), it throws `Error: Unknown class name` as it does today.

So an undeclared-but-known class like `:statement` safely falls back to evaluating on all nodes, while an unknown class like `:foo` still throws. Whether "match all nodes" is the right default for undeclared classes, or whether they should instead be pruned, is the main question I'd like reviewers to weigh in on.

### 2. Should `matchesSelectorClass()` eventually be deprecated in favor of the map?

The JS language's class selectors come in two shapes:

- `:function` matches an enumerable list of node types (`FunctionDeclaration`, `FunctionExpression`, `ArrowFunctionExpression`), so it fits cleanly into the `selectorClassNodeTypes` map.
- `:statement` matches by two suffix tests joined by a subtype fallthrough — everything ending in `Statement`, plus everything ending in `Declaration`, on the model `Declaration <: Statement`. TypeScript's `TSInterfaceDeclaration` is caught by the second, not the first. Neither a finite node-type list nor a single suffix pattern expresses a two-suffix union over an open dialect set.

I tested the map-only variants empirically against ESLint v10.11.0 (`Linter.verify` against the locally installed source at `D:/ai-brain/eslint`):

| Mode | What happens |
|---|---|
| Map powers static analysis only, method removed (core wires `() => false`) | Every class selector silently stops matching, **including `:function`** — esquery still needs `matchClass` for the final match decision, so the traverser's fallback vetoes all candidates. `padding-line-between-statements` loses `:statement` hits (1→0). `no-shadow-restricted-names` loses `:function` hits (1→0). |
| Map powers static analysis **and** runtime, throwing on unknown | `:function` works, but `:statement`/`:expression` throw `Error: Unknown class name` mid-lint — hard crash of `padding-line-between-statements` and any user config using `:statement` in `no-restricted-syntax`. |
| Map powers static analysis **and** runtime, returning `false` on unknown | `:function` works, but suffix classes silently die — including on TS dialect nodes (`TSInterfaceDeclaration`) that suffix matching currently catches. |

**Ancestry impossibility.** A separate problem dooms full deprecation: the method signature is `matchesSelectorClass(className, node, ancestry)` — it's a predicate over both the node and its ancestry. The `:expression` class exemplifies this:

```
node.type="Identifier"  parent = CallExpression  -> :expression = true
node.type="Identifier"  parent = MetaProperty    -> :expression = false
node.type="Identifier"  no ancestry              -> :expression = true
```

Same node type, opposite answers, decided by ancestry[0]. No declarative node-type representation — `Map`, glob, or `selectorClassPatterns` — can encode a parent check. The method isn't just a convenience for enumerable types; its `(node, ancestry)` signature is the only place ancestry-sensitive matching lives.

**Interaction with the plugin-kit default.** One subtlety worth flagging: the deprecation idea only crashes in combination with a throwing plugin-kit default. Today, the hardcoded `:function` branch in `lib/linter/esquery.js`'s `analyzeParsedSelector()` `case "class"` sub-branch prunes static analysis to 3 JS node types before the language matcher is ever called, so a JSON file with `:function` never reaches the matcher — it returns `null` at static analysis and never dispatches. Once that branch is removed (per this RFC), `:function` falls through to the any-type list and gets evaluated on every JSON node. Combine that with a throwing default and you get `Error: Unknown class name: function` mid-lint. See the cross-cutting interaction matrix above.

**Recommendation.** `matchesSelectorClass()` cannot be removed while JS still needs suffix matching and ancestry checks. "Deprecation" should mean "plugins rarely need to write one" (via the plugin-kit default for map-backed classes), not "remove the method." Full removal requires a new data-driven representation for ancestry-sensitive matching — that's future work, not this RFC's scope.

## Cross-Cutting Interaction Matrix

The RFC's core change (removing the hardcoded `:function` branch) and fasttime's plugin-kit default (mirroring the map at runtime) interact in a way that neither change is safe alone against:

| Core state | `@eslint/json` adopts kit default | `@eslint/json` without default |
|---|---|---|
| Current core (hardcoded `:function` branch) | OK — `:function` is pruned at static analysis, matcher never called | OK — silent no-match (current behavior) |
| Post-RFC core (hardcoded branch removed) | **CRASH** — `:function` falls to any-type list, throwing default raises `Unknown class name: function` | OK — `:function` silently matches nothing |

The crash needs both changes to exist. Reproduced on eslint v10.11.0 (<code>D:/ai-brain/eslint</code>) with <code>verify-rfc148-battery.cjs</code>, which patches a copy of the installed core to drop the hardcoded <code>:function</code> branch and then assigns a throwing map-mirror as <code>@eslint/json</code>'s <code>matchesSelectorClass</code>.

## Help Needed

I'm able to implement this myself and will open the reference implementation in `eslint/eslint` once the RFC is accepted.

Measuring the traversal impact will need a dedicated benchmark built around a `:function`-based rule. Recommended rules like `no-shadow-restricted-names` already register `:function`, so the general `npm run test:performance` suite exercises this path in aggregate but can't isolate its cost; a focused benchmark is needed for a clean signal.

## Frequently Asked Questions

**Why use a `Map` instead of a plain object (`Record<string, Array<string>>`)?**

A plain object risks inherited-key collisions: a lookup for a class name like `constructor` or `toString` returns an `Object.prototype` member unless you use a null-prototype object or `hasOwn` checks. Verified empirically — `constructor`, `toString`, `valueOf`, `__proto__`, `hasOwnProperty` are all truthy on an empty object via property lookup, while `Map.get` returns `undefined` (safe) for all of them. This matters because esquery lowercases the class name, and `constructor` is a legal selector class (`:constructor`). A `Record`-based design would have made `:toString` silently misbehave by returning an inherited function instead of `undefined`. `Map` also gives explicit `.get()` / `.has()` semantics and works as a stable object reference for keying the `WeakMap` selector cache. (`visitorKeys` stays a plain object because its keys are fixed AST node type names, which don't have this problem.)

**Why does JavaScript only declare `:function` and omit `:statement` or `:expression`?**

Statements and expressions each cover dozens of ESTree node types, and dialects like JSX or TypeScript add more. Hardcoding those lists in core would be brittle across AST versions. Leaving them out lets static analysis fall back to `null` (evaluate on all nodes) while `matchesSelectorClass()` matches them at runtime via suffix checks like `*Statement` and `*Declaration`.

**What happens if a plugin's `selectorClassNodeTypes` omits a node type that `matchesSelectorClass()` matches?**

The traverser skips dispatching the selector on those node types, which silently drops matches. The rule of thumb is to declare a class in `selectorClassNodeTypes` only when its node types are fixed and exhaustive; when they aren't, omit the entry and fall back to `null`. Returning an empty array `[]` instead prunes the selector from traversal completely — neither the static analyzer nor the runtime matcher ever invokes it. This is a supported way for a language to deliberately retire a class.

**Does the kit default returning `false` lose the "unknown class throws" safety?**

It defers it. The throwing behavior for truly unknown classes (`:foo`) can be caught once at config time — when rules register selectors against the active language's known classes — rather than per-node during traversal. A typo like `:statment` surfaces up front with a clear message, not as a crash mid-lint. The kit default returning `false` prevents crashes in languages that currently silently no-op (e.g. `@eslint/json`); config-time validation restores the loud-failure guarantee for typos without the crash risk.

**How does this interact with custom parsers (like `@typescript-eslint/parser`) using the JavaScript language?**

`:function` narrows to the standard ESTree function types. For parser-specific node types like `TSDeclareFunction`, rules can target them with a direct node selector, or the parser can ship its own `Language` plugin with its own `selectorClassNodeTypes` if it wants custom pseudo-class semantics.

## Related Discussions

- [Discussion #20856: Should we move JS-specific ESQuery analysis logic into the Language interface?](https://github.com/eslint/eslint/discussions/20856)
- [RFC #99: ESLint Language Plugins](https://github.com/eslint/rfcs/pull/99)
- [RFC PR #148: Pull Request for this RFC](https://github.com/eslint/rfcs/pull/148)
- [esquery PR #140: External class resolution mechanism](https://github.com/estools/esquery/pull/140)
