- Repo: eslint/eslint
- Start Date: 2023-02-06
- RFC PR: (fill in later)
- Authors: Nicholas C. Zakas

# ESLint Prelint Plugins

## Summary

This proposal provides a way for plugin authors to define functionality that should be performed *before* any rules are applied against the AST.

This RFC depends on the [Language Plugins RFC](https://github.com/eslint/rfcs/pull/99).

## Motivation

For many years, plugin authors have wanted ways to hook into ESLint outside of the context of rules. The most glaring example was the addition of [`context.markVariableAsUsed()`](https://github.com/eslint/eslint/blob/17b65ad10d653bb05077f21d8b1f79bee96e38d8/lib/linter/linter.js#L893) to allow React developers to mark the `React` global as used. Because rules were the only way to do that, we agreed to not check unused variables in `no-unused-vars` until `Program:exit` so we could give them the opportunity to call `context.markVariableAsUsed()` before we checked.

Another use case is in processors, which always parse the input text into some AST, then return code blocks, often throwing away the AST from the initial parse. In a world where ESLint supports linting of multiple different languages, we would want to keep [that AST around to lint as well as its blocks](https://github.com/eslint/eslint/issues/14745).

This also came up in the [language plugins RFC](https://github.com/eslint/rfcs/pull/99#issuecomment-1414397526) as something plugin authors want outside of the normal way that processors work.

By creating functionality that can run before rules, we can potentially solve for more than just these two use cases and open up other possibilities.

## Detailed Design

This proposal consists of the following changes:

1. Allows plugins to publish prelints in addition to rules, processors, etc.
1. Allows end users to select which prelints to use in their config files.
1. Defines an API for defining prelints.
1. Changes the ESLint core to run prelints before rules.

**Prerequisites:** This proposal requires the flat config system and cannot be used with the eslintrc config system.

**Note:** This design is intended to work both in the current version of ESLint and also in the "rewrite" version, which I'll refer to as ESLint X to distinguish between the two. Not all features of this proposal will be available in both versions. 

### Prelints in plugins and configs

Prelints can be defined in plugins by using the `prelints` key and giving the prelint a name. This allows multiple prelints to be contained in a single plugin if desired. For example:

```js
// inside eslint-plugin-markdown
export default {

    prelints: {
        js: {
            // prelint object
        },
        css: {
            // prelint object
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

Here, `js` and `css` are prelints defined by the plugin. End users can specify which prelint to use via the `prelints` key in a config object, such as:

```js
// inside eslint.config.js
import markdown from "eslint-plugin-markdown";

export default [
    {
        plugins: {
            markdown
        },
        files: ["**/*.md"],
        language: "markdown/markdown",
        prelints: {
            "markdown/js": true,
            "markdown/css": true
        }
    }
];
```

Here, the `"markdown/js"` prelint is a prelint that will extract JavaScript code from Markdown files and `"markdown/css"` is a prelint that will extract CSS code from Markdown files.

Prelints can be configured similar to rules, except prelints will accept any truthy value as "on" and any falsy value as "off". There is no need for severities because prelints do not report violations. If a prelint supports options, it can be passed in config as well:

```js
// inside eslint.config.js
import markdown from "eslint-plugin-markdown";

export default [
    {
        plugins: {
            markdown
        },
        files: ["**/*.md"],
        language: "markdown/markdown",
        prelints: {

            // prelint with options
            "markdown/js": {
                codeBlockTags: ["js", "javascript", "ecmascript"]
            },
            "markdown/css": true
        }
    }
];
```

Here, the `"markdown/js"` prelint is being passed options it can use to modify its behavior.

### Prelint Definition Object

Each prelint definition object must implement the following interface:

```ts
interface Prelint {

    /**
     * Space for any needed meta information.
     */
    meta: {
        docs: {
            description: string;
        }
    }

    /**
     * Creates the prelint.
     */
    create(context: PrelintContext): Object;

}

// Supplemental interfaces

interface PrelintContext {
    /**
     * Options passed in config.
     */
    options: Array<any> | Object;

    /**
     * The SourceCode object to work on.
     */
    sourceCode: sourceCode;

    /**
     * Shared settings from config.
     */
    settings: object;

    /**
     * ID of the prelint (i.e., "markdown/js")
     */
    id: string;

    /**
     * Current working directory.
     */
    cwd: string;

    /**
     * Filename that ESLint is working on, including code blocks.
     */
    filename: string;

    /**
     * Physical filename ESLint is working on, excluding code blocks.
     */
    physicalFilename: string;


    /**
     * Creates a text fragment from a subset of text in the current file.
     */
    createTextFragment(file: TextFragment): void;

    /**
     * Creates a binary fragment from a subset of bytes in the current file.
     */
    createBinaryFragment(file: BinaryFragment): void;
}

// Supplemental interfaces

interface TextFragment {

    /**
     * The name of the file without a path.
     */
    filename: string;

    /**
     * The slice of text representing the fragment from the parent file.
     */
    range: [number, number];

    /**
     * The number to add to any line-based violations. 0 if undefined.
     */
    lineStart?: number;

    /**
     * The number to add to any violations that occur in the first line of
     * the fragment. 0 if undefined.
     */
    columnStart?: number;

    /**
     * The number to add to the column number of each violation in the body,
     * and the number of characters to strip off the text slice before linting.
     * 0 if undefined.
     */
    indentOffset?: number;
}

interface BinaryFragment {

    /**
     * The name of the file without a path.
     */
    filename: string;

    /**
     * The slice of bytes representing the fragment from the parent file.
     */
    range: [number, number];

    /**
     * The number to add to violation offset locations. 0 if undefined.
     */
    byteStart?: number;

    /**
     * The number to add to the offset location of each violation in the body.
     * 0 if undefined.
     */
    byteOffset?: number;
}

interface SourceCode {
    // language-defined
}
```

Each prelint follows the same basic structure of as a rule, only it cannot report violations and instead can only inspect the AST, use `SourceCode` to make any necessary changes, and create fragments as necessary. Otherwise, it looks the same as a rule, such as:

```js
export default {
    meta: {
        description: "My prelint"
    },
    create(context) {

        const sourceCode = context.sourceCode;

        // work with sourceCode here

        return {
            Identifier() {
                // visitor method
            }
        }
    }
};
```

Because the prelints follow the same format as rules, we can use the same type of traversal.

#### The `PrelintContext#createTextFragment()` method

The design of `TextFragment` is intended to automate the mapping of line/column reporting that currently has to be done manually inside of a processor in the `postprocess()` step. By specifying the line and column location of the fragment inside its parent, we can calculate that automatically.

During prelint traversal, the core will gather up all of the created fragments  and then queue them up for linting in addition to the original (parent) file. To each rule, text fragments would look the same as if they came from a processor.

As an example, suppose we have an HTML language plugin that can parse the following file:

```html
<!doctype html>
<html>
<head>
    <title>Example</title>
    <script>
        function sayHi() {
            console.log("Hi");
        }
    </script>
    <style>
        body {
            background: blue;
        }
    </style>
</head>
</html>
```

The HTML language plugin will first parse the HTML into an AST. Inside of that AST, there will be a `Script` node representing the JavaScript portion of the file and a `Style` node representing the CSS portion. We could create a text fragment for each use prelints. Here is an example that extracts the JavaScript into a text fragment:

```js
export default {
    meta: {
        description: "Extracts JS from HTML."
    },
    create(context) {

        const sourceCode = context.sourceCode;

        return {
            Script(node) {

                // imagine the content property exists
                const textNode = node.content;

                context.createTextFragment({
                    filename: "0.js",
                    range: [70, 137]
                    lineStart: 6,
                    columnStart: 0,
                    indentOffset: 8                    
                });
            }
        }
    }
};
```

The `indentOffset` property is used to strip any indents off of the text slice before passing back to ESLint. In fix mode, once the fixing is complete, ESLint will then use `indentOffset` to re-establish the original indentation. Essentially, ESLint will capture the first `indentOffset` number of characters on the first line and then reapply that to the fixed text. 

Rules and languages can be targeted at the text fragments in a config file by treating them as if they were parts returned by a processor, such as:

```js
import js from "@eslint/js";
import html from "eslint-plugin-html";
import css from "eslint-plugin-css";

export default [

    {
        plugins: {
            js,
            css,
            html
        }
    },

    // target the HTML files
    {
        files: ["**/*.html"],
        language: "html/html",
        prelints: {
            "html/js": true,
            "html/css": true
        }
        rules: {
            "html/doctype": "error"
        }
    },

    // target the JS files -- include fragments
    {
        files: ["**/*.js"],
        language: "js/js",
        rules: {
            "js/semi": "error"
        }
    },

    // target the CSS files -- includes fragments
    {
        files: ["**/*.css"],
        language: "css/css",
        rules: {
            "css/number-units": "error"
        }
    },

    // target only fragment JS files
    {
        files: ["**/*.html/*.js"],
        language: "js/js",
        rules: {
            "js/semi": "warn"
        }
    },

    // target only fragment CSS files
    {
        files: ["**/*.html/*.css"],
        language: "css/css",
        rules: {
            "css/number-units": "warn"
        }
    }
];
```

### Core Changes

In order to make all of this work, we'll need to make the following changes in the core:

1. `linter.js` will need to be updated to treat fragments the same as processor code blocks (recursively linting) and to do a first traversal of the file using prelints.
1. `FlatConfigArray` will need to be updated to define the `prelints` key.
1. Create a new `PrelintContext` object.
1. `RuleTester` will need to be updated to support passing in `prelints`.

## Documentation

Most existing end-user facing documentation will remain unchanged.

Plugin developer documentation will need to be updated:

* A new page on writing prelints for ESLint will need to be written.

## Drawbacks

1. **Performance.** There will likely be a small performance penalty when using prelints due to the extra traversal. It's possible this penalty will match the penalty of using a processor today, or it may be slightly more. We'll need to do some performance testing to be sure.
1. **eslintrc Compatibility.** Unfortunately, there's no easy way to make this functionality available to eslintrc config users. When implemented, eslintrc users will be left without the ability to use other languages.

## Backwards Compatibility Analysis

This proposal is 100% backwards compatible.

This proposal does not require any changes from end-users.

## Alternatives

1. In the original version of the [language plugins RFC](https://github.com/eslint/rfcs/pull/99), I proposed adding a `getVirtualFiles()` method to the `SourceCode` interface that would allow language parsers to specify virtual files as part of their postprocessing. However, this would prevent end users from being able to configure which virtual files they care about.
1. We could choose to iterate on processors to gain similar functionality.

## Open Questions

### Can't we come up with a better name that "prelint"?

It's possible...this name is kind of gross. I went through a lot of iterations on the name before deciding on this but I'm definitely open to other ideas.

## Help Needed

N/A

## Frequently Asked Questions

### How will autofix work?

Because we have the mapping of physical file to fragments defined via lines, columns, and indent offsets, we should be able to apply autofixes to fragments and then merge them into the physical file automatically. This should eliminate the need for postprocessing functions like we have in processors.

### How will this coexist with processors?

Ideally, we would deprecate processors in favor of prelints, but we don't have to make that decision right now.

## Related Discussions

* [Language Plugins RFC](https://github.com/eslint/rfcs/pull/99)
* [#14745 Allow processor API to be configurable and to formally be able to lint both a file and its blocks](https://github.com/eslint/eslint/issues/14745)
