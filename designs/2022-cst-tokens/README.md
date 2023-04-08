- Repos: eslint/eslint, js-cst-tokens/cst-tokens
- Start Date: 2023-04-7
- RFC PR: #111
- Authors: Conrad Buck (github:conartist6)

# A literacy engine for plugins

## Summary

This RFC proposes investigating a possible dependency on `cst-tokens`, a virtual machine for executing linguistic algorithms against stream input in a left-to-right manner. `cst-tokens` is an explicit attempt to create a narrow waist for language definition: it's goal is to support the language APIs needed by major projects like ESLint, Prettier, and Babel while also creating a single language definition API that will allow a language defined once to be used for linting, pretty-printing, or transpilation.

## Motivation

Solving language definition in a way that eliminates the `m * n` problem for tools and languages creates immense value by fundamentally altering the way maintenance burdens are distributed within the overall community. This has two practical upshots:

- Significantly less maintenance may be needed overall, as tooling authors can avoid duplicated effort and language authors especially can avoid the efforts involved in having to define their language once for every tool they want to support.

- Programmers can be made more productive as individuals freed from infastructure maintenance build new intercompatible languages and tools. Effective literacy is increased by making things that should be easy as easy as they should be.

## Detailed Design

I will use this section to describe the detailed design of `cst-tokens`, as its mechanisms are the ones that I am proposing ESLint use. This section intends to demonstrate the way the mechanisms defined in `cst-tokens` can serve the same use cases as mechanisms in use currently (as combined with the changes proposed in #99).

### Architecture

The most obvious differences between the two designs lie in how languages are defined.

ESLint #99 defines languages as a combination of a `visitorKeys` object and a `parse` method. `cst-tokens` defines a language using a node parser grammar and a token parser grammar. This means that `cst-tokens` does not require any external parser because its grammar definitions themselves create a fully featured parser. Since that parser is quite slow compared to highly optimized parsers, `cst-tokens` is also able operate by traversing an existing AST (no parsing required) while still running the token parser, enabling it to create a consistent definition of trivia without redoing the work of resolving ambiguity (just as ESLint currently does).

#### Traversal

It is not fundamentally unreasonable to visit and AST node, say a `require` call, and to need to do some asynchronous work like resolving the required path against a filesystem. This requires that it be possible to suspend and resume the execution of a single visitor, which means that it must also be possible to suspend and resume execution of the traversal algorithm itself.

Both ESLint #99 and `cst-tokens` offer a specific and currently-novel solution to this problem. ESLint #99 turns all visitors into async functions and must use the `await` keyword anywhere a visitor is invoked. `cst-tokens` turns all visitors into generator functions and must use they `yield` keyword anywhere a visitor is invoked.

I argue that being forced to use `yield` agressively is far and away better than being forced to use `await` agressively, for two main resons:

- **Performance:** For reasons of predictability, every single `await` must allow the current tick and schedule work to continue in the next tick. Since most of the time nothing asynchronous is happening that needs to be awaited, the practice of running programs that `await` heavily is that they spend a lot of time just executing `await` itself.

- **Color:** When visitors are designed with async functions it is only possible to visit from an async function. All synchronous code is barred from use of the visitor API even when the caller knows that no asynchronous work will be necessary.

While these are the most important and probably the deciding concerns, it is also convenient that `cst-tokens` has no need to distinguish between `enter` and `exit` visitors.

#### Ranges

All match and productions results are ranges, where a range is `[startToken, endToken]`. An empty range is represented as `null`, and a range of one token is `[token, token]`. Iterating over a range is only possible with the help of the `context.prevTokens` `WeakMap`. An necessary idiosyncacy of this is that ranges can only be iterated backwards.

Ranges are also optimally structured for subrange exclusion. To skip the tokens in a subrange as you are iterating, simply skip from its `endToken` to its `startToken`. All together range iteration with skips looks like this:

```js
function *ownTokensFor(range, context) {
    if (!range) return;

    const { prevTokens, paths, ranges } = context;
    const [start, end] = range;
    const prev = prevTokens.get(start);

    for (let token = end; token !== prev; token = prevTokens.get(token)) {
        if (token.type === EndNode) {
            const path = paths.get(token);
            token = prevTokens.get(ranges.get(path)[0]);
        }

        yield token;
    }
}
```

#### AST Mutation for autofixing

ESLint's `TokenStore` defines a CST for a given AST, and implements the ability to query it. Unfortunately this store is not mutable. It is built as `new TokenStore(ast.tokens, ast.comments)`. This virtually guarantees that the only completely correct way to recover from any change, such as a fix being applied, is to throw away the entire structure and start over from scratch.

`cst-tokens` uses principles of Dynamic Programming to define CSTs as a fully composable data strcture suitable for immutable editing with amortized costs. As an example of how its architecture facilitates this, consider one of the fundamental limitations it places on ASTs: ASTs must never contain 2D arrays. This is not an egregious limitation on AST design, yet it creates the possibility of allowing an array of 100,000 items being stored as 100 arrays of 100 items. Thus the memory cost of generating a new immutable structure for some item within the array should only be 100 not 100,000.

#### Streaming

Due to the architecture of generator functions and the nature of grammars requesting that tokens be eaten, the `cst-tokens` engine can always suspend its execution in order to load more of the input. This eliminates the need to load the entire input into a string at the outset, and creates the possiblity of meaningfully processing infinte inputs.

#### Encapsulation

`cst-tokens` strongly embraces the architectural possibilities of `WeakMap`! To the best of my knowledge these have not been written about extensively.

The most obvious example is the `token` objects that are the namesake of `cst-tokens`. These objects are of the shape `{type, value}`, they have no prototype, and they are completely immutable. This is done to prevent users doing something like implementing syntax highlighting as `token.color = color`. You can still fundamentally do this though! The difference is that now you write `tokenColors.set(token, color)`.

Thus a concept of "weak layers" is born from the fact that you can have a single token (or path or state) and learn more and more about it as you gain access to more and more `WeakMaps`. In this way the definition of a token is extensible while also being predictable: the base object is always `{type, value}`. This is also a key perf concern as it ensures that core token handling code is (and stays) monomorphic!

Weak layers are expected to be useful for a variety of tasks like attaching scope and control flow information to existing trees.

Best of all the core doesn't need to know what all the possible layers are. For example you might write `scopeLayer = context.layers.get(Symbol.for('cst-tokens/layers/scope'))`. Since symbols are not valid `WeakMap` keys, `weakMaps` in this scenario is just a `Map` behind a simple facade that only allows `get`.

In this way an unlimited number of weak layers can be created, and the core can be responsible for verifying that layers are available. For example some plugin might require a scope layer to work correctly, but perhaps it is being run against a language which has defined no scope layer. The core would produce a nice error message explaining the problem.

#### Hoisting context

Because layers are part of `context` it is possible to retain the data about a tree stored in layers that would normally be garbage collected at the end of a traversal. The solution to this problem is the context hoisting pattern:

```js
const ctx = new Context();
const rootRange = traverse(tree, source, ctx);
```

Now the whole tree is in memory along with everything that is known about it. Additional investigations is needed into the high-level patterns that build on top of this basic pattern.

#### Grammar enhancers

One of the coolest aspects of working with generator grammars is that they're composable, just as pure functions would be. One of the most interesting parts of `cst-tokens` is how it defines whitespace for Javascript: by first defining an "atrivial" grammar which includes no whitespace, and then by enhancing that grammar by wrapping its productions with higher order generator functions which intercept instructions that match tokens and compute whether it is necessary to first match a space token. You can [already see this working](https://github.com/js-cst-tokens/cst-tokens/blob/trunk/play/languages/js-lite/enhancers/trivia.js).


## Documentation

One of the benefits of creating strong and independent abstractions is that they can have a strong base of documentation. Documentation is as yet lacking though.

## Drawbacks

Building for everyone is a lot more work! ESLint's bread is buttered on the side of its curent users. It is probable that providing unified literacy APIs would increase the overall number of users in the ecosystem, maybe even drastically, but lowering the barriers to competition between tools in the ecosystem may also ultimate weaken ESLint's position of dominance in the market.

Rejecting this proposal is not free from drawbacks either, as the author of this proposal is fully committed to creating a unified API for lanugage definition and code literacy. ESLint would be forced into a competition with a new generation of tools that were strictly more powerful and easier to use than ESLint.

## Backwards Compatibility Analysis

Backwards compatibility could be maintained if there were multiple supported language APIs for plugins. Plugins written for the `cst-tokens` language API could declare as much and would be given that API, while plugins written for the legacy SourceCode API would remain unchanged and would continue to be called via that API.

The new API is strictly incompatible with the old not least because the old API used trees with `parent` pointers.

An interesting facet of backwards compatibility is that a core competency of this code is in upgrading code. `cst-tokens` has the formatting sensitivity of ESLint but the power of Babel, making it a more-than-capable environment for writing scripts that help users upgrade plugins from an older style to a newer style. It's generator-based architecture even creates the possibility of creating interactive codemods which keep the user in the loop to resolve critical ambiguities.

## Alternatives

#99

## Open Questions

`cst-tokens` has a variety of its own open questions and is under ongoing develpment.

If `cst-tokens` were to reach `1.0.0` and become core infrastructure for major projects like ESLint and Babel, establishing formal governance for the project would be important.

Should this 

## Help Needed

Yes, absolutely!

## Frequently Asked Questions

Q: Who are you?

A: No one of consequence.

## Related Discussions

??
