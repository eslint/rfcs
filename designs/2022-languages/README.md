- Repo: eslint/eslint
- Start Date: 2022-11-25
- RFC PR: (leave this empty, to be filled in later)
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
exports default {

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

exports default [
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

### Language Definition Objects

Each language definition object must implement the following interface:

```ts
interface ESLintLanguage {

    /**
     * Indicates how ESLint should read the file.
     */
    fileType: "text" | "binary";

    /**
     * Validates languageOptions for this language.
     */
    validateOptions(options: LanguageOptions): void;

    /**
     * Parses the given file input into its component parts.
     */
    parse(file: File, env: LanguageContext): ParseResult | Promise<ParseResult>;

    /**
     * Creates SourceCode object that ESLint uses to work with a file.
     */
    createSourceCode(input: ParseResult, env: LanguageContext): SourceCode | Promise<SourceCode>;

}

// Supplemental interfaces

interface LanguageOptions {
    // language-defined
}

interface File {
    path: string;
    body: string | ArrayBuffer
}

interface LanguageContext {
    options: LanguageOptions;
}

interface ParseResult {
    ast: ASTNode;
    body: string | ArrayBuffer;
}
```

At a high-level, ESLint uses the methods on a language object in the following way:

* During config validation, ESLint calls `validateOptions()` on the `languageOptions` specified in the config. It is the expectation that 
* When ESLint reads a file, it checks `fileType` to determine whether the language would prefer the file in text or binary format.
* After reading a file, ESLint passes the file information to `parse()` to create a raw parse result.
* The raw parse result is then passed into `createSourceCode()` to create the `SourceCode` object that ESLint will use to interact with the code.


### The `validateOptions()` Method

The intent of this method is to validate the `languageOptions` object as specified in a config. With flat config, all of the options in `languageOptions` are related to JavaScript and are currently validated [inside of `FlatConfigArray`](https://github.com/eslint/eslint/blob/f89403553b31d24f4fc841424cc7dcb8c3ef689f/lib/config/flat-config-array.js#L180). With this proposal, that validation will instead need to be based on the `language` specified in the config, which will require retrieving the associated language object and calling the `validateOptions()` method.

The `validateOptions()` method must throw an error if any part of the `languageOptions` object is invalid.

No specific `languageOptions` keys will be required for languages. The `parser` and `parserOptions` keys are unique to JavaScript and may not make sense for other languages.

#### The `parse()` Method

The `parse()` method receives the file information and information about the ESLint context in order to correctly parse the file. This is a raw parse, meaning that `parse()` should do nothing more than convert the file into an abstract format without doing any postprocessing. This is important so that ESLint can tell how long parsing takes vs. other operations. (One problem with the current `parseForESLint()` approach is that we lose visibility into the true parsing time.)

The `parse()` method is expected to return an object with at least `ast` and `body` properties; the object may also have other properties as necessary for the language to properly create a `SourceCode` object later. The return value from `parse()` is passed directly into `createSourceCode()`.

**Async:** For use with current ESLint, `parse()` must be synchronous; ESLint X will allow this method to be asynchronous.

#### The `createSourceCode()` Method

The `createSourceCode()` method is intended to create an instance of `SourceCode` suitable for ESLint and rule developers to use to work with the file. As such, it may perform postprocessing of a parse result in order to get it into a format that is easier to use. 

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
     * Traversal of AST.
     */
    traverse(): Iterable<TraversalStep>;

    /**
     * Determines if a node matches a given selector.
     */
    match(node: ASTNode, selector: string): boolean;
}

interface TraversalStep {
    type: "call" | "visit";
    target: string | ASTNode;
    args: object;
}
```

Other than these three members of the interface, languages define any additional methods or properties that they need to provide to rule developers. For instance, 
the JavaScript `SourceCode` object currently has methods allowing retrieval of tokens, comments, and lines. It is up to the individual language object implementations to determine what additional properties and methods may be required inside of rules.


### Core Changes

TODO

The functionality in `context.getScope()` will need to move to `SourceCode#getScope(node)`, where `node` must be passed in to retrieve the scope. We will need to map `context.getScope()` to call `SourceCode#getScope(node)` during the transition period.


## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

## Drawbacks

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think 
    about any opposing viewpoints that reviewers might bring up. 

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

## Frequently Asked Questions

**Why allow binary files?**

I think there may be a use in validating non-text files such as images, videos, and WebAssembly files. I may be wrong, but by allowing languages to specify that they'd prefer to receive the file in binary, it at least opens up the possibility that people may create plugins that can validate binary files in some way.

**Why not have `parse()` return a `SourceCode` instead of needing to call another method?**

It's important that we are able to determine the amount of time it takes to parse a file in order to optimize performance. Combining the parsing with creating a `SourceCode` object, which may require additional processing of the parse result, means obscuring this important number.

Additionally, `createSourceCode()` allows integrators to do their own parsing and create their own `SourceCode` instances without requiring the use of a specific `parse()` method.

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
