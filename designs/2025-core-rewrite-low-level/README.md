- Repo: eslint/eslint
- Start Date: 2024-07-15
- RFC PR: (TBD)
- Authors: Nicholas C. Zakas

# ESLint Core Rewrite - Low-Level API

## Summary

This proposal describes how to rewrite the low-level ESLint core from scratch into a composable API. Essentially, splitting `Linter` into small pieces that we can use to build a higher-level API and other utilities. This low-level API is intended to be exposed to the public, so the it's important to get the interfaces correct.

## Motivation

The architecture of the code in the ESLint repository hasn't changed considerably since ESLint was first created 11 years ago. As a result, there is a lot of technical debt that has accumulated and prevents us from making some of the changes we'd like to make. Some of the problems we currently face:

* **Synchronous core logic.** The `Linter` class is completely synchronous, meaning we can't support asynchronous rules or asynchronous parsing, two feature requests we receive frequently. We'd need to either add a second class that works asynchronously or add alternate methods to `Linter` that work asynchronously. Both options looked like significant work, effectively rewriting huge parts of the core.
* **Limited API.** The public API is limited because ESLint, at the start, was never intended to be used as an API. The initial `linter` object was meant only to enable a browser demo. It was later replaced by the `Linter` class, and even later, an `ESLint` class to fully mimic the command line interface. When we want to expose new functionality, it needs to exist in one of these two similar-sounding APIs, and unfortunately, the main decision point is whether or not the API needs to be available in the browser. If so, then it needs to be in `Linter`, otherwise it needs to be in `ESLint`. This is both confusing and not sustainable in the long term.
* **Lack of type checking.** We've had the desire to add type checking into the repository to help catch potential problems, but attempts to do this in the past required a lot of work and coordination. Eventually, we came to the conclusion that it wasn't worth the effort.
* **Stuck in CommonJS.** Similar to the type checking situation, while there is a desire to convert the codebase into ESM, the amount of work it would take to do so is prohibitive. We'd spend a lot of time on the conversion just to end up with an equivalent feature set.

## Detailed Design

This proposal consists of the following changes:

1. Definition of a new composable, session-based API
1. Implementation of the new API inside of the `eslint` package as a sanity check
1. Systematic extraction of APIs as they solidify into `@eslint/core`

This proposal builds on top of the refactoring that has already started in the core. The primary difference for existing classes is the addition of session information.

### API Sketch

Here's how I envision this new API working.

```js
import {
    Session,
    ParserService,
    VFile,
    FileContext,
    FileReport,
    Config,
} from "@eslint/core";

import js from "@eslint/js";

// first create a session
const session = new Session({
    persistent: true,
    cwd: process.cwd(),
    flags: [],
    env: {
        SOME_TOKEN: "foo"
    }
});

// start the session
session.start();

// create a new file
const file = new VFile("path/to/file.js", "let foo = 1");

// create a config
const config = new Config({
    plugins: { js },
    language: "js/js",
    ...js.configs.recommended
});

// create the parser and parse
const parser = new ParserService({ session });
const parseResult = await parser.parse(vfile, config);

// or sync
// const parseResult = parser.parseSync(vfile, config);

if (!parseResult.ok) {
    // handle parseResult.errors
}

const { sourceCode } = parseResult;

// create the file context
const fileContext = new FileContext({ session, file, sourceCode, config });

// create the reporter
const report = new FileReport({ session, sourceCode, config });

// create the visitor from the enabled rules
const visitorCreator = new RuleVisitorCreator({ session });
const visitor = await visitorCreator.create({ config, context: fileContext, report});

// or sync
// visitor = visitorCreator.createSync(config, fileContext);

const traverser = SourceCodeTraverser.getInstance({ session, language });
await traverser.traverse(sourceCode, visitor);

// or sync
// traverser.traverseSync(sourceCode, visitor);

// output linting results
console.log(report.messages);
```

### The `Session` Class

The API proposed in this RFC is based around the concept of a *session*. An ESLint session carries shared state into each object that receives it, allowing developers to avoid passing the same bundle of options around to multiple different objects. 

The `Session` class encapsulates the following shared functionality:

1. **cwd.** The session is the source of truth for the cwd.
1. **Feature flags.** The source of truth for feature flags is the session.
1. **Logging.** All console logging is managed through the session.
1. **Timing.** The session provides the interface for timing stats.

In order to use the new API, you must first create a session, like this:

```js
import { Session } from "@eslint/core";

const session = new Session({
    persistent: true,           // required: is the session persistent (i.e., in an IDE) or not?
    cwd: process.cwd(),         // optional: default is process.cwd()
    flags: [],                  // optional: default is []
    logLevel: "debug",          // optional: default is "warn"
    statsEnabled: true,         // optional: default is false
});
```

The `persistent` option is required because we have noted before that there's a difference between a session that is persistent (such as in an IDE) vs. one that is not (such as a CLI). While this proposal does not define how we should handle the differences, having this information required up front will help us down the road.

The `Session` interface is as follows:

```ts
interface Session extends EventEmitter {
    
    /**
     * An autogenerated unique identifier for the session.
     */
    readonly id: string;

    /**
     * The current working directory for the session.
     */
    readonly cwd: string;
    
    /**
     * Whether or not the session is persistent.
     */
    readonly persistent: boolean;
    
    /**
     * The enabled flags for this session.
     */
    readonly flags: Record<string,boolean>;
    
    /**
     * Environment variables for the session.
     */
    readonly env: Record<string, string>;
    
    /**
     * Indicates if stats should be collected for this session.
     */
    readonly statsEnabled: boolean;

    /**
     * Indicates if the session is started.
     */
    readonly started: boolean;

    /**
     * The age of the session in milliseconds. Starts when `start()` is called
     * and doesn't stop.
     */
    readonly age: number;
    
    /**
     * The length of the session. Starts when `start()` is called and stops when
     * `stop()` is called. Easier to use for analytics than age because it
     * remains at its final value.
     */
    readonly length: number;

    /**
     * Start the session.
     */
    start(): void;
    
    /**
     * Stop the session.
     */
    stop(): void;

    /**
     * Throw an error if not started. Useful for any methods that should not
     * be used unless the session is started.
     */
    assertStarted(): void;
    
    /**
     * Serialize the session for creating clones in other threads or processes.
     * Preserves `id` and other serializable options.
     */
    toJSON(): Record<string,any>;
    
    /**
     * Deserialize into a clone of the given session information.
     */
    static fromJSON(Record<string,any>);
}
```

Each `Session` instance is also an event emitter, allowing us to emit events at key points during the session's lifecycle. This allows for better logging and debugging.

### The `VFile` Class

We currently have a [`VFile` class](https://github.com/eslint/eslint/blob/main/lib/linter/vfile.js), and this should be part of the new core. The purpose of the `VFile` class is to represent a file with all of the necessary information ESLint needs to work with it, including whether or not it has a BOM. This doesn't need session information because it's strictly a data structure.

```ts
interface VFile {
    path: string;
    physicalPath: string;
    body: string | Uint8Array;
    rawBody: string | Uint8Array;
    bom: boolean;
}
```

### The `Config` Class

We currently have a [`Config` class](https://github.com/eslint/eslint/blob/main/lib/config/config.js). The `Config` class represents a single, normalized config object that applies to a single file. While the CLI pulls normalized configs out of a `FlatConfigArray`, most of the APIs in this RFC require only a single, normalized config object, and it doesn't make sense to force people to go through the exercise of creating a `FlatConfigArray` instance just to get a single object.

The `Config` constructor accepts an object, which is an unnormalized config object, and throws errors if there are any validation problems. A `Config` instance normalizes `language` to a `Language` instance and `processor` to a `Processor` instance. Otherwise, all of the keys are the same as in any configuration object.

```ts
interface Config {
    toJSON(): Record<string, any>;
    getRuleDefinition(ruleId: string): RuleDefinition | undefined;
    validateRulesConfig(rulesConfig: object): void;
}
```

### The `ParserService` Class

We currently have a [`ParserService` class](https://github.com/eslint/eslint/blob/main/lib/services/parser-service.js). The `ParserService` class is responsible for parsing source files using the appropriate parser for the given language and configuration. It is constructed with a `Session` instance and provides both asynchronous and synchronous methods for parsing files. The result of parsing includes the AST and any errors encountered during parsing.

```ts
type ParseResult = 
    { ok: true; sourceCode: SourceCode } |
    { ok: false; errors: ParseError[] };

interface ParserService {
    parse(file: VFile, config: Config): Promise<ParseResult>;
    parseSync(file: VFile, config: Config): ParseResult;
}
```

### The `FileContext` Class

We currently have a [`FileContext` class](https://github.com/eslint/eslint/blob/main/lib/linter/file-context.js). The `FileContext` class encapsulates all the contextual information needed to lint a single file. This includes the session, the file (`VFile`), the parsed source code, and the configuration (`Config`). It acts as a container for all state and data relevant to a single linted file.

```ts
interface FileContext {
    cwd: string;
    filename: string;
    physicalFilename: string;
    sourceCode: SourceCode;
    parserOptions: Record<string, unknown>;
    parserPath: string;
    languageOptions: Record<string, unknown>;
    settings: Record<string, unknown>;

    extend(extension: object): FileContext;
}
```

### The `FileReport` Class

The `FileReport` class is responsible for collecting and managing linting messages produced during the analysis of a file. It is constructed with a `Session` instance and provides an interface for storing, retrieving, and formatting messages. The report is passed to rule visitors.

```ts
interface FileReport {
    /** The messages reported by the linter for this file. */
    messages: LintMessage[];

    /**
     * Adds a rule-generated message to the report.
     * @param ruleId The rule ID that reported the problem.
     * @param severity The severity of the problem (0 = off, 1 = warning, 2 = error).
     * @param args The arguments passed to `context.report()`.
     * @returns The created problem object.
     */
    addRuleMessage(ruleId: string, severity: 0 | 1 | 2, ...args: any[]): LintMessage;

    /**
     * Adds an error message to the report. Meant to be called outside of rules.
     * @param descriptor The descriptor for the error message.
     * @returns The created problem object.
     */
    addError(descriptor: LintProblem): LintMessage;

    /**
     * Adds a fatal error message to the report. Meant to be called outside of rules.
     * @param descriptor The descriptor for the fatal error message.
     * @returns The created problem object.
     */
    addFatal(descriptor: LintProblem): LintMessage;

    /**
     * Adds a warning message to the report. Meant to be called outside of rules.
     * @param descriptor The descriptor for the warning message.
     * @returns The created problem object.
     */
    addWarning(descriptor: LintProblem): LintMessage;
}
```

### The `RuleVisitorCreator` Class

The `RuleVisitorCreator` class is responsible for creating the visitor object that will be used to traverse the AST and apply enabled rules. It is constructed with a `Session` instance and provides methods to create the visitor asynchronously or synchronously, given a configuration, file context, and reporter.

```ts
interface RuleVisitorCreator {
    create(options: { config: Config; context: FileContext; report: FileReport }): Promise<ASTVisitor>;
    createSync(config: Config, context: FileContext): ASTVisitor;
}

interface ASTVisitor {
    [nodeType: string]: (node: any) => void;
}
```

**Note:** The `ASTVisitor` interface is intentionally generic and not rule-specific. It is possible we will have other types of visitors besides those that run rules (such in [prelint plugins](https://github.com/eslint/rfcs/pull/105)). This approach gives us flexibility to add other visitor types in the future.

### The `SourceCodeTraverser` Class

We currently have a [`SourceCodeTraverser` class](https://github.com/eslint/eslint/blob/main/lib/linter/source-code-traverser.js). The `SourceCodeTraverser` class provides methods for traversing the source code's AST. It is typically used as a singleton, obtained via `getInstance()`, and is constructed with a `Session` and language. The traverser can operate asynchronously or synchronously and is responsible for walking the AST and invoking the appropriate visitor methods.

```ts
interface SourceCodeTraverser {
    
    traverse(sourceCode: SourceCode, visitor: ASTVisitor, options?: { steps?: TraversalStep[] }): Promise<void>;
    traverseSync(sourceCode: SourceCode, visitor: ASTVisitor, options?: { steps?: TraversalStep[] }): void;

    /** Each instance is specific to a Language, so it's more performant to reuse instances when possible. */
    static getInstance(language: Language): SourceCodeTraverser;
}
```


## Documentation

TODO

## Drawbacks

## Backwards Compatibility Analysis

This proposal is 100% backwards compatible. Converting `Linter` to use the new APIs while ensuring existing tests pass allows everything to continue to work even while pulling components out of the `eslint` package.

## Alternatives

The primary alternative is to keep iterating on the `Linter` and `ESLint` interfaces as the public API. We could gradually transition `Linter` to having async methods for all current synchronous methods. This would give us the flexibility to implement async parsing and rules but without the benefit of any real concept of a session.

We could also layer the idea of a persistent session on top of the existing API by passing around a flag to let us know when a session is intended to be long-lived. This information could then be exposed to plugins as a way to better manage their resources. 

## Open Questions

### Should `persistent` be required for `Session`?

The risk here is that people will forget to set this value and so the default would have to be logical. If the default is `false` then this essentially is the same as what we have now. In that case, anyone who needs a persistent session would have to manually set `persistent` to `true`. The downside of that is persistent sessions are the most troublesome and would be nice to not to miss that option. Especially as it relates to plugins that need to know whether or not they should clean up their caches periodically, it's a significant problem if developers forget to set `persistent: true`.

We could set the default to `true`. In that case, plugins may put processes into place that just aren't needed, slowing down the overall experience.

It seems like forcing a value for `persistent` ensures we get the best of both worlds, but this is definitely worth discussing.

### Should `Session` have a `console` utility?

In general, the only time we output anything to the console is either using `console.log()` in the CLI or `process.emitWarning()` to output warnings. It could be beneficial to provide a `console` utility, which would also make it easier to suppress output for testing if we enforce usage of it. Maybe this could combine the current `WarningService` and `globalThis.console`? Also, maybe we should be using `console.error()` more than we currently are?

### Is it confusing that `language` and `processor` are normalized to objects in `Config`?

In a user-facing config, `language` is always a string while `processor` can be a string or object. We could leave these keys as their original values and add more properties for the normalized ones (i.e., `languageInstance` and `processorInstance`). I'm not sure which option is better.

### How to handle statistics with `Session`?

The statistics we currently store are on a per-run basis of the CLI. The assumption is that the session is not persistent and all of the available data is for everything that happened during the session. This leads to several questions:

1. How do we track statistics for a persisent session?
1. Does it even make sense to track statistics at the session level, or is this going to be the responsibility of some other part of ESLint (e.g., the CLI)?
1. If we do want to track statistics on the session, what kind of API do we expose for that?

### How to expose this info to rules?

We could create a `context.session` property to allow rules to learn something about the session. This should likely *not* be the `Session` instance itself but rather a smaller subset of the available properties. What would that subset be?

### How to expose this info to plugins?

For languages, we can add a `session` property to the [`LanguageContext`](https://github.com/eslint/rewrite/blob/f5e6d683ee00b24b98777291c0a9a83719fe3402/packages/core/src/types.ts#L807-L809) type. I think this can be the `Session` object itself? That would expose the session inside the `parse()` and `createSourceCode()` methods.

I'm not sure where else session information might be useful in a plugin. Without some constructor or initialization API (which might be worth exploring in some future RFC), is there any other location where this info might be helpful?

## Help Needed

N/A

## Frequently Asked Questions

### Why `persistent` instead of `persist` on `Session`?

`persistent` is an adjective that describes how the session is used. `persist` is a verb that implies the session is doing something special when the option is set, but really, it's just a flag that's carried through the session to let other components understand the context in which they're executed.

### Why do we have methods and `*Sync` methods?

The long-term goal is to move the core to be async, and so the naming reflects the preference for async APIs. Additionally, Node.js internal APIs created a de facto standard by naming synchronous APIs with the `Sync` suffix, so this is familiar to JavaScript developers.

The sync APIs are there primarily so we can reimplement `Linter` on top of the new core APIs without changing functionality.

## Related Discussions

* [#15475 Change Request: Allow async parser](https://github.com/eslint/eslint/issues/15475)
* [#15394 Change Request: Support async rules](https://github.com/eslint/eslint/issues/15394)
