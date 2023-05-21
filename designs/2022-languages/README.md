- Repo: eslint/eslint
- Start Date: 2022-11-25
- RFC PR: https://github.com/eslint/rfcs/pull/99
- Authors: Nicholas C. Zakas

# ESLint Language Plugins

## Summary

This proposal provides a way for plugin authors to define parsers, rules, etc., for any arbitrary programming languages. This would allow ESLint to work on any type of programming language for which there is a plugin defined, not just for JavaScript and its variants.

## Motivation

Over the nine years ESLint has been in existence, it has grown into being not just the most popular JavaScript linter, but arguably the most popular linter for any programming language. Twitter is filled with people lamenting that they don't have an "ESLint for X", where X is some other file type they're working with. Numerous projects have popped up that mimic a lot of what ESLint does at it's core:

* [Stylelint](https://stylelint.io/);
* [Markdownlint](https://github.com/DavidAnson/markdownlint)

In addition, there are numerous projects that hack around other languages to make them work in ESLint, such as:

* [GraphQL-ESLint](https://github.com/B2o5T/graphql-eslint)
* [eslint-plugin-json](https://www.npmjs.com/package/eslint-plugin-json)
* [typescript-eslint](https://typescript-eslint.io)

Instead of forcing others to rewrite what is core linting functionality, or asking people to produce an ESTree format AST for some language other than JavaScript, we can make everyone's lives easier by providing formal language support such that languages can be plugged in to ESLint easily.

We did also previously accept [#56](https://github.com/eslint/rfcs/pull/56) as an extension to `parseForESLint()` to support more languages and [eslint/eslint#14745](https://github.com/eslint/eslint/issues/14745) has been open for a couple of years to figure out a way to lint both a file and its virtual parts.

## Detailed Design

This proposal consists of the following changes:

1. Allows plugins to publish languages in addition to rules, processors, etc.
1. Allows end users to select which language to use in their config files.
1. Defines an API for defining languages.
1. Changes the ESLint core to consume language definitions instead of assuming JavaScript is always used.

**Prerequisites:** This proposal requires the flat config system and cannot be used with the eslintrc config system.

**Note:** This design is intended to work both in the current version of ESLint and also in the "rewrite" version, which I'll refer to as ESLint X to distinguish between the two. Not all features of this proposal will be available in both versions. 

### Languages in plugins and configs

Languages can be defined in plugins by using the `languages` key and giving the language a name. This allows multiple languages to be contained in a single plugin if desired. For example:

```js
// inside a plugin called eslint-plugin-lang
export default {

    languages: {
        lang1: {
            // language object
        },
        lang2: {
            // language object
        }
    },
    processors: {
        // ..
    },
    rules: {
        // ...
    }
};
```

Here, `lang1` and `lang2` are languages defined by the plugin. End users can specify which language to use via the `language` key in a config object, such as:


```js
// inside eslint.config.js
import lang from "eslint-plugin-lang";

export default [
    {
        plugins: {
            lang
        },
        files: ["**/*.lang"],
        language: "lang/lang1",
        languageOptions: {
            // options
        }
    }
];
```

Here, `"lang/lang1"` refers to the `lang1` language as exported from `eslint-plugin-lang`. Anything inside of `languageOptions` is then handled by `lang1` rather than by ESLint itself. 

### Language Definition Object

Each language definition object must implement the following interface:

```ts
interface ESLintLanguage {

    /**
     * Indicates how ESLint should read the file.
     */
    fileType: "text" | "binary";

    /**
     * First line number returned from the parser (text mode only).
     */
    lineStart?: 0 | 1;

    /**
     * First column number returned from the parser (text mode only).
     */
    columnStart?: 0 | 1;

    /**
     * The property to read the node type from. Used in selector querying.
     */
    nodeTypeKey: string;

    /**
     * The traversal path that tools should take when evaluating the AST
     */
    visitorKeys: Record<string,Array<string>>;

    /**
     * Validates languageOptions for this language.
     */
    validateOptions(options: LanguageOptions): void;

    /**
     * Helper for esquery that allows languages to match nodes against
     * class. esquery currently has classes like `function` that will
     * match all the various function nodes. This method allows languages
     * to implement similar shorthands.
     */
    matchesSelectorClass(className: string, node: ASTNode, ancestry: Array<ASTNode>): boolean;

    /**
     * Parses the given file input into its component parts. This file should not
     * throws errors for parsing errors but rather should return any parsing 
     * errors as parse of the ParseResult object.
     */
    parse(file: File, context: LanguageContext): ParseResult | Promise<ParseResult>;

    /**
     * Creates SourceCode object that ESLint uses to work with a file.
     */
    createSourceCode(file: File, input: ParseResult, context: LanguageContext): SourceCode | Promise<SourceCode>;

}

// Supplemental interfaces

type LanguageOptions = Record<string, unknown>;

interface File {

    /**
     * The path that ESLint uses for this file. May be a virtual path
     * if it was returned by a processor.
     */
    path: string;

    /**
     * The path to the file on disk. This always maps directly to a file
     * regardless of whether it was returned from a processor.
     */
    physicalPath: string;

    /**
     * Indicates if the original source contained a byte-order marker.
     * ESLint strips the BOM from the `body`, but this info is needed
     * to correctly apply autofixing.
     */
    bom: boolean;

    /**
     * The body of the file to parse.
     */
    body: string | ArrayBuffer
}

interface LanguageContext {
    options: LanguageOptions;
}

interface ParseResult {

    /**
     * Indicates if the parse was successful. If true, the parse was successful
     * and ESLint should continue on to create a SourceCode object and run rules;
     * if false, ESLint should just report the error(s) without doing anything
     * else.
     */
    ok: boolean;

    /**
     * The abstract syntax tree created by the parser.
     */
    ast: ASTNode;

    /**
     * The body of the file, either text or bytes.
     */
    body: string | ArrayBuffer;

    /**
     * Any parsing errors, whether fatal or not.
     */
    errors?: Array<ParseError>;
}

interface ParseError {
    message: string;
    line: number;
    column: number;
    endLine?: number;
    endColumn?: number;
}

interface ASTNode {
    // language-defined
}
```

At a high-level, ESLint uses the methods on a language object in the following way:

* During config validation, ESLint calls `validateOptions()` on the `languageOptions` specified in the config. It is the expectation that this method throws an error if any of the options are invalid. 
* When ESLint reads a file, it checks `fileType` to determine whether the language would prefer the file in text or binary format.
* When preparing violation messages, ESLint uses `lineStart` and `columnStart` to determine how to offset the locations. Some parsers use line 0 as the first line but ESLint normalizes to line 1 as the first line (similar for columns). Using `lineStart` and `columnStart` allows ESLint to ensure a consistently reported output so one file doesn't start with line 0 and another starts at line 1. If not present, `lineStart` and `columnStart` are assumed to be 0.
* After reading a file, ESLint passes the file information to `parse()` to create a raw parse result.
* The raw parse result is then passed into `createSourceCode()` to create the `SourceCode` object that ESLint will use to interact with the code.
* The `nodeTypeKey` property indicates the property key in which each AST node stores the node name. Even though ESTree uses `type`, it's also common for ASTs to use `kind` to store this information. This property will allow ESLint to use ASTs as they are instead of forcing them to use `type`. This is important as it allows for querying by selector.

#### The `validateOptions()` Method

The intent of this method is to validate the `languageOptions` object as specified in a config. With flat config, all of the options in `languageOptions` are related to JavaScript and are currently validated [inside of the schema](https://github.com/eslint/eslint/blob/6380c87c563be5dc78ce0ddd5c7409aaf71692bb/lib/config/flat-config-schema.js#L445). With this proposal, that validation will instead need to be based on the `language` specified in the config, which will require retrieving the associated language object and calling the `validateOptions()` method [inside of `FlatConfigArray`](https://github.com/eslint/eslint/blob/f89403553b31d24f4fc841424cc7dcb8c3ef689f/lib/config/flat-config-array.js#L180).

The `validateOptions()` method must throw an error if any part of the `languageOptions` object is invalid.

No specific `languageOptions` keys will be required for languages. The `parser` and `parserOptions` keys are unique to JavaScript and may not make sense for other languages.

#### The `matchesSelectorClass()` Method

Inside of `esquery`, there are [some shortcuts](https://github.com/estools/esquery/blob/7c3800a4b2ff5c7b3eb3b2cf742865b7c908981f/esquery.js#L213-L233) like `function` and `expression` that will match more than one type of node. These are all specific to ESTree and might not make sense for other languages. However, these shortcut classes are very convenient and other languages might want to implement something similar.

The `esquery` [External class resolve](https://github.com/estools/esquery/pull/140) pull request implements an option that allows us to pass in a function to interpret pseudoclasses from `ESLintLanguage#matchesSelectorClass()`.

#### The `parse()` Method

The `parse()` method receives the file information and information about the ESLint context in order to correctly parse the file. This is a raw parse, meaning that `parse()` should do nothing more than convert the file into an abstract format without doing any postprocessing. This is important so that ESLint can tell how long parsing takes vs. other operations. (One problem with the current `parseForESLint()` approach is that we lose visibility into the true parsing time.)

The `parse()` method is expected to return an object with at least `ast` and `body` properties; the object may also have other properties as necessary for the language to properly create a `SourceCode` object later. The return value from `parse()` is passed directly into `createSourceCode()`.

This method will be called [inside of `Linter`](https://github.com/eslint/eslint/blob/28d190264017dbaa29f2ab218f73b623143cd1af/lib/linter/linter.js#L805) and will require that the language object be passed into the private `parse()` function.

**Async:** For use with current ESLint, `parse()` must be synchronous; ESLint X will allow this method to be asynchronous.

#### The `createSourceCode()` Method

The `createSourceCode()` method is intended to create an instance of `SourceCode` suitable for ESLint and rule developers to use to work with the file. As such, it may perform postprocessing of a parse result in order to get it into a format that is easier to use.

This method will be called [inside of `Linter`](https://github.com/eslint/eslint/blob/28d190264017dbaa29f2ab218f73b623143cd1af/lib/linter/linter.js#L828) and will require that the language object be passed into the private `parse()` function.

**Async:** For use with current ESLint, `createSourceCode()` must be synchronous; ESLint X will allow this method to be asynchronous.

### The `SourceCode` Object

ESLint already has a `SourceCode` class that is used to [postprocess and organize the AST data](https://github.com/eslint/eslint/blob/f89403553b31d24f4fc841424cc7dcb8c3ef689f/lib/source-code/source-code.js#L149) for JavaScript files. This proposal extends the intended purpose of `SourceCode` such that it encapsulates all functionality related to a single file, including traversal. As such, `SourceCode` objects must implement the following interface:

```ts
interface SourceCode {

    /**
     * Root of the AST.
     */
    ast: ASTNode;

    /**
     * The body of the file that you'd like rule developers to access.
     */
    body: string | ArrayBuffer;

    /**
     * The traversal path that tools should take when evaluating the AST
     */
    visitorKeys?: Record<string,Array<string>>;

    /**
     * Traversal of AST.
     */
    traverse(): Iterable<TraversalStep>;

    /**
     * Return all of the inline areas where ESLint should be disabled/enabled.
     */
    getDisableDirectives(): Array<DisableDirective>;

    /**
     * Return any inline configuration provided via comments.
     */
    getInlineConfig(): Array<FlatConfig>;
}

type TraversalStep = VisitTraversalStep | CallTraversalStep;

interface VisitTraversalStep {
    type: "visit";
    target: ASTNode;
    phase: "enter" | "exit";
    args: Array<any>;
}

interface CallTraversalStep {
    type: "call";
    target: string;
    phase: string | null | undefined;
    args: Array<any>;
}

interface DisableDirective {
    type: "enable" | "disable";
    ruleId: string;

    /**
     * The point from which the directive should be applied.
     */
    from: Location;

    /**
     * Location of the comment.
     */
    loc: LocationRange;
}

interface LocationRange {
    start: Location;
    end: Location;
}

interface Location {
    line: number;
    column: number;
}
```

Other than these interface members, languages may define any additional methods or properties that they need to provide to rule developers. For instance, 
the JavaScript `SourceCode` object currently has methods allowing retrieval of tokens, comments, and lines. It is up to the individual language object implementations to determine what additional properties and methods may be required inside of rules. (We may want to provide some best practices for other methods, but they won't be required.)

#### The `SoureCode#visitorKeys` Property

The JavaScript language allows a `parser` option to be passed in, and so the result AST may not be the exact structure represented on the `ESLintLanguage` object. In such a case, an additional `visitorKeys` property can be provided on `SourceCode` that overrides the `ESLintLanguage#visitorKeys` property just for this file.

#### The `SoureCode#traverse()` Method

Today, ESLint uses a [custom traverser](https://github.com/eslint/eslint/blob/b3a08376cfb61275a7557d6d166b6116f36e5ac2/lib/shared/traverser.js) based on data from [`eslint-visitor-keys`](https://npmjs.com/package/eslint-visitor-keys). All of this is specific to an ESTree-compatible AST structure. There are two problems with this:

1. We cannot traverse anything other than ESTree structures.
1. Because this traversal is synchronous, rules must always be synchronous.

The `SourceCode#traverse()` method solves these problems by allowing each `SourceCode` instance to specify the traversal order through an iterable. Returning an iterable means several things:

1. The method can be a generator or just return an array. This gives maximum flexibility to language authors.
1. The ESLint core can move step by step through the traversal, waiting for asynchronous calls to return before moving to the next node. This will ultimately enable async rules in ESLint X.
1. It's possible to add method calls into the traversal, such as the current `onCodePathStart()` method for JavaScript, in a standard way that is available to other languages.
1. If the `SourceCode` object ends up doing more than one traversal of the same AST (which is how ESLint currently work), it can cache the traversal steps in an array to reuse multiple times, saving compute time.

As an example, the [current traversal](https://github.com/eslint/eslint/blob/b3a08376cfb61275a7557d6d166b6116f36e5ac2/lib/shared/traverser.js#L127) can be rewritten to look like this:

```js
class Traverser {

    // methods skipped

    *_traverse(node, parent) {
        if (!isNode(node)) {
            return;
        }

        this._current = node;
        this._skipped = false;
        yield {
            type: "visit",
            target: node,
            phase: "enter",
            args: [node, parent]
        }

        if (!this._skipped && !this._broken) {
            const keys = getVisitorKeys(this._visitorKeys, node);

            if (keys.length >= 1) {
                this._parents.push(node);
                for (let i = 0; i < keys.length && !this._broken; ++i) {
                    const child = node[keys[i]];

                    if (Array.isArray(child)) {
                        for (let j = 0; j < child.length && !this._broken; ++j) {
                            yield *this._traverse(child[j], node);
                        }
                    } else {
                        yield *this._traverse(child, node);
                    }
                }
                this._parents.pop();
            }
        }

        if (!this._broken) {
            yield {
                type: "visit",
                target: node,
                phase: "exit",
                args: [node, parent]
            }
        }

        this._current = parent;
    }

}
```

Note that we use this data inside of [`NodeEventGenerator`](https://github.com/eslint/eslint/blob/90a5b6b4aeff7343783f85418c683f2c9901ab07/lib/linter/node-event-generator.js) to make the calls into the visitor objects.

Using `SourceCode#traverse()`, we will be able to traverse an AST like this:

```js
// Simplified example only!! -- not intended to be the final implementation
for (const step of sourceCode.traverse()) {

    switch (step.type) {

        // visit a node
        case "node": {
            rules.forEach(rule => {
                Object.keys(rule).forEach(key => {

                    // match the selector
                    if (sourceCode.match(step.target, key)) {

                        // account for enter and exit phases
                        if (key.endsWith(":exit")) {
                            if (step.phase === "exit") {

                                // async visitor methods!!
                                await rule[key](...step.args)
                            }
                        } else {
                            await rule[key](...step.args)
                        }
                    }
                });
            });
            break;
        }

        // call a method
        case "call": {
            rules.forEach(rule => {
                Object.keys(rule).forEach(key => {
                    if (step.target === key) {
                        await rule[key](...step.args)
                    }
                });
            });
            break;
        }
        default:
            throw new Error(`Invalid step type "${ step.type }" found.`);
    }
}
```

In ESLint X, we can swith the `for-of` loop for a `for await-of` loop in order to allow asynchronous traversal, as well.

#### The `SoureCode#getDisableDirectives()` Method

ESLint has all kinds of disable directives to use inside of JavaScript. It's impossible for the ESLint core to know how disable directives might be formatted or used in other languages, so we need to abstract this out and delegate it to the `SourceCode` object. The `SourceCode#getDisableDirectives()` method returns an array of disable directive objects indicating when each rule is disabled and enabled.

The ESLint core will then use the disable directive information to:

* Filter out violations appropriately.
* Report any unused disable directives.

This functionality currently lives in [`apply-disable-directives.js`](https://github.com/eslint/eslint/blob/0311d81834d675b8ae7cc92a460b37115edc4018/lib/linter/apply-disable-directives.js) and will have to be updated to call `SourceCode#getDisableDirectives()`.

This is a method instead of a property because if ESLint is run with `noInlineConfig: true` then there is no reason to calculate the disable directives.

#### The `SoureCode#getInlineConfig()` Method

In JavaScript, we use `/* eslint */` comments to specify inline configuration of rules. Other languages may want to do the same thing, so this method provides a way for a language to extract any inline configuration and return it to the ESLint core.

This functionality currently lives in [`linter.js`](https://github.com/eslint/eslint/blob/ea10ca5b7b5bd8f6e6daf030ece9a3a82f10994c/lib/linter/linter.js#L466) and will have to be updated to call `SourceCode#getInlineConfig()`.

This is a method instead of a property because if ESLint is run with `noInlineConfig: true` then there is no reason to calculate inline config.

### Extracting JavaScript Functionality

An important part of this process will be extracting all of the JavaScript functionality for the ESLint core and placing it in a language object. As a first step, that language object will live in the main ESLint repository so we can easily test and make changes to ensure backwards compatibility. Eventually, though, I'd like to move this functionality into its own `eslint/js` repo.

#### Update Default Flat Config

The default flat config will need to be updated to specify a default `language` to use. We would need to hardcode the language for eslintrc configs as there won't be a way to set that in the config itself. (Likely, `config.language || defaultLanguage` in `Linter`.)

#### Move Traversal into `SourceCode`

The JavaScript-specific traversal functionality that currently lives in [`linter.js`](https://github.com/eslint/eslint/blob/dfc7ec11b11b56daaa10e8e6d08c5cddfc8c2c59/lib/linter/linter.js#L1140-L1158) will need to move into `SourceCode#traverse()`.

#### Generic AST selectors

ESLint currently uses [`esquery`](https://npmjs.com/package/esquery) to match visitor patterns to nodes. `esquery` is defined for use specifically with ESTree-compatible AST structures, which means that it cannot work for other structures without modification. 

To make `esquery` more generic, I've submitted the following pull requests:

* [Allow for custom node type keys](https://github.com/estools/esquery/pull/139) that implements an option to specify which key in an AST node contains the node type.
* [External class resolve](https://github.com/estools/esquery/pull/140) implements an option that allows us to pass in a function to interpret pseudoclasses like `:expression` from `ESLintLanguage#matchesSelectorClass()`.

#### Split Disable Directive Functionality

The [`applyDisableDirectives()`](https://github.com/eslint/eslint/blob/dfc7ec11b11b56daaa10e8e6d08c5cddfc8c2c59/lib/linter/linter.js#L1427-L1434) method is called inside of `Linter`, and if `Linter` calls `SourceCode#getDisableDirectives()`, we can likely reuse a lot of the existing functionality with some minor tweaks.

The [`getDirectiveComments()`](https://github.com/eslint/eslint/blob/dfc7ec11b11b56daaa10e8e6d08c5cddfc8c2c59/lib/linter/linter.js#L367) function will move into `SourceCode` to postprocess the AST with all directive comments and then expose just the disable comments through `SourceCode#getDisableDirectives()`.

#### Move Rule Context Methods and Properties to `SourceCode`

For JavaScript, we will need to deprecate the following methods and properties and redirect them to the `SourceCode` object:

* `getAncestors()`
* `getDeclaredVariables()`,
* `getScope()`
* `parserServices`

The following properties we can remove once we remove the eslintrc config system, so they will still show up for JavaScript but won't be included for non-JavaScript:

* `parserOptions`
* `parserPath`

#### Update Rules to New APIs

With a new Rule Context, we should also start converting existing rules over to the new APIs as they move from `context` onto `SourceCode`.

### Core Changes

In order to make all of this work, we'll need to make the following changes in the core:

1. [`linter.js`](https://github.com/eslint/eslint/blob/dfc7ec11b11b56daaa10e8e6d08c5cddfc8c2c59/lib/linter/linter.js#L1140-L1158) will need to be updated to call `SourceCode#traverse()`.
1. `FlatConfigArray` will need to be updated to define the `language` key and to delegate validation of `languageOptions` to the language.
1. `Linter` will need to be updated to honor the `language` key and to use JS-specific functionality where we need to provide backwards compatibility for existing JS rules. There will be a lot of code removed and delegated to the language being used, including filtering of violation via disable directives, traversing the AST, and formulating the final violation list.
1. The `context` object passed to rules will need to be updated so it works for all languages.
1. `FlatRuleTester` will need to be updated to support passing in `language`. We should also updated `FlatRuleTester` to warn about the newly-deprecated `context` methods and properties.

#### Updating Rule Context

The rule context object, which is passed into rules as `context`, will need to be significantly altered to support other languages. All of the methods that relate specifically to language features will need to move into `SourceCode`, leaving us with a rule context object that follows this interface:

```ts
interface RuleContext {

    languageOptions: LanguageOptions;
    settings: object;
    options: Array<any>;
    id: string;

    // method replacements
    cwd: string;
    filename: string;
    physicalFilename: string
    sourceCode: SourceCode;

    report(violation: Violation): void;

    // deprecated methods only available in JS
    getCwd(): string;
    getFilename(): string;
    getPhysicalFilename(): string;
    getSourceCode(): SourceCode;
}

interface Violation {
    messageId: string;
    data: object;
    line: number;
    column: number;
    endLine?: number;
    endColumn?: number
    suggest: Array<Suggestion>;
    fix(fixer: Fixer): Iterable<FixOp>;
}

interface Suggestion {
    desc: string;
    fix(fixer: Fixer): Iterable<FixOp>;
}
```

We should strictly use this interface for all non-JavaScript languages from the start.

**Note:** I have purposely removed the `message` property from the `Violation` interface. I'd like to move to just one way of supplying messages inside of rules and have that be `messageId`. This will also reduce the complexity of processing the violations.

## Documentation

Most existing end-user facing documentation will remain unchanged.

Plugin developer documentation will need to be updated:
* The documentation on writing new rules needs to be updated to include API changes.
* A new page on writing languages for ESLint will need to be written. This will need to be quite extensive.

## Drawbacks

1. **Performance.** There will likely be a small performance penalty from this refactoring. Moving to iterators from standard loops seems to incur a slowdown. My hope is that we can mitigate the effects of this performance hit by refactoring code in other ways.
1. **eslintrc Compatibility.** Unfortunately, there's no easy way to make this functionality available to eslintrc config users. When implemented, eslintrc users will be left without the ability to use other languages.

## Backwards Compatibility Analysis

This proposal is 100% backwards compatible in the short-term, and all breaking changes are optional and can be phased in over time to avoid disrupting the ecosystem.

This proposal does not require any changes from end-users.

## Alternatives

1. We could continue using the `parserForESLint()` approach to support other languages. This was proposed and accepted in [#56](https://github.com/eslint/rfcs/pull/56).
1. We could choose to not support languages other than JavaScript.

## Open Questions

### Should we eliminate `:exit` from selectors?

The `:exit` was left over from before we used `esquery` and is a bit of an outlier from how selectors generally work. We could eliminate it and ask people to write their rule visitors like this:

```js
{
    "FunctionExpression": {
        enter(node, parent) {
            // ...
        },
        exit(node, parent) {
            // ...
        }
    }
}
```

And if they just wanted to use `:exit`, we could ask them to write this:

```js
{
    "FunctionExpression": {
        exit(node, parent) {
            // ...
        }
    }
}
```

This would allow the visitor keys to stay strictly as selectors while still allowing both enter and exit phases of traversal to be hit.

## Help Needed

N/A

## Frequently Asked Questions

### Why allow binary files?

I think there may be a use in validating non-text files such as images, videos, and WebAssembly files. I may be wrong, but by allowing languages to specify that they'd prefer to receive the file in binary, it at least opens up the possibility that people may create plugins that can validate binary files in some way.

### How will we autofix binary files?

At least to start, we won't allow autofix for binary files. While it is technically possible to overwrite particular byte offsets in the same way we do character ranges, it's difficult to know how this should work in practice without an actual use case. My suggestion is that we implement linting for binary files and see what people end up using it for before designing autofix for binary files.

### Will the JavaScript language live in the main eslint repo?

As a first step, yes, the JavaScript language object will still live in the main eslint repo. This will allow us to iterate faster and keep an eye on backwards compatibility issues.

In the future, I anticipate moving all JavaScript-specific functionality, including rules, into a separate repository (`eslint/js`) and publishing it as a separate package (`@eslint/js`). That would require some thought around how the website should change and be built as rule documentation would also move into a new repo.

### Why not have `parse()` return a `SourceCode` instead of needing to call another method?

It's important that we are able to determine the amount of time it takes to parse a file in order to optimize performance. Combining the parsing with creating a `SourceCode` object, which may require additional processing of the parse result, means obscuring this important number.

Additionally, `createSourceCode()` allows integrators to do their own parsing and create their own `SourceCode` instances without requiring the use of a specific `parse()` method.

### How will rules be written for non-JS languages?

Rules will be written the same way regardless of the language being used. The only things that will change are the AST node names and the `SourceCode` object passed in. Otherwise, rules can be written the same way, including the use of CSS-like selector strings to find nodes.

### Will non-JS languages be able to autofix?

Yes. The same text-based mechanism we currently use for JavaScript autofixes will also work for other languages.

### What about AST mutation for autofixing?

That is out of scope for this RFC.

### Will the ESLint team create and maintain additional languages?

Yes. In the short-term, we will create a JSON language plugin and a Markdown language plugin (to coincide with or replace `eslint-plugin-markdown`).

We may kickstart other language plugins as well and invite others to maintain them.

## Related Discussions

* [#6974 Proposal for parser services](https://github.com/eslint/eslint/issues/6974)
* [#8392 Allow parsers to supply their visitor keys](https://github.com/eslint/eslint/issues/8392)
* [#15475 Change Request: Allow async parser](https://github.com/eslint/eslint/issues/15475)
