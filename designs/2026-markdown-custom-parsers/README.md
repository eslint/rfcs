- Repo: eslint/markdown
- Start Date: 2026-08-04
- RFC PR: https://github.com/eslint/rfcs/pull/152
- Authors: lumirlumir

# Support `languageOptions.parser` for Markdown

## Summary

This RFC proposes adding `languageOptions.parser` to `@eslint/markdown` so users can replace the built-in [`mdast-util-from-markdown`](https://github.com/syntax-tree/mdast-util-from-markdown) parser with another synchronous parser. The immediate motivation is to integrate an optional Rust-based parser, but the API is not Rust-specific and can be used by other Markdown parsers. The built-in JavaScript parser remains the default, so existing configurations and browser usage are unchanged.

## Motivation

Markdown parsing can dominate the time spent linting Markdown. In the prototypes described in [eslint/markdown#703](https://github.com/eslint/markdown/issues/703) and [eslint/markdown#706](https://github.com/eslint/markdown/pull/706), parsing took approximately **75 ms** of a **100 ms** run. Replacing it with `@eslint-markdown/parser`, a Rust-based parser built on [`satteri`](https://github.com/bruits/satteri), reduced parsing time to approximately **2 ms** and the total runtime to approximately **27 ms**, making the overall run approximately **3.5 times faster**. Although these results are preliminary, they demonstrate that selecting a faster parser can materially improve linting performance.

`@eslint/markdown` currently uses `mdast-util-from-markdown` from the mdast ecosystem. [`satteri`](https://github.com/bruits/satteri) is a high-performance Markdown and MDX processor that parses Markdown in Rust and exposes the result to JavaScript as mdast. Its [`markdownToMdast()`](https://satteri.bruits.org/docs/entry-points/#trees-without-compiling) API supports CommonMark and GFM as well as frontmatter and math parsing. Initial experiments indicate that it can support most of the cases already implemented by `@eslint/markdown`, although broader compatibility testing is still required.

Changing the default parser to `satteri` is not currently suitable. Although `satteri` provides a browser-targeted WebAssembly binding, adopting it as the default would add platform-specific native binaries to the default Node.js dependency graph and a WebAssembly artifact and initialization path to the default browser dependency graph. This would increase distribution size and introduce additional runtime and bundler compatibility requirements for every user, including users who do not need the performance improvement.

A parser option solves this by keeping the current portable parser as the default while allowing users to opt into another implementation. It also allows users to provide their own mdast-related parser with the syntax plugins they need. This proposal adds the option specifically to the Markdown language implementation. It does not propose a new parser mechanism for other ESLint language plugins, such as JSON and CSS.

## Detailed Design

### Goals and Non-Goals

This proposal has the following goals:

1. Allow a configuration to select a custom parser for `markdown/commonmark` or `markdown/gfm`.
2. Preserve the existing behavior when no custom parser is configured.
3. Define a small parser contract that can be implemented by JavaScript, native Node.js addons, or WebAssembly packages.
4. Allow parsers that preserve the mdast contract expected by `MarkdownSourceCode` and existing `@eslint/markdown` rules.

This proposal does not:

- add a Rust parser to `@eslint/markdown` by default.
- change the default parser.
- define how native or WebAssembly parser packages are built or distributed.
- support asynchronous parsers.
- guarantee that every `@eslint/markdown` rule works with every custom parser.

### New Language Options

`MarkdownLanguageOptions` will gain one property:

```ts
export interface MarkdownLanguageOptions extends LanguageOptions {
    frontmatter?: false | "yaml" | "toml" | "json";
    math?: boolean;
    parser?: MarkdownParser;
}
```

`parser` is an object with a synchronous `parse()` method. Parser-specific options can be added directly to `languageOptions` like `LanguageOptions`, `MarkdownLanguageOptions` permits additional properties, and those properties are forwarded to the parser.

The parser contract is:

```ts
import type { Root } from "mdast";
import type { ObjectMetaProperties } from "@eslint/core";

/** @deprecated Use `MarkdownParserMode` instead. */
export type ParserMode = MarkdownParserMode;

export type MarkdownParserMode = "commonmark" | "gfm";

export type MarkdownParser = ObjectMetaProperties & {
	parse(
		text: string,
		options: MarkdownLanguageOptions & {
			mode: MarkdownParserMode;
			parser?: never;
		},
	): Root;
};
```

`ParserMode` was already exported, so it remains as a deprecated alias for backwards compatibility. New code should use `MarkdownParserMode`.

Parser metadata is optional and informational. `@eslint/markdown` does not change parsing behavior based on `meta`.

The parser is an object rather than a bare function to follow the shape used by ESLint parser packages and to leave room for metadata without changing the invocation API. Like `fromMarkdown()`, the method receives the Markdown text as its first argument and parser options as its second argument.

This RFC proposes only `parse()` for the initial implementation. Whether Markdown parsers need a `parseForESLint()`-style method or a way to return additional parser services is left for a future proposal after a concrete use case is identified. `MarkdownLanguage` already owns creation of `MarkdownSourceCode`, so the Rust prototype only needs to produce the syntax tree.

### Parser Invocation

`MarkdownLanguage#parse()` selects the configured parser, falling back to the parser in `defaultLanguageOptions`, and calls its `parse()` method with the Markdown source text and a newly created options object:

```js
const {
    parser = this.defaultLanguageOptions.parser,
    ...restLanguageOptions
} = context?.languageOptions ?? {};

// ...

const root = parser.parse(text, {
    ...restLanguageOptions,
    mode: this.#mode,
});
```

ESLint merges `defaultLanguageOptions` before parsing, so `frontmatter` and `math` have their effective default value of `false`. `mode` comes from the configured language: `markdown/commonmark` supplies `"commonmark"`, and `markdown/gfm` supplies `"gfm"`.

The `parser` property itself is omitted from the options passed to `parse()`, and the language's `mode` is always applied after the other language options. All other standard and parser-specific language options are forwarded unchanged.

The default parser is an object stored in `defaultLanguageOptions.parser`. Its `parse()` method removes `mode` from the forwarded language options and calls `mdast-util-from-markdown` with the existing micromark and mdast extensions. This keeps the built-in parser's behavior unchanged while using the same parser invocation path as a custom parser.

The parser call remains synchronous. Returning a promise is not supported.

### Rust Parser Integration

The proposed `@eslint-markdown/parser` package will use [`satteri`](https://github.com/bruits/satteri) under the hood, which exposes its Rust implementation to JavaScript through Node-API. The intended data flow is:

1. `@eslint/markdown` passes the Markdown source string and language options to `@eslint-markdown/parser`, which wraps `satteri` behind the `MarkdownParser` contract expected by `@eslint/markdown`.
2. `@eslint-markdown/parser` passes the Markdown source string to `satteri` through its Node-API binding.
3. `satteri` parses the source and stores the resulting mdast nodes in a Rust-managed memory pool called an arena.
4. The Rust arena is encoded into a compact binary buffer and passed to JavaScript.
5. JavaScript reads the buffer and materializes the mdast nodes without serializing and deserializing the complete tree as JSON.
6. The materialized [mdast `Root`](https://github.com/syntax-tree/mdast#root) is returned to `@eslint-markdown/parser`.
7. `@eslint-markdown/parser` returns the compatible mdast `Root` from its `parse()` method to `@eslint/markdown`.

This does not use `execSync()` or spawn a child process. Node.js loads the compiled `.node` addon as a dynamic library, and JavaScript calls its exported function directly. The call therefore remains synchronous and runs on the Node.js main thread, just like the current call to `fromMarkdown()`.

The `@eslint-markdown/parser` package, including its platform-specific native binaries and any future WebAssembly build, is separate from `@eslint/markdown`. Its implementation and distribution are not required in order to add the general parser option.

### AST Compatibility

The supported compatibility target for a custom parser is an [mdast `Root`](https://github.com/syntax-tree/mdast#root). A parser intended to work with the bundled `@eslint/markdown` rules should produce the same standard node shapes and position information as the built-in parser.

The complete compatibility contract is the mdast and unist data used by `@eslint/markdown`:

- nodes use the `type` property.
- the top-level node is an mdast `Root`.
- child relationships follow mdast.
- nodes that may be reported by rules contain accurate unist `position` data.
- standard CommonMark, GFM, frontmatter, and math nodes match the shapes produced by the current parser.

`@eslint/markdown` will not recursively validate the returned AST because doing so would add overhead to every parse. Parser authors are responsible for compatibility. Parser-specific nodes may be added, but existing rules are only expected to work when the nodes they consume remain compatible.

The public `MarkdownParser` type requires `parse()` to return an mdast `Root`. At runtime, `@eslint/markdown` validates the parser interface but does not recursively validate the returned tree. Parser authors are responsible for producing a compatible mdast tree; incompatible trees may fail during source code construction or rule execution and are outside the supported contract.

### Error Handling

The existing error behavior remains in place. If a custom parser throws an error, `MarkdownLanguage#parse()` catches it and returns an unsuccessful `ParseResult` instead of rethrowing it.

### Validation

`MarkdownLanguage#validateLanguageOptions()` will add the following checks:

- `parser`, when present, must be a non-null object with a callable `parse` property.

The existing validation for `frontmatter` and `math` remains unchanged.

### Configuration Example

A future Rust-based parser package could be selected as follows:

```js
import markdown from "@eslint/markdown";
import parser from "@eslint-markdown/parser";

export default [
    {
        files: ["**/*.md"],
        plugins: {
            markdown,
        },
        language: "markdown/gfm",
        languageOptions: {
            frontmatter: "yaml",
            math: true,
            parser,
            parserSpecificOption: true, // Parser-specific option
        },
    },
];
```

A JavaScript parser can implement the same interface:

```js
import { fromMarkdown } from "mdast-util-from-markdown";
import { customSyntax } from "micromark-extension-custom-syntax";
import { customSyntaxFromMarkdown } from "mdast-util-custom-syntax";

const parser = {
    meta: {
        name: "example-markdown-parser",
        version: "1.0.0",
    },
    parse(text) {
        return fromMarkdown(text, {
            extensions: [customSyntax()],
            mdastExtensions: [customSyntaxFromMarkdown()],
        });
    },
};
```

### Implementation Approach

The implementation in `eslint/markdown` will include:

1. Exporting `MarkdownParserMode`, the deprecated `ParserMode` alias, and `MarkdownParser`, and adding `parser` to `MarkdownLanguageOptions` in `src/types.ts`.
2. Extending `MarkdownLanguage#validateLanguageOptions()` with the validation described above.
3. Representing the built-in parser in `defaultLanguageOptions.parser`, then selecting and invoking the configured or default parser in `MarkdownLanguage#parse()`.
4. Adding unit tests with a small parser object for invocation, option forwarding, invalid configuration, thrown errors, and both Markdown modes.
5. Adding `@eslint-markdown/parser` as a development dependency and running integration tests against both the default JavaScript parser and the Rust-based parser. These tests will cover CommonMark, GFM, frontmatter, math, source locations, and the bundled rules.
6. Adding type tests for the public parser interfaces and configuration options.

`@eslint-markdown/parser` remains an optional runtime dependency and is developed and released separately. It is included in `@eslint/markdown` only as a development dependency for integration and compatibility testing.

## Documentation

The `@eslint/markdown` README will document:

- the `parser` language option and how additional language options are forwarded to it.
- the required synchronous `parse(text, options)` contract.
- the mdast and positional-information requirements.
- an example using an external parser package.
- an example wrapping `mdast-util-from-markdown` with custom extensions.
- the fact that parser behavior, platform support, and rule compatibility are the parser author's responsibility.

After the parser option is released and an experimental version of `@eslint-markdown/parser` is published, a blog post would be useful to invite users to try the Rust-based parser and provide feedback. The post can explain how to opt in and share the available compatibility and benchmark results, clearly identifying any preliminary findings.

## Drawbacks

Custom parsers may handle Markdown edge cases differently, resulting in different AST nodes, source locations, or fixes. They may also introduce installation and platform-specific issues, especially when using native binaries.

## Backwards Compatibility Analysis

This proposal is additive. Configurations that do not specify `languageOptions.parser` continue to use `mdast-util-from-markdown` with the same GFM, frontmatter, and math extensions as today. The default dependency graph and browser behavior do not change.

Only configurations using the new option are affected by the new validation and custom parser behavior. A custom parser that returns an incompatible tree may produce a parse error or rule failures, but no existing configuration can encounter that behavior before opting into the feature.

## Alternatives

### Replace the Default Parser

`@eslint/markdown` could replace `mdast-util-from-markdown` with a Rust-based parser once it is sufficiently compatible and mature. This would make the performance improvement available without a configuration change. Although `satteri` already distributes platform-specific Node-API binaries and a browser-targeted WebAssembly binding, adopting it as the default would add those artifacts to the default dependency graph and introduce a WebAssembly initialization path and additional bundler compatibility requirements for browser consumers. Because the integration remains experimental, this proposal keeps the portable JavaScript parser as the default and makes the Rust-based parser opt-in while its compatibility and distribution behavior are evaluated.

### Accept Bare Parser Functions

`languageOptions.parser` could accept a function with the same signature as `fromMarkdown()`. A parser object was selected because it matches the package shape familiar from ESLint's JavaScript parsers, supports optional metadata, and can grow additional parser-level capabilities in a backwards-compatible way if a future RFC requires them.

### Require and Validate mdast Trees

`@eslint/markdown` could require every custom parser to return mdast and recursively validate the tree before running rules. This would provide a stronger compatibility boundary, but the validation would add runtime overhead and still could not detect every semantic difference between Markdown parsers. The proposed API documents mdast as the compatibility target without performing recursive runtime validation.

## Open Questions

1. Does the Markdown parser contract need a `parseForESLint()`-style method? ESLint parsers can use `parseForESLint()` to return additional values such as parser services along with the AST. In this proposal, `MarkdownLanguage` creates `MarkdownSourceCode`, and the current integration only needs the parser to produce an mdast syntax tree. The current view is therefore that `parse()` is sufficient and `parseForESLint()` is unnecessary. Is there a concrete use case that requires including it in the initial contract?

1. Does the Markdown parser contract need a `NonMdastParser` type analogous to ESLint's [`NonESTreeParser`](https://github.com/eslint/eslint/blob/v10.9.1/lib/types/index.d.ts#L1021)? ESLint's JavaScript parser types distinguish parsers that return ESTree from parsers that may return another AST format. This initial proposal intentionally supports only parsers that return a compatible mdast `Root`, because `MarkdownSourceCode`, traversal, and the bundled rules are built around the mdast contract. The current view is therefore that `NonMdastParser` should not be included. Is there a concrete use case for a non-mdast parser that should be supported by this language rather than provided through a separate ESLint language?

## Help Needed

N/A

## Frequently Asked Questions

### Is this RFC specific to Rust?

No. Rust parser performance motivated the proposal, but any synchronous parser that returns compatible mdast can implement the interface.

### Why does `@eslint-markdown/parser` use `satteri` instead of `markdown-rs`?

The initial plan was to use [`markdown-rs`](https://github.com/wooorm/markdown-rs) and transfer its Rust AST to JavaScript by serializing the complete tree as JSON and then calling `JSON.parse()`. Further experimentation showed that this serialization and deserialization added significant overhead.

[`satteri`](https://github.com/bruits/satteri) instead stores the parsed tree in a Rust arena, transfers it through a compact binary buffer, and materializes the mdast nodes in JavaScript using a binary reader. This avoids the full-tree JSON serialization and deserialization used by the earlier `markdown-rs` integration design and was substantially faster in the prototype. For this reason, the implementation of `@eslint-markdown/parser` was changed to use `satteri`.

### How can the synchronous Markdown parsing API call into Rust?

`@eslint-markdown/parser` loads `satteri` through its Node-API binding and invokes it directly on the Node.js main thread. The call remains synchronous from `MarkdownLanguage#parse()` through the parser wrapper and native binding.

### Does the integration invoke Rust through `execSync()`?

No. It neither spawns a Rust executable nor creates a child process for each parse. Node.js loads the compiled `.node` addon as a dynamic library, and JavaScript calls its exported function directly.

### Did `markdown-rs` provide a synchronous Node.js API directly?

No. `markdown-rs` is a Rust crate rather than a ready-made Node.js package, so the earlier design required a Node-API binding layer. The current prototype instead uses `satteri`, which already provides Node-API and WebAssembly bindings, behind the ESLint-compatible `@eslint-markdown/parser` wrapper.

### Why add `languageOptions.parser` instead of replacing the default parser?

Replacing `mdast-util-from-markdown` may be appropriate later if the Rust-based parser becomes sufficiently compatible and mature, but it is not part of this RFC. Making it the default now would add platform-specific native binaries to the default Node.js dependency graph and a WebAssembly artifact and initialization path for browser consumers. A parser option keeps the current installation and portability characteristics for existing users while allowing the experimental parser to be evaluated through explicit opt-in.

### Why add a parser option when ESLint languages supersede the older parser abstraction?

The two abstractions operate at different boundaries in this proposal. `markdown/commonmark` and `markdown/gfm` remain the ESLint languages and continue to define the `MarkdownSourceCode` and rule contract. `languageOptions.parser` selects only the implementation that produces the compatible mdast `Root` for one of those languages. A parser that exposes a different AST contract should be implemented as a separate ESLint language instead.

### Why not add a `useNativeRustParser` language option instead?

A dedicated boolean would couple `@eslint/markdown` to the experimental `@eslint-markdown/parser` package. A regular dependency would add the Rust parser and its platform-specific artifacts to every user's dependency tree, even when unused. Making it an optional peer would still require conditional loading, and asynchronous dynamic imports do not fit the current synchronous parsing path. Accepting a parser object instead keeps the dependency explicitly opt-in and supports future mdast-compatible parsers through the same API.

### Can a parser return a non-mdast AST?

The public `MarkdownParser` type requires an mdast `Root`, but the runtime does not recursively validate the AST. A JavaScript parser may therefore return other node shapes at runtime, much like a custom JavaScript parser can. However, that is outside the supported contract, and `MarkdownSourceCode` and the bundled rules are designed for mdast.

### Can a custom parser be asynchronous?

No. The language parsing API is synchronous, and native Node.js addons can expose synchronous functions without spawning a child process.

### Are existing rules guaranteed to work with every custom parser?

No. Matching node types and properties does not guarantee identical parsing semantics. For example, parsers may disagree about whether a list marker without following whitespace starts a list or a paragraph, so a rule may observe a different node even though both parsers produce valid mdast. Existing rules should work when a parser follows the documented mdast shapes, positions, and Markdown semantics, but parser authors must document relevant syntax and edge-case differences.

### Can a rule use `meta.languages` to require a particular parser?

No. `meta.languages` describes compatibility with the selected ESLint language, not with a particular parser implementation, so it cannot express parser-specific compatibility. Parser and rule authors must document any such requirements or edge-case differences. A parser that exposes a different AST contract should be provided through a separate ESLint language rather than through `languageOptions.parser`.

## Related Discussions

- [eslint/markdown#703: Support `languageOptions.parser` to improve parser performance by integrating a Rust parser](https://github.com/eslint/markdown/issues/703)
- [eslint/markdown#706: Experimental implementation of `languageOptions.parser`](https://github.com/eslint/markdown/pull/706)
- [lumirlumir/npm-eslint-markdown#638: Rust-based `@eslint-markdown/parser` prototype](https://github.com/lumirlumir/npm-eslint-markdown/pull/638)
- [Original ESLint Discord discussion](https://discord.com/channels/688543509199716507/688770853588172860/1519556220917252146)
