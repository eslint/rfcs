- Start Date: 2018-11-20
- RFC PR: (leave this empty, to be filled in later)
- Authors: Nicholas C. Zakas (@nzakas)

# Processors Improvements

## Summary

This proposal provides a way to explicitly define which processor(s) to use for different files inside of configuration. It also allows the chaining of multiple processors to fully process a file.

## Motivation

The current processor design is inverted from the way ESLint configuration works in all other cases. Plugins must specify which file extension(s) a processor should be applied to, and that setting is honored as soon as the plugin is loaded into ESLint. End-users do not have the option of disabling the processor or changing which files it should apply to.

Additionally, there are complex conditions such as Vue code inside of a Markdown file (see [#11035](https://github.com/eslint/eslint/issues/11035)) that are currently very difficult because it requires coordination between plugins directly rather than changes in configuration.

The goals of this proposal are:

1. Allow end-users to configure which processors should be used for which files.
2. Allow chaining of multiple processors for complex conditions, such as Vue code inside of a Markdown file.

## Detailed Design

Design Summary:

1. Require plugins to export named processors instead of/in addition to using file extensions.
1. Add a `processors` option to configuration.
1. Use `overrides` in configuration to target the files to use processors.
1. Apply multiple processors to target files using configuration.

Definitions:

* **Extension-named processor** is the original processor design, where a processor is named only by the file extension to which the process should apply.
* **Named processor** is introduced in this design, where a processor is given a name just like you would name a rule. The processor is not tied to a specific file extension by default.

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

In order to use a named processor, end-users would specify a `processors` option in their configuration. The option must be an array and must contain at least one string identifier. The string identifier must correspond to a named processor from a plugin and have the form *pluginName/processorName*, which would reference the `pluginName` processor in the `eslint-plugin-pluginName` plugin. For example: 

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

## Alternatives

@mysticatea submitted an alternative design proposal: [rfs/#1](https://github.com/eslint/rfcs/pulls/1)

## Related Discussions

* [Better support for multiple processors](https://github.com/eslint/eslint/issues/11035)
* [Alternate proposal](https://github.com/eslint/rfcs/pulls/1)
