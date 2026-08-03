- Repo: eslint/markdown
- Start Date: 2026-08-04
- RFC PR: (leave this empty, to be filled in later)
- Authors: lumirlumir

# Support `languageOptions.parser` for Markdown

## Summary

This RFC proposes adding `languageOptions.parser` to `@eslint/markdown` so users can replace the built-in [`mdast-util-from-markdown`](https://github.com/syntax-tree/mdast-util-from-markdown) parser with another synchronous parser. It also proposes `languageOptions.parserOptions` for parser-specific configuration. The immediate motivation is to integrate an optional Rust-based parser, but the API is not Rust-specific and can be used by other Markdown parsers. The built-in JavaScript parser remains the default, so existing configurations and browser usage are unchanged.

## Motivation

Markdown parsing can dominate the time spent linting Markdown. In the prototypes described in [eslint/markdown#703](https://github.com/eslint/markdown/issues/703) and [eslint/markdown#706](https://github.com/eslint/markdown/pull/706), parsing took approximately **75 ms** of a **100 ms** run. Replacing it with `@eslint-markdown/parser`, a Rust-based parser built on [`satteri`](https://github.com/bruits/satteri), reduced parsing time to approximately **2 ms** and the total runtime to approximately **27 ms**, making the overall run approximately **3.5 times faster**. Although these results are preliminary, they demonstrate that selecting a faster parser can materially improve linting performance.

`@eslint/markdown` currently uses `mdast-util-from-markdown` from the mdast ecosystem. [`satteri`](https://github.com/bruits/satteri) is a high-performance Markdown and MDX processor that parses Markdown in Rust and exposes the result to JavaScript as mdast. Its [`markdownToMdast()`](https://satteri.bruits.org/docs/entry-points/#trees-without-compiling) API supports CommonMark and GFM as well as frontmatter and math parsing. Initial experiments indicate that it can support most of the cases already implemented by `@eslint/markdown`, although broader compatibility testing is still required.

Changing the default parser to a native parser is not currently suitable. Native Node.js addons require binaries for each supported operating system and architecture, while browser environments require a separate WebAssembly build and loading code. Adding those artifacts to the default installation would increase distribution and maintenance costs for every user, including users who do not need the performance improvement.

A parser option solves this by keeping the current portable parser as the default while allowing users to opt into another implementation. It also allows users to provide their own mdast-related parser with the syntax plugins they need. This proposal adds the option specifically to the Markdown language implementation. It does not propose a new parser mechanism for every ESLint language plugins such as JSON and CSS.

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

`MarkdownLanguageOptions` will gain two properties:

```ts
export interface MarkdownLanguageOptions extends LanguageOptions {
    frontmatter?: false | "yaml" | "toml" | "json";
    math?: boolean;
    parser?: MarkdownParser;
    parserOptions?: Record<string, unknown>;
}
```

`parser` is an object with a synchronous `parse()` method. `parserOptions` contains additional options for that method and is used only when `parser` is also specified.

The parser contract is:

```ts
import type { Root } from "mdast";
import type { ObjectMetaProperties } from "@eslint/core";

export type NonMdastParser = ObjectMetaProperties & {
	parse(text: string, options?: any): unknown;
};

export type MdastParser = ObjectMetaProperties & {
	parse(text: string, options?: any): Root;
};

export type MarkdownParser = NonMdastParser | MdastParser;
```

Parser metadata is optional and informational. `@eslint/markdown` does not change parsing behavior based on `meta`.

The parser is an object rather than a bare function to follow the shape used by ESLint parser packages and to leave room for metadata without changing the invocation API. Like `fromMarkdown()`, the method receives the Markdown text as its first argument and parser options as its second argument.

This RFC proposes only `parse()` for the initial implementation. Whether Markdown parsers need a `parseForESLint()`-style method or a way to return additional parser services is left for a future proposal after a concrete use case is identified. `MarkdownLanguage` already owns creation of `MarkdownSourceCode`, so the Rust prototype only needs to produce the syntax tree.

### Parser Invocation

When `languageOptions.parser` is present, `MarkdownLanguage#parse()` calls its `parse()` method with the Markdown source text and a newly created options object:

```js
const root = parser.parse(text, {
    mode: this.#mode,
    frontmatter: context?.languageOptions?.frontmatter,
    math: context?.languageOptions?.math,
    ...context?.languageOptions?.parserOptions,
});
```

ESLint merges `defaultLanguageOptions` before parsing, so `frontmatter` and `math` have their effective default value of `false`. `mode` comes from the configured language: `markdown/commonmark` supplies `"commonmark"`, and `markdown/gfm` supplies `"gfm"`.

The values in `parserOptions` take precedence over the corresponding values derived from `languageOptions` when invoking the parser. Although `mode`, `frontmatter`, and `math` should normally be configured through the standard language and language options, a user can provide a different value specifically to the parser by setting the same property in `parserOptions`. This matches ESLint's existing parser-option precedence, as clarified in [eslint/eslint#20926](https://github.com/eslint/eslint/pull/20926).

When `languageOptions.parser` is absent, `MarkdownLanguage#parse()` continues to call `mdast-util-from-markdown` with the existing micromark and mdast extensions. `parserOptions` is not a new configuration surface for the built-in parser and has no effect unless `parser` is also configured. Users who want to customize `mdast-util-from-markdown` can expose a small parser object that calls it with their chosen extensions.

The parser call remains synchronous. Returning a promise is not supported.

### Rust Parser Integration

The proposed `@eslint-markdown/parser` package will use `satteri`, which exposes its Rust implementation to JavaScript through Node-API. The intended data flow is:

1. JavaScript passes the Markdown source string to `satteri` through its Node-API binding.
2. `satteri` parses the source and stores the resulting mdast nodes in a Rust-managed memory pool called an arena.
3. The Rust arena is encoded into a compact binary buffer and passed to JavaScript.
4. JavaScript reads the buffer and materializes the mdast nodes without serializing and deserializing the complete tree as JSON.
5. The materialized [mdast `Root`](https://github.com/syntax-tree/mdast#root) is returned to `@eslint-markdown/parser`.

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

As with custom JavaScript parsers, the runtime API does not attempt to prove that the returned AST matches the expected format. A parser may technically return different node types, and custom rules may be able to consume them, but that usage is outside the compatibility guarantee for the bundled rules. Whether the public TypeScript type should explicitly model non-mdast parsers remains an open question.

This has a limitation for rule `meta.languages`: a rule can declare support for `markdown/commonmark` or `markdown/gfm`, but it cannot further constrain support to a particular parser or AST dialect. Parser and rule authors must document any such compatibility requirements. (TODO: Think about it further.)

### Error Handling

The existing error behavior remains in place. If a custom parser throws an error, `MarkdownLanguage#parse()` catches it and returns an unsuccessful `ParseResult` instead of rethrowing it.

### Validation

`MarkdownLanguage#validateLanguageOptions()` will add the following checks:

- `parser`, when present, must be a non-null object with a callable `parse` property.
- `parserOptions`, when present, must be a non-null, non-array object.

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
            parserOptions: {
                implementationOption: true,
            },
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

1. Exporting `NonMdastParser`, `MdastParser`, and `MarkdownParser` types and adding `parser` and `parserOptions` to `MarkdownLanguageOptions` in `src/types.ts`.
2. Extending `MarkdownLanguage#validateLanguageOptions()` with the validation described above.
3. Selecting and invoking the custom parser in `MarkdownLanguage#parse()` while leaving the current default-parser path unchanged.
4. Adding unit tests with a small parser object for invocation, option precedence, invalid configuration, thrown errors, and both Markdown modes.
5. Adding `@eslint-markdown/parser` as a development dependency and running integration tests against both the default JavaScript parser and the Rust-based parser. These tests will cover CommonMark, GFM, frontmatter, math, source locations, and the bundled rules.
6. Adding type tests for the public parser interfaces and configuration options.

`@eslint-markdown/parser` remains an optional runtime dependency and is developed and released separately. It is included in `@eslint/markdown` only as a development dependency for integration and compatibility testing.

## Documentation

The `@eslint/markdown` README will document:

- the `parser` and `parserOptions` language options.
- the required synchronous `parse(text, options)` contract.
- the mdast and positional-information requirements.
- an example using an external parser package.
- an example wrapping `mdast-util-from-markdown` with custom extensions.
- the fact that parser behavior, platform support, and rule compatibility are the parser author's responsibility.

A blog post is not required for the API change. A future optimized parser may be announced separately when it is stable and has published compatibility and benchmark results.

## Drawbacks

Custom parsers may handle Markdown edge cases differently, resulting in different AST nodes, source locations, or fixes. They may also introduce installation and platform-specific issues, especially when using native binaries.

## Backwards Compatibility Analysis

This proposal is additive. Configurations that do not specify `languageOptions.parser` continue to use `mdast-util-from-markdown` with the same GFM, frontmatter, and math extensions as today. The default dependency graph and browser behavior do not change.

Only configurations using the new options are affected by the new validation and custom parser behavior. A custom parser that returns an incompatible tree may produce a parse error or rule failures, but no existing configuration can encounter that behavior before opting into the feature.

## Alternatives

### Replace the Default Parser

`@eslint/markdown` could replace `mdast-util-from-markdown` with a Rust-based parser if their output is sufficiently compatible. This would give all Node.js users the performance improvement without a configuration change. It was not selected because native binaries vary by operating system and architecture, browsers require a separate WebAssembly path, and the alternative parser is still experimental. Keeping the JavaScript parser as the default preserves the current portability and installation behavior.

### Accept Bare Parser Functions

`languageOptions.parser` could accept a function with the same signature as `fromMarkdown()`. A parser object was selected because it matches the package shape familiar from ESLint's JavaScript parsers, supports optional metadata, and can grow additional parser-level capabilities in a backwards-compatible way if a future RFC requires them.

### Require and Validate mdast Trees

`@eslint/markdown` could require every custom parser to return mdast and recursively validate the tree before running rules. This would provide a stronger compatibility boundary, but the validation would add runtime overhead and still could not detect every semantic difference between Markdown parsers. The proposed API documents mdast as the compatibility target without performing recursive runtime validation.

## Open Questions

1. `parseForESLint()`-style API? TODO
1. `NonMdastParser` type? TODO

## Help Needed

N/A

## Frequently Asked Questions

### Is this RFC specific to Rust?

No. Rust parser performance motivated the proposal, but any synchronous parser that returns compatible mdast can implement the interface.

### Why does `@eslint-markdown/parser` use `satteri` instead of `markdown-rs`?

The initial plan was to use [`markdown-rs`](https://github.com/wooorm/markdown-rs) and transfer its Rust AST to JavaScript by serializing the complete tree as JSON and then calling `JSON.parse()`. Further experimentation showed that this serialization and deserialization added significant overhead.

[`satteri`](https://github.com/bruits/satteri) instead stores the parsed tree in a Rust arena, transfers it through a compact binary buffer, and materializes the mdast nodes in JavaScript using a binary reader. This avoids the full-tree JSON serialization and deserialization used by the earlier `markdown-rs` integration design and was substantially faster in the prototype. For this reason, the implementation of `@eslint-markdown/parser` was changed to use `satteri`.

### Will the Rust parser become the default?

Not as part of this RFC. `mdast-util-from-markdown` remains the default. Replacing it would require separate compatibility, distribution, browser, and maintenance analysis.

### Can a parser return a non-mdast AST?

The runtime does not recursively validate the AST. Custom rules may be able to consume other node shapes, much like rules written for a custom JavaScript parser. However, `MarkdownSourceCode` and the bundled rules are designed for mdast, so non-mdast parsers are not guaranteed to work. The exact public type for this case is an open question.

### Can a custom parser be asynchronous?

No. The language parsing API is synchronous, and native Node.js addons can expose synchronous functions without spawning a child process.

### Are existing rules guaranteed to work with every custom parser?

No. They should work when the parser produces the mdast node shapes and positions described by the compatibility contract. Parser authors and users are responsible for differences in syntax support or edge-case behavior.

### Does `parserOptions` configure the built-in parser?

No. It configures the selected custom parser. To customize `mdast-util-from-markdown`, users can provide a parser object that wraps `fromMarkdown()` with the desired extensions.

## Related Discussions

- [eslint/markdown#703: Support `languageOptions.parser` to improve parser performance by integrating a Rust parser](https://github.com/eslint/markdown/issues/703)
- [eslint/markdown#706: Experimental implementation of `languageOptions.parser`](https://github.com/eslint/markdown/pull/706)
- [eslint/eslint#20926: Clarify precedence of `parserOptions` over `languageOptions`](https://github.com/eslint/eslint/pull/20926)
- [lumirlumir/npm-eslint-markdown#638: Rust-based `@eslint-markdown/parser` prototype](https://github.com/lumirlumir/npm-eslint-markdown/pull/638)
- [Original ESLint Discord discussion](https://discord.com/channels/688543509199716507/688770853588172860/1519556220917252146)
