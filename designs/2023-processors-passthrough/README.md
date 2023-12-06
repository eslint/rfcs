- Repo: eslint/eslint
- Start Date: 2023/12/06
- RFC PR: (leave this empty, to be filled in later)
- Authors: Anthony Fu ([@antfu](https://github.com/antfu))

# Processors Passthrough

## Summary

Allow processors to create a passthrough file reusing the same filename as the original file.

## Motivation

Resolves [#14745](https://github.com/eslint/eslint/issues/14745)

Currently, processors can only create virtual files with a different filename. For example, the [`eslint-plugin-markdown`](https://github.com/eslint/eslint-plugin-markdown) creates virtual files for all the code blocks but swallows the original markdown file. In the vision of [Arbitrary Language Plugins](https://github.com/eslint/rfcs/blob/main/designs/2022-languages/README.md), it loses the ability for other plugins to process and lint the markdown file itself.

Another example is for framework components like Vue SFC, a single file that could contain multiple languages (HTML, CSS, JS). While [`eslint-plugin-vue`](https://github.com/vuejs/eslint-plugin-vue) handles both the HTML and JS parts directly on the `.vue` file, it does not yet support the CSS part. It would be nice if a processor could create a virtual CSS file to be processed by other plugins, while having the original `.vue` file processed by `eslint-plugin-vue`.

## Detailed Design

I came up with two possible solutions.

### Solution 1: Optional `filename` in `preprocess`

Currently, the `filename` option in the return of preprocess function is required, and [had a hard convention to add the `/{index}_{filename}` suffix](https://github.com/eslint/eslint/blob/fd0c60c3be1f213e5a6d69d8a3248e963619e155/lib/linter/linter.js#L1524).

We could make it optional, and if not provided, the original filename is used. This would allow processors to create a passthrough file reusing the same filename as the original file

```js
const processor = {
  preprocess(text, filename) {
    return [
      // without `filename` option, the original filename is used, behaving like "passthrough"
      { text: text }, // <--
      
      // virtual files as before
      { text: codeSnippet, filename: "1.js" },
      // ...
    ]
  },

  postprocess(messages, filename) {
    return [
      // forward all messages as they didn't change
      ...messages[0],
      // remapping the positions of messages from the rest virtual files
      ...messages
        .slice(1)
        .flatMap(m => resolveMapping(m, filename))
    ]
  },
}
```

### Solution 2: New `absolute` option in `preprocess`

We might also give full control of the virtual file name to the processor, by adding an `absolute` option to the return of `preprocess`. If `absolute` is `true`, the filename is used as-is, otherwise, the filename is resolved relative to the original filename.

```js
import { join } from 'path'

const processor = {
  preprocess(text, filename) {
    return [
      { text: text, filename: filename, absolute: true }, // <--
      
      // processors can also create virtual files and construct the filename their own
      { text: codeSnippet, filename: join(filename, "some/context/1.js"), absolute: true },
      // ...
    ]
  },

  postprocess(messages, filename) {
    // ...
  },
}
```

It would be more powerful and more explicit.

### Plugins Adoptions

For `eslint-plugin-markdown`, it could have a setting option to allow passthrough the original markdown file. So that other plugins (e.g. `eslint-plugin-markdownlint`) could lint the markdown content.

For `eslint-plugin-vue` would have an optional processor package (for example, `eslint-processor-vue-blocks`), which creates virtual files for each blocks like `<style>` `<i18n>` while keeping the original `.vue` file for `eslint-plugin-vue` to process.

## Documentation

We could update [this document page](https://eslint.org/docs/latest/extend/custom-processors) to note this ability.

## Drawbacks

If we go with solution 1, we only need to be careful that the processor does not get called recursively and create an infinite loop. The current codebase seems to already handling this well.

If we go with solution 2, it might create some confusion or conflicts with other existing files if the processor emits arbitrary filenames. For conflicts with existing files, we could throw hard errors to prevent them.

## Backwards Compatibility Analysis

It would be an optional addition to the existing API, so it should be backward compatible.

## Alternatives

## Open Questions

- Which solution should we go with? Or is there a better solution?
- Should we allow array to the `processor` option to support multiple processors on the same file (as proposed in [#14745](https://github.com/eslint/eslint/issues/14745)), so processors can be composed? It could be another RFC though.

## Help Needed

## Frequently Asked Questions

## Related Discussions

- https://github.com/eslint/eslint/issues/14745
