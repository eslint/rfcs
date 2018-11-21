- Start Date: 2018-11-20
- RFC PR: (leave this empty, to be filled in later)
- Authors: Nicholas C. Zakas (@nzakas), Toru Nagashima (@mysticatea)

# Processors Improvements

## Summary

This proposal provides a way to explicitly define which processor(s) to use for different files inside of configuration. It also allows the chaining of multiple processors to fully process a file.

## Motivation

The current processor design is inverted from the way ESLint configuration works in all other cases. Plugins must specify which file extension(s) a processor should be applied to, and that setting is honored as soon as the plugin is loaded into ESLint. End-users do not have the option of disabling the processor or changing which files it should apply to.

Additionally, there are complex conditions such as Vue code inside of a Markdown file (see [#11035](https://github.com/eslint/eslint/issues/11035)) that are currently very difficult because it requires coordination between plugins directly rather than changes in configuration.

Finally, there are some file types (such as Markdown) that can have code blocks using any number of programming languages. A single Markdown file might contain HTML code blocks, JavaScript code blocks, Vue code blocks, TypeScript code blocks, etc., and ESLint processors have no way to inform ESLint of these different types of code blocks being extracted. Processors today must simply filter out any code blocks that it believes ESLint can't handle.

The goals of this proposal are:

1. Allow end-users to configure which processors should be used for which files.
2. Allow chaining of multiple processors for complex conditions, such as Vue code inside of a Markdown file.
3. Allow processors to specify the file type of code blocks to change how ESLint behaves.

## Detailed Design

Design Summary:

1. Require plugins to export named processors instead of/in addition to using file extensions.
1. Add a `processors` option to configuration.
1. Use `overrides` in configuration to target the files to use processors.
1. Apply multiple processors to target files using configuration.
1. Allow processors to specify a filename for code blocks.

Definitions:

* **Extension-named processor** is the original processor design, where a processor is named only by the file extension to which the process should apply.
* **Named processor** is introduced in this design, where a processor is given a name just like you would name a rule. The processor is not tied to a specific file extension by default. A named processor name cannot begin with a `.` in order to avoid collisions with extension-named processors.
* **Parent file** is a file that is being processed to extract code blocks using processors.
* **Code block** is a snippet of code contained within a parent file. Any parent file may have contain multiple code blocks.

### Processor Plugin Changes

The first step is to change the format of processors in plugins. Currently, processors are exported by specifying the file extension to which the process should be applied. With this design, processors are exported by name, just like rules. For example:

```js
module.exports = {
    processors: {
        markdown: {
            preprocess() {},
            postprocess() {}
        }
    }
};
```

This plugin now exports a named `markdown` processor that can be referenced directly from configuration. Existing plugins would need to be changed, but can be done in a backwards-compatible way (see Backwards Compatibility section below).

### Add a `processors` Configuration Option

In order to use a named processor, end-users would specify a `processors` option in their configuration. The option must be an array and must contain at least one string identifier. The string identifier must correspond to a named processor from a plugin and have the form *pluginName/processorName*, which would reference the `processorName` processor in the `eslint-plugin-pluginName` plugin. For example: 

```js
module.exports = {
    plugins: ["markdown"],
    parserOptions: {
        ecmaVersion: 2018
    },
    processors: ["markdown/markdown"]
};
```

This configuration file will apply the named processor `markdown` from the `eslint-plugin-markdown` plugin to all files.

The `processors` option would be interpreted inside of `CLIEngine` to determine which processors to apply to files before linting should begin.

### Apply Processors Using `overrides`

Of course, in most cases the end-user will not want to apply a processor to all files. In that case, the `overrides` configuration option may be used to narrowly define to which files the processors should be applied. For example:

```js
module.exports = {
    plugins: ["markdown"],
    parserOptions: {
        ecmaVersion: 2018
    },
    overrides: [
        {
            files: ["*.md"],
            processors: ["markdown/markdown"]
        }
    ]
};
```

In this configuration, the named processor `markdown` from the `eslint-plugin-markdown` plugin is applied only to files with the extension `*.md`. This mimics the behavior of the extension-named processor that `eslint-plugin-markdown` currently exports.

The `CLIEngine` method `getConfigForFile()` would automatically get the correct value for `processors` when the configuration is calculated.

**Note:** End-users would still need to provide `--ext .md` on the command line for ESLint to automatically lint files ending with `.md`. No part of this proposal changes the behavior of `--ext`.

### Apply Multiple Processors

In special cases, end-users may want to apply multiple processors to files. This can be accomplished by listing multiple named processors using the `processors` configuration option. For example:

```js
module.exports = {
    plugins: ["markdown", "vue"],
    parserOptions: {
        ecmaVersion: 2018
    },
    overrides: [
        {
            files: ["*.md"],
            processors: ["markdown/markdown", "vue/vue"]
        }
    ]
};
```

This configuration applies two named processors to all files matching `*.md`. First, the `markdown` named processor from `eslint-plugin-markdown` is applied. The result of that preprocessing is then passed to the `vue` named processor from `eslint-plugin-vue`. The result of that preprocessing is then passed to ESLint for linting. When ESLint has completed linting, the reverse sequence occurs: the linting results are first passed to the `vue` named processor from `eslint-plugin-vue`, and once that postprocessing is complete, the result is then passed to the `markdown` named processor from  `eslint-plugin-markdown`.

End-users can also mix and match processors, such as in this example:

```js
module.exports = {
    plugins: ["markdown", "vue", "html"],
    parserOptions: {
        ecmaVersion: 2018
    },
    overrides: [
        {
            files: ["*.md"],
            processors: ["markdown/markdown", "vue/vue"]
        },
        {
            files: ["*.htm", "*.html"],
            processors: ["html/html", "vue/vue"]
        }
    ]
};
```

This configuration uses the same `vue` named processor from `eslint-plugin-vue` for both HTML and Markdown files, but only after an appropriate named processor is used to extract the Vue code.

**Note:** It would be update to the Vue processor to understand the difference between different types of code it may be passed.

### Virtual Filenames for Code Blocks

The `preprocess()` method inside of a processor can now return an array containing either strings (the current behavior) or an object with two properties:

* `filePath` - (string) a virtual file path to map the code block to
* `text` - (string) the source code text

For example:

```js
module.exports = {
    preprocess(text, filename) {
        
        const {vueText, jsText } = doSomething(text, filename);
        
        return [
            {
                filePath: filename + ".vue",
                text: vueText
            },
            {
                filePath: filename + ".js",
                text: jsText
            }
        ]
    }
};
```

This processor returns two code blocks: one that contains Vue code and is given a virtual filename ending with `.vue` and one that contains regular JavaScript code and is given a virtual filename ending with `.js`.

When a `preprocess()` method returns an object with a `filePath` property, `CLIEngine` will call `getConfigForFile()` on the `filePath` property to determine the correct configuration for the code block.

When a `preprocess()` method returns only a string, `CLIEngine` will interpret that as a JavaScript file and call `getConfigForFile()` using the parent file's filename to determine the correction configuration for the code block.

## Changes to `CLIEngine` and `Linter`

Currently, application of processors happens inside of the `Linter#verify()` and `Linter#verifyAndFix()` methods. In order to implement this proposal, application of processors must happen outside of `Linter` because configuration calculation happens outside of `Linter`.

The application of processors must move into `CLIEngine#executeOnText()` and `CLIEngine#executeOnFiles()` to work alongside with the current configuration calculation. That also means `CLIEngine` becomes responsible for recombining and postprocessing results from multiple `Linter#verify()` calls.

## Documentation

The following areas of the website will need to be updated:

* [Processors in Plugins](https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins)
* [Configuring ESLint](https://eslint.org/docs/user-guide/configuring#configuring-eslint)

Additionally, a blog post announcement should be made both to publicize the change and to encourage plugin owners to update their processors to be named instead of extension-named.

## Drawbacks

There are some drawbacks to this proposal, namely:

1. Existing plugins would have to be updated so they can be used in this way.
1. Existing configurations would have to be updated to make use of this design.

Both of these drawbacks create a barrier to adoption and require changes by the end-user in order to take advantage of this design.

## Backwards Compatibility Analysis

This proposal is 100% backwards compatible until we remove the old way of defining processors. Both named and extension-based processors can be define in the same plugin, such as:

```js
const processor = {
    preprocess() {},
    postprocess() {}
};

module.exports = {
    processors: {
        ".md": processor,
        markdown: processor
    }
};
```

We can continue to honor the extension-based processors by converting the definition into an entry in the referencing configuration's `overrides` section. So, for the existing extension-based Markdown processor, we would internally do this:

```js
module.exports = {
    plugins: ["markdown"],
    parserOptions: {
        ecmaVersion: 2018
    },
    overrides: [

        // this entry is added internally for backwards compatibility
        {
            files: ["*.md"],
            processors: ["markdown/markdown"]
        },

        // any already-existing overrides would go here
    ]
};
```

Making this extra `overrides` entry internally would eliminate a special case when determining which processors to use for which files.

At a future point in time, we can determine whether or not continue with this backwards-compatible behavior.

Additionally, the `preprocess` and `postprocess` options to `Linter#verify()` and `Linter#verifyAndFix()` will no longer be used by `CLIEngine`, but could remain in place in order to avoid a breaking change to the Node.js API.

## Alternatives

@mysticatea submitted an alternative design proposal: #1

The key differences between #1 and this proposal are:

1. #1 uses the existing `settings` configuration key to pass information about which processor(s) to use. This proposal uses a new `processors` configuration key to determine which processor(s) to use and the optional use of `overrides` to better define with processor(s) to use for which file types. We don't currently require anyone to use `settings` for any core feature in ESLint, so I'd rather not start doing that.
1. #1 relies on extension-named processors to work. This proposal requires the creation of named processors and the use of the `processors` configuration key to work.
1. #1 does not allow different configuration options based on virtual filenames for code blocks.

The similarities between #1 and this proposal are:

1. Code blocks must be given virtual filenames in order to determine which processors to use.
1. Multiple processes may be chained together in order to process code blocks multiple times.
1. Application of processors must happen outside of `Linter#verify()` and `Linter#verifyAndFix()`.

## Related Discussions

* [Better support for multiple processors](https://github.com/eslint/eslint/issues/11035)
* [Alternate proposal](https://github.com/eslint/rfcs/pulls/1)
