- Repo: eslint/eslint
- Start Date: 2026-08-21
- RFC PR: (leave this empty, to be filled in later)
- Authors: Nicholas C. Zakas

# Updated JavaScript/TypeScript parser

## Summary

This RFC proposes replacing ESLint's JavaScript analysis stack (`espree`, `eslint-scope`, `eslint-visitor-keys`, and code path analysis) with a new first-party toolkit, `@eslint/jskit`, that understands JavaScript, TypeScript, and JSX equally well and has no dependency on the `typescript` package. The toolkit is written in TypeScript and ships with an optional native core written in Rust, `@eslint/jskit-native`, which produces byte-identical output roughly 2.5× faster on the work it covers; when it isn't installed or isn't built for a platform, the TypeScript implementation runs instead and produces the same results. The work ships in two phases: first as a standalone parser that anyone can drop into `languageOptions.parser` to try out, and later as a new language plugin (`@eslint/jsnext`) that brings TypeScript-aware rules, TypeScript-specific rules, and a new control flow analysis. Typed linting is out of scope for this proposal.

## Motivation

### The problem we've been circling for two years

In August 2024, I opened [Rethinking TypeScript support in ESLint](https://github.com/eslint/eslint/discussions/18830) to describe a problem I kept hearing about from users, plugin developers, and commercial integrators: linting TypeScript with ESLint works, but almost nobody describes it as a good experience. The problems I listed then are the same ones we have today:

* **Requiring a separate plugin.** TypeScript is the majority dialect in the ecosystem, and yet ESLint out of the box can't parse it. Users have to find typescript-eslint, understand it, and configure it before they can lint the code they actually write.
* **Lack of cacheability with type-aware linting.** With type-aware linting enabled, the parser works across the whole project, which defeats ESLint's file-level cache. It's not always obvious to users when they have opted-in to type-aware linting by enabling a rule that uses it, and then are surprised when caching doesn't work.
* **Requiring `tsc`.** Integrators who embed the ESLint API in a product, rather than shipping the CLI, don't want to also embed and version the `typescript` package.
* **Too much parser responsibility.** typescript-eslint hooks in through `parseForESLint()`, which predates language plugins. That makes parsing, scope analysis, and type-service setup a single black box to the core, so we can't reason about or measure what's actually happening during a lint run.

None of this is a criticism of typescript-eslint or the people who maintain it. They built dependable TypeScript support inside the constraints the ESLint core gave them, and they did it well. The reason those constraints exist is that the core was slow to treat TypeScript as anything other than someone else's problem.

Two things have changed since that discussion.

First, we started treating TypeScript syntax as a core concern. [eslint/eslint#19173](https://github.com/eslint/eslint/issues/19173) has been steadily working through the core rules that behave incorrectly on TypeScript syntax, and 22 core rules now declare `meta.docs.dialects: ["JavaScript", "TypeScript"]`. [RFC 135](https://github.com/eslint/rfcs/blob/main/designs/2025-rule-languages/README.md) added the metadata that lets a rule say which languages it supports at all. We have committed, in practice, to core rules that work on TypeScript.

Second, the toolkit exists. The strawman I described in the 2024 discussion — write our own parser, teach scope analysis about TypeScript, put the visitor keys next to both — has been prototyped and is passing differential conformance against the implementations it replaces. It has since grown a second, native implementation of the same analyses in Rust, which is described below.

### What the 2024 discussion concluded

The discussion did not produce consensus, and it's worth being precise about where the disagreement was, because this RFC has to answer it.

The ESLint team was broadly supportive of treating TypeScript syntax as a first-class core concern. The typescript-eslint maintainers were supportive of that too, and specifically supportive of deduplicating the extension rules, but were opposed to a new parser. Their objections were:

1. **A new parser cuts you off from type-aware linting**, because TypeScript's type APIs only accept nodes from a `ts.SourceFile` they produced.
2. **Maintenance burden.** TypeScript ships every three months, and each release may add syntax the parser must learn.
3. **AST compatibility risk.** The ecosystem is written against the typescript-eslint AST. A second implementation that differs anywhere breaks rules, and there is no ESTree-style specification body to keep the two honest. There is an [AST specification](https://github.com/typescript-eslint/typescript-eslint/tree/main/packages/ast-spec) that typescript-eslint itself maintains.
4. **Existing parsers already exist** (swc, oxc, Babel, hermes-parser), so the effort is redundant.

There was also a disagreement about whether depending on `typescript` is a risk worth caring about. My position hasn't changed: ESLint needs control over its core dependencies. We learned that lesson when `esprima` went unmaintained, which is why `espree` exists, and why we deliberately built on Acorn as a pluggable base we could replace. Being unable to fix or route around a core dependency is a risk regardless of who maintains it or how well.

That risk stopped being hypothetical while this RFC was being written. TypeScript 7.0 has shipped and doesn't have a JS API to hook into, meaning that `@typescript-eslint/parser` cannot yet use it directly (the recommended arrangement is to keep the TypeScript 6.0 API installed alongside it and point the parser at that). The new toolkit's benchmark carries a row for TypeScript 7 that currently reports itself skipped for exactly this reason. This is not a criticism of typescript-eslint; keeping up with a compiler release of that size is genuinely hard work, and they are doing it. It is a demonstration of the coupling: when the dependency moves, everything downstream of it waits, and neither we nor our users have any way to move first.

I think objections 2, 3, and 4 are answerable, and this RFC answers them with the design below rather than with argument: the maintenance burden is real but bounded, AST compatibility is enforced by differential testing against `@typescript-eslint/parser` on every file we can find rather than asserted, and the existing parsers were evaluated and don't produce the AST or the analyses we need. Objection 1 is correct, we lose type-aware linting, but I don't think that is necessarily a terminal state. Rather, it's something we can build towards on top of this new tooling foundation.

### What we get that isn't about TypeScript

Two problems that have nothing to do with TypeScript get fixed by the same work:

* **`eslint-scope` doesn't see TypeScript.** It walks past type annotations, so a type-only import looks unused and a type reference looks undefined. Every rule that consults scope is wrong on TypeScript files today unless something replaces the scope analyzer.
* **Code path analysis is unreliable.** ESLint's code path analysis has never been trustworthy enough to build on, and its API (segments, `currentSegments`, `childCodePaths`) asks rules to hand-maintain state that the analysis should be answering directly. Fifteen core rules use it. It has been effectively frozen for years because changing it safely is very hard.

## Detailed Design

### JSKit

[`@eslint/jskit`](https://github.com/eslint/jsnext) is a single package containing three analyses that share one representation of a program:

| Analysis  | Entry points                       | Produces                                                              |
| --------- | ---------------------------------- | --------------------------------------------------------------------- |
| **Parse** | `parse()`, `validate()`, `toAST()` | A binary AST, a binary token stream, and a line-start table            |
| **Scope** | `analyze()`, `analyzeTree()`       | A binary scope graph: scopes, symbols, references, definitions         |
| **Flow**  | `createGraph()`                    | A binary basic-block control flow graph for every execution unit       |

```js
import { parse, toAST, analyze, createGraph } from "@eslint/jskit";

const result = parse(`const answer: number = 42; answer;`);

// Scope analysis reads the binary buffer directly.
const scope = analyze(result, { sourceType: "module", dialect: "ts" });

// So does control flow analysis, over both buffers.
const flow = createGraph(result, scope);

// An ESTree AST is built only if something asks for one.
const { ast, errors } = toAST(result, { sourceType: "module", dialect: "ts" });
```

The reason the three are one package rather than three is the buffer between them. `parse()` returns an `ArrayBuffer` rather than an object tree, and both `analyze()` and `createGraph()` run their entire analysis against those buffers without ever materializing a node. Splitting them into packages would mean either publishing the binary format as a public contract between packages or giving up the representation entirely.

#### Parsing is three phases, not one

Most parsers do one pass and hand back a tree. This one splits that into three:

| Phase    | Entry point                 | Produces                       | Fails how           |
| -------- | --------------------------- | ------------------------------ | ------------------- |
| Parse    | `parse(code)`               | Binary buffers                 | Throws `ParseError` |
| Validate | `validate(result, options)` | An array of problems           | Never throws        |
| Decode   | `toAST(result, options)`    | ESTree objects   | Never throws        |

The dividing line is whether the answer depends on context the text alone doesn't supply. `parse()` accepts the union of everything JavaScript and TypeScript allow, and throws only when the text can't be tokenized or shaped into a tree. Everything that is merely *not allowed here* such as `with` in strict mode, a redeclared binding, `return` outside a function, TypeScript syntax in a `.js` file, JSX in a file that isn't JSX, top-level `await` in a script, all parses cleanly and is reported by `validate()`.

That's why `sourceType`, `dialect`, and `jsx` are options of phase 2 rather than phase 1, and it means one parse can be validated several ways (or skipped entirely if, for example, we just want to know if a file exports a particular binding). It also means the most expensive phase, allocating JavaScript objects, is optional. A tool that only needs to inspect part of a file reads the buffer and never pays for the rest.

There are no version options. The latest JavaScript, TypeScript, and JSX syntax is accepted, always.
#### Scope analysis understands types

`analyze()` produces the scope graph as a third binary buffer, addressing AST nodes by their offsets in the parse buffer. It reproduces `eslint-scope` for JavaScript and `@typescript-eslint/scope-manager` for TypeScript as one walk, checked against both. Where the two references disagree about ground both cover, `eslint-scope` wins; where only the TypeScript analyzer has an opinion, it wins. Three disagreements survive as options rather than decisions, because both answers are defensible: `jsxPragma`, `jsxFragmentName`, and `globals`.

There are two entry points over one walk. `analyze()` reads the binary buffers. `analyzeTree()` reads an ordinary ESTree tree from any ESLint-compatible parser, which is the compatibility path and lets scope analysis be replaced without replacing the parser:

```js
import * as espree from "espree";
import { analyzeTree } from "@eslint/jskit";

const tree = espree.parse(code, { ecmaVersion: "latest", range: true });
const scopes = analyzeTree(tree, { dialect: "js" });
```

Three consumers read the resulting buffer: `toScopeManager()` for the escope-compatible object graph that rules use today, `Scopes` for point queries straight off the buffer, and `toScopeTree()` for a serializable debugging view.

The design was driven by a survey of all 293 core rules in `eslint@10.8.1`, 88 of which touch scope. Half of all scope use is one question, "does this identifier still refer to the global `Symbol`/`RegExp`/`console`?" We can get this answer without creating the entire scope tree, which is the direction I'd like to head in the future.
#### Control flow analysis is new work

`createGraph()` builds a basic-block control flow graph for every execution unit in a program: the program itself, each function, each class field initializer, each static block. Blocks record the variable writes they perform, in execution order and tied to the scope analysis; edges record the branch condition that decided them.

Those two facts are the forward-looking half of the design. Branch conditions on edges refine a type environment and writes invalidate one, which is exactly what a future narrowing pass needs and what ESLint's code path analysis never recorded.

The format was designed against the same kind of survey: every core rule that consumes code path analysis today. The fifteen consumers divide into four jobs, and the format answers each one directly rather than asking rules to maintain state:

| What rules actually do                            | How often     |
| ------------------------------------------------- | ------------- |
| Ask "is this point reachable?"                    | 7 of 15 rules |
| Use code paths as a correct function stack        | 4 of 15       |
| Split "on every path" from "on some path" at exit | 2 of 15       |
| Carry variable state along edges                  | 2 of 15       |

`thrownSegments`, `finalSegments`, `initialSegment`, and `childCodePaths` have zero consumers in core.

This is the one analysis with no reference implementation to diff against, so its integration tests are its contract. TypeScript's own control flow graph is used as a correctness comparison.

#### Conformance is the correctness argument

The compatibility objection from the 2024 discussion is the one that matters most, and the answer is not "we'll be careful." Every JavaScript and TypeScript file in `node_modules` is run through the parser and the scope analyzer and compared against the implementation it replaces:

```text
files=… ok=… mismatch=0 threw=0   # AST vs espree
ok=… bad=0                        # tokens and comments vs espree
files=… ok=… mismatch=0 threw=0   # AST vs @typescript-eslint/parser

binary files=… ok=… mismatch=0 threw=0   # scopes vs eslint-scope
tree   files=… ok=… mismatch=0 threw=0
binary files=… ok=… mismatch=0 threw=0   # scopes vs @typescript-eslint/scope-manager
tree   files=… ok=… mismatch=0 threw=0
```

Zero mismatches is the standard; anything else is a regression. In addition, test262 checks what the parser rejects in JavaScript (no valid program rejected, no invalid one accepted), TypeScript's own conformance suite is run for both dialects, and `espree`'s test suite is run directly.

Every intentional difference is written down in [`docs/deviations.md`](https://github.com/eslint/jsnext/blob/main/docs/deviations.md), and a difference that isn't on that list is a bug. The largest entry: where `@typescript-eslint/parser` omits a property or leaves it `undefined`, this parser sets it to `null`, so a node of a given type has the same shape every time. This is to ensure we always have the same shape for nodes and to better align with how ESTree represents missing details.

The known gaps are also written down. The notable one is that TypeScript's parser recovers from syntax errors and produces a tree while this one throws. Matching that recovery is not a goal.

#### The core has a second implementation, in Rust

The buffer that lets the three analyses share one representation of a program turns out to be the thing that lets the toolkit have two implementations of itself.

[`@eslint/jskit-native`](https://github.com/eslint/jsnext/tree/main/packages/jskit-native) is `parse()`, `validate()`, `analyze()`, and `createGraph()` rewritten in Rust behind Node-API bindings. The three producers write **the same binary formats, byte for byte**, and `validate()` reports the same problems in the same order with the same messages. Everything downstream of that output, `toAST()`, `Scopes`, `toScopeManager()`, `FlowBufferReader`, the ESLint parser object, is the one TypeScript implementation reading output it cannot tell apart from its own.

The Rust sources mirror the TypeScript sources file by file, `tokenizer.rs` beside `tokenizer.ts` and `validator.rs` beside `validate.ts`, so the two can be read side by side.

**Two things are deliberately not native, for the same reason.**

* `toAST()` builds a JavaScript object per node, and building millions of them through Node-API calls costs far more than building them in JavaScript from a buffer that's already there. Its speed comes instead from a generated decoder: a build step turns a declarative schema into one function per node kind, each assembling its whole ESTree node as a single object literal, so every node of a kind gets the same hidden class. This is the same trade `oxc-parser` makes in its deserializers.
* `analyzeTree()` reads the caller's own ESTree objects, and crossing the boundary per node would cost more than the walk saves. It stays the TypeScript compatibility path.

**Nothing requires it.** `@eslint/jskit-native` is an `optionalDependency`. The Node entry point tries to load it and registers it if it's there; the neutral bundle, which is what a browser gets, never attempts to. When the package is missing, wasn't built for the platform, or `JSKIT_NATIVE=0` is in the environment, the TypeScript implementation runs: same buffers, same errors, same problems, just slower. No machine needs a Rust toolchain to install, use, or work on the toolkit, and the build script exits cleanly when `cargo` isn't on the path.

**Parity is enforced the same way conformance is: differentially, and byte-exactly.** Four harnesses run the corpus through both implementations and compare raw output:

```bash
node tools/diff-parse.mjs    ../../node_modules   # parse buffers
node tools/diff-validate.mjs ../../node_modules   # problem lists, both dialects
node tools/diff-analyze.mjs  ../../node_modules   # scope buffers
node tools/diff-graph.mjs    ../../node_modules   # flow buffers
```

`mismatch=0` is the standard, and both implementations must also agree on which files they reject. The full corpus — 21,000+ files — passes all four with zero mismatches, as do the JSX/TSX fixtures under every combination of options, which matters because `node_modules` contains no JSX. `diff-validate.mjs` runs against test262 as well, because that's the one corpus full of programs that are *supposed* to produce problems.

**The distribution story.** One package carries prebuilt binaries for linux x64 and arm64 (gnu), macOS x64 and arm64, and Windows x64. Each is built on a runner of its own platform, and the parity tests run against that binary on that machine before it's uploaded — a binary that was never exercised on its own platform doesn't get published. The two packages release together with linked version numbers and an exact pin, because the binary formats are one contract with two implementations and a mismatched pair is not a thing that should be installable. A platform with no binary falls back, and works.

I want to be straightforward about the cost here: this is a second implementation of the same code in a second language, and that is a real maintenance burden. See the drawbacks.

#### Performance

Numbers below are from the run recorded in [`benchmarks/parse/results.json`](https://github.com/eslint/jsnext/blob/main/packages/jskit/benchmarks/parse/results.json), on ~196 KiB generated modules, Node 24, Linux x64. Absolute figures move a lot with how warm the machine is; the ratios within a table are what to read.

Start with the job ESLint actually asks for — a tree plus tokens plus comments, with `range` and `loc` on every one of them — because that's the only tier where a number translates directly into a lint run. Operations per second:

| Parser                                     | JavaScript | TypeScript | JSX      |
| ------------------------------------------ | ---------- | ---------- | -------- |
| `eslintParser.parse()`, native             | **35.2**   | **32.0**   | **30.9** |
| `eslintParser.parse()`, TypeScript         | 22.4       | 22.3       | 21.4     |
| `meriyah`                                  | 22.0       | —          | 19.3     |
| `espree`                                   | 10.9       | —          | 9.4      |
| `@babel/eslint-parser`                     | 4.1        | 3.2        | 3.5      |
| `@typescript-eslint/parser` + TypeScript 5 | 2.6        | 2.0        | 2.5      |

That's roughly **3× `espree`** on JavaScript and JSX, and **12–16× `@typescript-eslint/parser`** across all three.

The native core's own contribution is larger than that table makes it look, because that tier includes work the native core deliberately doesn't do. On the buffer-producing steps, which is what was rewritten:

| Step                     | TypeScript     | Native            | Speedup            |
| ------------------------ | -------------- | ----------------- | ------------------ |
| `parse()`                | 73.7/69.8/77.3 | 194.3/164.2/179.4 | 2.6× / 2.4× / 2.3× |
| `parse()` + `validate()` | 47.3/48.8/51.6 | 126.3/112.8/131.0 | 2.7× / 2.3× / 2.5× |

Each cell is JavaScript / TypeScript / JSX, in operations per second.

End to end through `eslintParser`, the same binding is worth about **1.5×**, not 2.5×. That gap is the honest shape of the thing: `toAST()` and location building are TypeScript on both paths, they dominate that tier, and no parser avoids them while producing what ESLint expects. The Rust makes the fast half twice as fast; it can't do anything about the other half.

Scope analysis, measured separately by running `benchmarks/scope/benchmark.js` with and without the binding:

| Measurement                 | TypeScript | Native | Compared against                   |
| --------------------------- | ---------- | ------ | ---------------------------------- |
| `analyze()`, JavaScript     | 1.2×       | 3.4×   | `eslint-scope`                     |
| `analyze()`, TypeScript     | 2.7×       | 5.6×   | `@typescript-eslint/scope-manager` |
| parse + analyze, JavaScript | 3.5×       | 9.6×   | `espree` + `eslint-scope`          |
| parse + analyze, TypeScript | 18×        | 44×    | the `@typescript-eslint` pair      |

`oxc-parser` is the only other contender in the same class, and it's in the benchmark for that reason. `parse()` leads its best row in every suite (194 against 108 on JavaScript, 164 against 102 on TypeScript, 179 against 110 on JSX) but they're close, and they should be: both are Rust parsers that hand back a buffer instead of a tree. The gap between this toolkit and `oxc-parser` is not really about speed, which is why the alternatives section argues about the AST and the analyses instead.

Three caveats worth stating, because I'd rather set expectations correctly:

1. **Parsing is a small percentage of the time ESLint spends on a file.** Rules and traversal are the rest, and they cost the same on either tree. In some of my local testing this came out to around 15% but it's highly variable based on the ESLint configuration. There are, however, opportunities to rethink how we implement and execute rules to speed things up in the future.
2. **The comparison against `@typescript-eslint/parser` is against its non-type-aware mode**, which is already its fast path. As noted in the 2024 discussion, that mode is single-file and cacheable today.
3. **These ratios are not stable across machines.** This toolkit allocates far less than the implementations it's compared against, so a throttled machine slows it down proportionally more and *deflates* its ratios. Two runs of the same benchmark on the same laptop can disagree by a third.

To be clear: the goal isn't to beat `tsc`, it's to get consistent performance without depending on it.

### Phase 1: ship the parser

**Deliverable:** publish `@eslint/jskit` with `eslintParser` as a supported, opt-in parser, alongside `@eslint/jskit-native` as its optional native core. Nothing in ESLint core changes.

```js
// eslint.config.js
import { eslintParser } from "@eslint/jskit";

export default [
	{
		files: ["**/*.js", "**/*.ts", "**/*.tsx"],
		languageOptions: { parser: eslintParser },
	},
];
```

`eslintParser` implements `parseForESLint()` and returns three things:

- **`ast`** — the ESTree tree, with `range` and `loc` on every node, token, and comment.
- **`scopeManager`** — the scope graph over that tree, which ESLint takes in place of running `eslint-scope`. This is what makes scope analysis understand TypeScript.
- **`visitorKeys`** — needed for the same reason. Without them, nothing inside a TypeScript-only property is ever visited and rules see those nodes with no `parent`.

Configuration is deliberately minimal, because the file name already says most of what a parser needs to know:

- **Dialect comes from the extension.** `.js`, `.cjs`, `.mjs`, and `.jsx` are parsed as JavaScript, so TypeScript syntax in them is reported rather than quietly accepted. Everything else is parsed as TypeScript. Override with `parserOptions.dialect`.
- **JSX comes from the extension.** `.jsx` and `.tsx` accept JSX; everything else reports it. Override with `parserOptions.ecmaFeatures.jsx`, which is where `espree` reads it from, so an existing configuration keeps working.
- **Declaration files come from the extension.** `.d.ts`, `.d.mts`, and `.d.cts` are ambient, so a `const` in one needs no initializer. Override with `parserOptions.declaration`.
- **`sourceType`** comes from `languageOptions.sourceType`, which ESLint already resolves.

Because `parseForESLint()` predates language plugins, this phase inherits the "too much parser responsibility" problem I explained earlier. I think that's a fair tradeoff to start getting the performance imThat's accepted deliberately: `parseForESLint()` is the only hook that works with ESLint today, and Phase 1 is about getting the toolkit into people's hands, not about fixing the integration point. Phase 2 fixes the integration point.

Phase 1 allows us to start switching smaller projects (like `@eslint/config-array`) to TypeScript code to really exercise the new parser. We can do this package-by-package, checking our work as we go.

**What Phase 1 does not give us.** Core rules are still core rules. `no-undef` and `no-unused-vars` must be turned off for `**/*.ts`, exactly as typescript-eslint recommends, because `no-undef` reports every name from TypeScript's standard library (which isn't loaded) and `no-unused-vars` reports the parameters of declaration-only signatures (which have nowhere to be used). There are no TypeScript-specific rules and no typed linting.
### Phase 2: a new language plugin

**Deliverable:** a language plugin (working name `@eslint/jsnext` to differentiate from `@eslint/js`) that packages the toolkit as an ESLint language rather than as a parser.

```js
// eslint.config.js
import jsnext from "@eslint/jsnext";

export default [
	{
		files: ["**/*.js", "**/*.ts", "**/*.tsx"],
		plugins: { jsnext },
		language: "jsnext/js",
		extends: ["jsnext/recommended"],
	},
];
```

A language plugin, rather than a parser, is what lets the rest of the problems get fixed:

1. **The core stops guessing.** Under `parseForESLint()`, ESLint can't tell parsing from scope analysis from anything else the parser decided to do. As a language, each of those is a separate, measurable step with a defined interface.
2. **`SourceCode` is ours.** That's where the new control flow analysis is exposed. Rules get a real reachability query instead of hand-maintaining a segment set, and the fifteen core rules that use code path analysis today can be rewritten against something that isn't known-buggy.
3. **TypeScript-specific rules can exist.** Rules for TypeScript syntax, the ones core has never accepted because core doesn't parse TypeScript, belong here.
4. **Core rule behavior can be corrected per dialect.** The work already happening in [eslint/eslint#19173](https://github.com/eslint/eslint/issues/19173) has a home, and `meta.languages` from [RFC 135](https://github.com/eslint/rfcs/blob/main/designs/2025-rule-languages/README.md) is how a rule declares where it applies.

The deduplication question that [Josh Goldberg raised in the 2024 discussion](https://github.com/eslint/eslint/discussions/18830) is the one to get right here. If the syntax-only extension rules move into the core rules and the type-aware ones don't, we've moved the boundary rather than removed it, from "ESLint syntax vs. TypeScript syntax and types" to "ESLint syntax and TypeScript syntax vs. types." That's a real improvement, and it's also not the end state. The end state requires typed linting, which is Phase 3 and not in this RFC.

We will also use Phase 2 as a time to evaluate the APIs that we expose to rules related to scopes and control flow. A lot of the patterns we use in ESLint's core rules are very inefficient (i.e., `prefer-const` walking through the scope tree to figure out if something is writable). There are a lot of questions about scope we can answer during the analysis phase and have that data prepared and easily retrievable by the time rules are executed. Ideally, for scopes, we'd come up with new APIs that can be polyfilled in the current rules to make transitioning to the new toolkit seamless. The only real caveat is with code path analysis, which will need to go through a breaking change to get where we need to be. (Which I think is acceptable because of how infrequently it's used.)

### Out of scope: typed linting

This proposal does not include type-aware linting, and the parser it proposes cannot do type-aware linting in the way typescript-eslint does. TypeScript's type APIs require nodes from a `ts.SourceFile` that TypeScript itself produced, so any type-aware rule built on this parser would need a mapping layer, at which point most of the work `@typescript-eslint/parser` does today has to be done anyway.

I want to be careful about what that means, because "no typed linting" is easy to read as "no typed linting, ever":

- **It is a limitation of this phase, not a design goal.** The control flow analysis was deliberately built to record variable writes and branch conditions, which is what a narrowing pass needs. That was not an accident.
- **We will investigate it.** The likely shape is the pluggable "types provider" idea from the 2024 discussion, which would let TypeScript supply types to rules that want them without ESLint depending on TypeScript to lint. There are also questions worth answering about how far a purely syntactic narrowing pass gets, and where it stops being useful. Modern TypeScript codebases lean on conditional types and inference-heavy libraries, and a partially type-aware linter that is confidently wrong is worse than one that admits it doesn't know.
- **Until we have an answer, typescript-eslint remains the recommendation for type-aware linting.** That should be said plainly in the documentation rather than implied.

## Documentation

This needs a formal announcement on the ESLint blog, and it needs to be a careful one. The community reads "ESLint is building TypeScript support" as "ESLint is replacing typescript-eslint," and since that *is* the long-term goal (see the FAQ), the post has to be clear about what is true today, what isn't, and on what timeline.

The blog post for Phase 1 should cover:

* What `@eslint/jskit` is and what it isn't.
* That it is `0.x` and not covered by our semantic versioning policy.
* That typed linting is not available and typescript-eslint remains the recommendation for it.
* How to try it, including the `no-undef`/`no-unused-vars` caveat.
* That there is an optional native core, that it installs by default, and that `JSKIT_NATIVE=0` turns it off (which is the first thing to try when something looks wrong).
* How to report a mismatch, since conformance feedback is the point of shipping.

Documentation work:

* A new page under **Use ESLint** for linting TypeScript, replacing the current situation where our TypeScript guidance is mostly a pointer elsewhere.
* API documentation for `@eslint/jskit` (this exists in the repository and needs a home on eslint.org).
* Updates to [Configure a Parser](https://eslint.org/docs/latest/use/configure/parser) once Phase 1 ships, and [Configure a Language](https://eslint.org/docs/latest/use/configure/languages) once Phase 2 does.
* The [rules index](https://eslint.org/docs/latest/rules/) should show which rules are TypeScript-aware, which is what `meta.docs.dialects` was added for.

## Drawbacks

1. **We are taking on a parser.** TypeScript ships a release every three months and each one may add syntax. This is planned work forever, and as I said in the original discussion, it is not my favorite thing to do. The counter-argument is the one I gave then: planned work is a completely different thing from unplanned work, and we've already lived through the unplanned version once with `esprima`. And with the benefits of AI, the cost to keep up with things is a lot smaller than it was at the time of the discussion.

2. **Two implementations of the same AST can drift.** There is no ESTree-equivalent specification body for the TypeScript AST; there is typescript-eslint's `ast-spec`, and it is a de facto standard maintained by one project. Differential conformance catches drift after the fact, and does so on every file in `node_modules`, but it doesn't prevent it. The healthier version of this is a shared specification, which requires cooperation we don't currently have. In the meantime, we'll let typescript-eslint lead and we'll follow.

3. **The same analyses exist twice, in two languages.** Every change to `parse()`, `validate()`, `analyze()`, or `createGraph()` has to be made in TypeScript and again in Rust, or the binding falls behind and the differential runs fail. That is about 23,000 lines of Rust standing beside their TypeScript originals, and it asks a maintainer to be comfortable in both languages. The mitigation is that the mirroring is mechanical and that the parity harness fails loudly rather than quietly, but the burden is real and it is permanent.

4. **We are shipping a native binary.** Prebuilt binaries mean a platform matrix, a build-and-publish pipeline that has to work on five runners, and a class of installation failure ESLint has never had to think about. The design tries to make every one of those failures soft (the native package is optional, an unbuilt platform falls back, a missing binary falls back, and nothing but speed depends on it) but "soft failure" still means some users silently get the slow path and won't know it. It also means the toolkit's supply chain now includes a Rust dependency tree, small as it is.

5. **We will fragment TypeScript linting for a while.** During the transition there will be two ways to lint TypeScript with ESLint, with different capabilities, and users will have to understand the difference. This is a real cost and documentation only partly mitigates it.

6. **No typed linting means real rules can't be written.** A meaningful fraction of what people value in typescript-eslint requires types. Anyone who needs those rules gets nothing from Phase 1 or Phase 2 except a faster parse.

7. **We are replacing code path analysis with something that has no reference implementation.** Differential testing is what gives me confidence in the parser and the scope analyzer, and control flow analysis doesn't get to have it. Its integration tests and comparisons against TypeScript's own flow graph are the contract, which is a weaker guarantee.

8. **This will be read as a hostile act.** It isn't, and I'd rather say so directly than pretend the perception won't exist. The overlap with typescript-eslint's work is real, the answer to "is the goal to replace it" is yes, and the maintainers deserve to hear that from us rather than infer it.

9. **Error recovery.** TypeScript's parser continues after a syntax error and produces a tree; this one throws. That's the right trade for a linter today, but it forecloses editor-oriented use cases that a recovering parser enables.

## Backwards Compatibility Analysis

Nothing in this RFC changes ESLint's default behavior in either phase.

* **Phase 1** publishes a new package. `espree` and `eslint-scope` remain the defaults. Using `eslintParser` is opt-in through `languageOptions.parser`, exactly like any third-party parser today, and it can be adopted or dropped per config object.
* **Phase 2** publishes a new plugin providing a new language. The `js/js` language is untouched. Selecting `jsnext/js` is opt-in through `language`.
* **`@typescript-eslint/parser` keeps working.** It is a parser like any other, and nothing here changes the `parseForESLint()` contract.
* **The native core changes nothing observable.** It is an optional dependency of `@eslint/jskit`, not of ESLint, and it produces byte-identical buffers and identical diagnostics. A user on a platform we don't build for, or behind an install that skips optional dependencies, gets the same results more slowly.

Within the opt-in path, compatibility is the design constraint, not an afterthought:

* The AST matches `espree` for JavaScript and `@typescript-eslint/parser` for TypeScript, enforced by differential testing, with every intentional difference written down.
* The scope graph matches `eslint-scope` and `@typescript-eslint/scope-manager`, enforced the same way, and `toScopeManager()` is complete enough for `@eslint-community/eslint-utils` — `ReferenceTracker`, `findVariable`, `getStaticValue` — to work unmodified.
* `VISITOR_KEYS` matches `@typescript-eslint/visitor-keys` entry for entry, and its JavaScript entries are a superset of `eslint-visitor-keys`.

Behavior differences a user will actually notice when switching:

* **`ecmaVersion` is ignored.** Latest syntax, always. Configurations that pin an older `ecmaVersion` to reject newer syntax will stop rejecting it.
* **`no-undef` and `no-unused-vars` must be off for TypeScript files.** Same as with typescript-eslint, for the same reasons.
* **Some parse errors arrive from a different phase.** A malformed regular expression pattern, for instance, is a validation problem here rather than a tokenizer error. Through `eslintParser` this is invisible — ESLint has no notion of a non-fatal parse problem, so the first problem becomes a thrown `ParseError` on the right line, which is what its own parsers do.

Phase 2's `0.x` versioning is a deliberate statement that the language plugin's behavior may change between minor versions until it graduates. That graduation is a separate RFC.

## Alternatives

1. **Do nothing and keep recommending typescript-eslint.** This is what we've done for years, and it's why we're here. It leaves every problem in the motivation unaddressed, and it leaves ESLint unable to lint the majority dialect of its own ecosystem without a third-party package.

2. **Adopt `@typescript-eslint/parser` and `@typescript-eslint/scope-manager` as core dependencies.** [Brad Zacher proposed exactly this](https://github.com/eslint/eslint/discussions/18830) and it is the shortest path to TypeScript support. It also makes `typescript` a hard dependency of ESLint. I'm not willing to do that for the reasons in the motivation, and the objection is about control rather than about TypeScript or Microsoft. I'd say the same about any company-sponsored project we couldn't route around.

3. **Use an existing alternative parser (swc, oxc, Babel, hermes-parser).** All of these parse TypeScript and several were considered. Three things rule them out. Each produces a different AST, so the compatibility work with `espree` and `@typescript-eslint/parser` is the same either way. None supply the scope analysis or the control flow analysis, which is most of the value here. Adopting one solves the smallest part of the problem. And each is a large dependency we don't control, which is the objection in the motivation restated.

    `oxc-parser` deserves being singled out, because it is the closest thing to what this RFC proposes: a fast Rust parser behind Node-API bindings, with a raw-transfer mode that hands back a lazy view over a shared buffer rather than materializing nodes — the same idea as `parse()` returning a buffer. It is in our benchmark and it is genuinely fast. It also produces its own AST, reports syntax errors as data rather than the way ESLint expects, emits no token list, and stops at parsing. Note that "it's Rust" is no longer one of my objections; it can't be, since the answer to that objection in the 2024 discussion is now half of what this RFC ships.

4. **Build the parser as an Acorn plugin**, which is what I suggested in 2024. It's a smaller lift and keeps a shared base with `espree`. It also constrains the AST and the internal representation to what Acorn's architecture allows, which rules out the binary format the scope and flow analyses are built on and that format is where the performance and most of the design leverage come from.

5. **Ship the whole thing at once as a language plugin, skipping Phase 1.** Fewer packages, less confusion, one announcement. It also means the parser meets real code for the first time at the same moment the rules, the flow analysis, and the language integration do, and no feedback arrives until everything is finished. Phase 1 exists to get the parser wrong early and cheaply, and to let us dogfood on our own TypeScript.

6. **Ship the parser but skip the language plugin**, exposing everything through `parseForESLint()`. This works but it permanently inherits the "too much parser responsibility" problem, and it gives us nowhere to put TypeScript-specific rules or the control flow API.

## Open Questions

1. **The package name.** `@eslint/jsnext` describes the intent but is unlovely, and it becomes wrong the moment it ships. Alternatives are `@eslint/js-next`, `@eslint/ts`, or shipping the language inside `@eslint/jskit` and skipping the second package.

2. **What Phase 2 does about the duplicated rules.** Do the TypeScript-aware behaviors go into the existing core rules (with `meta.languages` gating), or do TypeScript-specific variants live in the plugin? The first is better for users and harder to do without breaking someone.

3. **What "supports TypeScript" means as a version policy.** We support ECMAScript features at stage 4. TypeScript has no equivalent gate. Do we track TypeScript releases, betas, or shipped-and-stable syntax only, and what do we say when we're behind?

4. **When, if ever, does this become the default?** Phase 2 says "a separate RFC." That's true, and it's also the question everyone will actually ask, so it may be worth stating a rough shape now.

5. **How do we work with typescript-eslint from here?** The 2024 discussion ended without a shared plan. A shared AST specification is the obvious place to cooperate. Whether that's wanted after this RFC is a question I can't answer alone.

6. **Error recovery.** Is a recovering parse mode worth building for editor integrations, or is throwing on the first error the permanent answer?

7. **Which platforms do we commit to, and what do we say about the rest?** The release builds linux x64/arm64 (gnu), macOS x64/arm64, and Windows x64. Missing from that list, at least: linux musl (Alpine, which a lot of CI runs on), Windows arm64, and FreeBSD. Falling back to the TypeScript implementation makes those work rather than fail, which is the right default, but a user on Alpine getting the slow path without being told is not great either. Do we add targets, surface the fallback somehow, or both?

8. **Is a native dependency acceptable in ESLint's core at all?** Phase 1 doesn't ask that question — `@eslint/jskit` is opt-in and the native package is optional inside it. Phase 2 doesn't either. Whatever RFC eventually proposes making this the default does, and it's worth knowing now whether the answer is going to be no.

## Help Needed

I've built the parser, the scope analyzer, and the control flow analysis, and I can ship Phase 1 without help.

Phase 2 needs the team:

* **Rules.** Rewriting the fifteen code path analysis consumers against the new flow API, and deciding rule-by-rule how TypeScript-aware behavior gets expressed. This is the bulk of the work and it isn't mine to do alone.
* **Language integration.** Wiring the toolkit into the `Language` and `SourceCode` interfaces, including whatever those interfaces turn out to be missing.
* **Documentation.** The TypeScript guidance on eslint.org needs to be written more or less from scratch.
* **Keeping up with TypeScript.** This is an ongoing commitment, not a task, and it should have more than one person who can do it.
* **Rust.** The native core needs more than one person who can maintain it, for the same reason. Today it is one implementation with one author, which is the least comfortable part of this proposal for me.

## Frequently Asked Questions

**Is the goal to replace typescript-eslint?**

Yes. I want TypeScript to be a first-party part of the ESLint ecosystem, working the same way everything else in ESLint works, rather than a thing you bolt on. That means ESLint parses TypeScript, ESLint's scope analysis understands TypeScript, and ESLint's rules behave correctly on TypeScript, without a separate plugin, a separate parser, and a separate set of duplicated rules.

This is a change from what I wrote in 2024, where I said the intent was not to replace typescript-eslint but to eliminate duplicated effort, with core providing a basic level and typescript-eslint providing a premium level. I no longer think that split is stable. It leaves TypeScript permanently second-class in ESLint, it leaves users permanently choosing between two overlapping implementations of the same rules, and "basic tier versus premium tier" is not a thing we do for any other language we support.

Two things that follow, and that I want to be explicit about:

* **This is a long road and typed linting is the hard part.** Until ESLint has an answer for type-aware rules, typescript-eslint does things we can't, and we should say so.
* **Saying this out loud is better than not saying it.** The typescript-eslint maintainers have been generous with their time and expertise, including in the discussion that led to this RFC. They deserve a direct answer about our intentions rather than having to infer one from our roadmap.

**Does this mean ESLint will depend on TypeScript?**

No. That's the point. `@eslint/jskit` has no dependency on the `typescript` package, and neither will the language plugin. If we later add typed linting through a types provider, TypeScript would be a peer dependency of that provider, not of ESLint.

**Will my existing typescript-eslint setup break?**

No. Nothing in this RFC changes any default. `@typescript-eslint/parser` remains a supported parser and typescript-eslint remains our recommendation for type-aware linting.

**Why not just contribute to typescript-eslint?**

For the syntax-aware core rules, we effectively are, [eslint/eslint#19173](https://github.com/eslint/eslint/issues/19173) is that work, done in coordination with them. The parser is different. The problem it solves is that ESLint's core analysis stack cannot see TypeScript at all, and that can't be fixed inside a plugin.

**Is this faster than typescript-eslint?**

Yes, substantially for TypeScript 6.x, on the parse and scope steps: about 16× on the job ESLint actually asks a parser for, and considerably more than that on parse plus scope analysis. But parsing is roughly 15% of a lint run, so a real project should expect a noticeable improvement rather than a dramatic one. And the comparison is against typescript-eslint's non-type-aware mode, which is already its fast path. See the performance section, including the caveats.

**Does this mean ESLint will depend on a native binary?**

Not in either phase of this RFC, and not in the way that question usually means. `@eslint/jskit-native` is an optional dependency of `@eslint/jskit`, which is itself opt-in. If it doesn't install, doesn't build, or doesn't exist for a platform, the TypeScript implementation runs and produces identical results. Nothing but speed depends on the binary being there. Whether ESLint's *default* stack should include one is a real question and it belongs to the RFC that proposes making this the default.

**Why write it twice instead of just writing it in Rust?**

Because the browser, and because bootstrapping. `@eslint/jskit-inspect` runs all three analyses in a browser tab, and a lot of what ESLint's ecosystem does with a parser happens somewhere a `.node` file can't be loaded. The TypeScript implementation is also what the Rust implementation is checked against — it came first, it's the one with the full test suite, and byte-for-byte differential parity against it is how I know the Rust is right. A Rust-only toolkit would have neither of those.

**What about drift between the TypeScript and Rust parsers?**

This is where AI makes the maintenance easier. We start with a conformance test then get AI to implement the changes in both parsers.

**Did you "vibe code" the new toolkit?**

I used AI to create the new toolkit, but I did not "vibe code" it. I started with a design and architecture that I wanted. I outlined the approach I wanted the package to follow, the primary external APIs, and where the seams were between different phases of logic. That approach included separating out the phases of parsing, validation, scope analysis, and control flow. I outlined the conformance tests against Espree and `@typescript-eslint/parser` and how to resolve conflicts between the two. The actual code was written by AI.

**Why not use oxc, since it's already a fast Rust parser?**

It's the closest existing thing and it's in the benchmark, so this is a fair question. It produces compatible ASTs but we'd still need the two analyses described in the RFC, so adopting it would leave nearly all of the work in place while adding a dependency we don't control. See the alternatives.

**What happens to `espree`?**

Nothing in this RFC. It remains the default parser and the JavaScript language keeps using it. If Phase 2 eventually graduates into `@eslint/js`, `espree`'s future is part of that separate RFC.

**Why ship something incomplete?**

Two reasons. 
The first is feedback. A parser is only as good as the code it has seen, and we need to get folks to try it out on code we've not seen. Shipping is how we find out what we got wrong while the cost of changing it is still low.

The second is dogfooding. Once the parser is published, we can start converting ESLint's own projects to TypeScript and lint them with it. That is the only way I know to be confident this works: use it on our own code, every day, before we ask anyone else to depend on it. It also gives us a real answer to a question we currently answer theoretically, which is what linting TypeScript with ESLint *should* feel like.

`@eslint/jskit` will be published as `0.x` for the duration of Phase 1, with `@eslint/jskit-native` tracking the same version number. Neither is covered by ESLint's semantic versioning policy until Phase 2.

Phase 1 is also where the native core gets the exposure it needs. Parity is checked against every file we can find, but "every file we can find" is not the same as everyone's code, and a second implementation is exactly the kind of thing that fails on the file nobody thought of. Installing the native package is the default, so most people will be running it; `JSKIT_NATIVE=0` turns it off, which makes "does this reproduce without the binary?" the first question we can ask about any report.

**Why a separate plugin rather than changing `@eslint/js`?**

Because it lets people opt in, and because it lets us break things. `@eslint/jsnext` is where the new stack proves itself with real users on real code, under `0.x` versioning, without any promise that today's behavior is next month's behavior. `js/js` keeps working, unchanged, the whole time.

The intent is that this is temporary. When the new stack is proven, it becomes the JavaScript language in `@eslint/js` and `@eslint/jsnext` becomes an alias. That's a separate RFC with a real migration plan; it is not a thing this RFC proposes to do.

**Why is control flow analysis in a parser package?**

Because it reads the parse buffer and the scope buffer directly, by byte offset, and never materializes a node. Splitting it out would mean publishing the binary format as a contract between packages. The package is `sideEffects: false` and the three analyses only reference each other through the functions that need them, so importing one doesn't ship the others.

**Why does the parser refuse to recover from syntax errors?**

Because a linter has nothing useful to say about a file that doesn't parse, and error recovery is where a parser accumulates most of its complexity. It's listed as an open question because editor integrations may disagree.

**Why doesn't the new parser support `ecmaVersion`?**

The `ecmaVersion` option was created around the release of ECMAScript 6, when implementation was taking forever and people needed a way to opt-in or opt-out. We also didn't know how ECMAScript would continue to evolve and at what pace. Since that time, ECMAScript editions get implemented rapidly so, practically speaking, most people only ever want to use `"latest"`, which is why we made that the default. Folks who need `ecmaVersion: 3` or `ecmaVersion: 5` can still use `espree` for that purpose. If we find that this is a significant need in the future, we can always add this option to `validate()` to filter out anything that doesn't belong.

**Why is everything in binary format?**

There are two reasons. First, because profiling revealed that it's the creation of JavaScript objects that contributes most to parsing slowdown. Every extra object, including `range` and `loc` for every node, slows things down considerably. The majority of information we currently have in object form (AST, scope tree, code path tree) is never read, so we're paying the cost of object creation for nothing. By keeping everything in a binary buffer, we're able to avoid object allocation until it's needed.

Second, we can pass binary data back and forth between JavaScript and Rust, as well as between the core and workers, with no cost. Passing data structures back and forth requires serialization and deserialization, which also negatively impacts performance. Binary data is about as free as there can be when crossing language or thread/process boundaries.

## Related Discussions

- [eslint/eslint#18830: Rethinking TypeScript support in ESLint](https://github.com/eslint/eslint/discussions/18830)
- [eslint/eslint#19173: Change Request: Make rules TypeScript syntax-aware](https://github.com/eslint/eslint/issues/19173)
- [eslint/eslint#19373: Change Request: Support `.ts` files with TypeScript syntax by default](https://github.com/eslint/eslint/issues/19373)
- [eslint/eslint#16557: Type-aware linting discussion](https://github.com/eslint/eslint/discussions/16557)
- [RFC 99: ESLint Language Plugins](https://github.com/eslint/rfcs/blob/main/designs/2022-languages/README.md)
- [RFC 135: Allow rules to specify the languages/dialects they work on](https://github.com/eslint/rfcs/blob/main/designs/2025-rule-languages/README.md)
- [eslint/jsnext](https://github.com/eslint/jsnext) — the implementation, including [`docs/deviations.md`](https://github.com/eslint/jsnext/blob/main/docs/deviations.md)
