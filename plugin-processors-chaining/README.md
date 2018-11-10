# Making Plugin's Processor API chainable

## Summary

We have [Processor API in plugins](https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins).
Every processor is mapped to a file extension by one-to-one and we cannot combinate multiple processors per a file extension.

This proposal makes the processors being possible to combinate by that those preprocess can return the pair of code and filename.

If the returned filename requires another processor, ESLint applies the processor recursively.

```ts
interface Processor {
    readonly supportsAutofix?: boolean
    preprocess(code: string, filePath: string): string[] //→ ({ code: string, filePath: string } | string)[]
    postprocess(messages: Problem[][], filePath: string): Problem[]
}

interface Plugin {
    processors?: {
        [ext: string]: Processor
    }
    // ... omit others ...
}
```


## Motivation

There are some file types that can contain multiple code blocks in various types such as Markdown and HTML. For example:

    ```js
    console.log("hello")
    ```

    ```vue
    <template>
        <div>hello</div>
    </template>
    <script>
        export default {}
    </script>
    ```

    ```c
    #include <stdio>
    int main() {
        printf("hello");
        return 0;
    }
    ```

If some of the blocks require another processor (in this case `vue` block requires the processor of `eslint-plugin-vue`), we cannot work it easily.

This proposal lets us work it by allowing chaining multiple processors.


## Detailed Design

```ts
type VirtualFile = { filePath: string; code: string }
type Options = { [key: string]: any }

interface Processor {
    readonly supportsAutofix?: boolean
    preprocess(code: string, filePath: string, options: Options): (VirtualFile | string)[]
    postprocess(messages: Problem[][], filePath: string, options: Options): Problem[]
}
```

- This proposal changes the return type of `Processor#preprocess` to containing the filename of each block. And if the returned filename requires another processor, ESLint applies the processor recursively.

- This proposal adds the 3nd parameter into `preprocess`/`postprocess` to give `config.settings` object (note: this `config.settings` is the shared object that all rules can access) because plugins may need additional settings to determine the kind of code blocks it should return. I imaged:

    ```json
    {
        "extends": [
            "eslint:recommended",
            "plugin:markdown/recommended",
            "plugin:vue/recommended"
        ],
        // Whole this `settings` object will be given to processors.
        "settings": {
            "markdown": {
                "targetBlocks": ["js", "vue"]
            }
        }
    }
    ```

This feature will be implemented in `Linter#verify` method that is applying pre/post processes currently. Below is the implementation of [**verify**](https://github.com/eslint/eslint/blob/5da378ac922d732ca1765f08edee0face1b1b924/lib/linter.js#L1046) I imaged.

(I'm sorry, I couldn't describe it in English.)

```js
class Linter {
    verify(code, config, options) {
        if (!options || typeof options === "string") {
            return this._verifyWithoutProcessors(code, config, options);
        }

        const { filename, processors } = options;
        const settings = config.settings || {};

        //----------------------------------------------------------------------
        // The current behavior for backward compatibility.
        if (!filename || !processors) {
            const {
                preprocess = s => [s],
                postprocess = lodash.flatten
            } = options;

            return postprocess(
                preprocess(code, filename, settings).map(codeBlock =>
                    this._verifyWithoutProcessors(codeBlock, config, options)
                ),
                filename,
                settings
            );
        }

        //----------------------------------------------------------------------
        // New behavior.
        const ext = path.extname(filename);
        const processor = processors[ext];
        if (!processor) {
            return this._verifyWithoutProcessors(code, config, options);
        }

        const {
            preprocess = s => [s],
            postprocess = lodash.flatten,
            supportsAutofix = false
        } = processor;

        const messages = postprocess(
            preprocess(code, filename, settings).map(item => {
                if (typeof item === "string") {
                    return this._verifyWithoutProcessors(item, config, options);
                }
                const thisOptions = { ...options, filename: item.filePath }

                // To avoid infinite looping, skip processing if the file
                // extension was not changed.
                if (path.extname(item.filePath) === ext) {
                    return this._verifyWithoutProcessors(item.code, config, thisOptions);
                }

                // Verify this code block with processors.
                return this.verify(item.code, config, thisOptions);
            }),
            filename,
            settings
        );

        // Delete `fix` property if the processor doesn't support autofix
        // because CLIEngine cannot distinguish whether a certain message
        // supports autofix or not, especially when this `Linter#verify` call
        // is in recursive.
        // I think this is no problem because the `fix` properties are broken
        // if the processor doesn't support autofix.
        if (!supportsAutofix) {
            for (const message of messages) {
                delete message.fix;
            }
        }

        return messages;
    }
}
```


## How to Document

It updates the following two sections:

- https://eslint.org/docs/developer-guide/nodejs-api#linterverify
- https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins section.


## Drawbacks

- Plugin authors have to come up with a way to name arbitrary chunks of code that they extract from a given file.


## Backwards Compatibility Analysis

I believe that the compatibility problem is nothing.


## Alternatives

- [`processor` option in `.eslintrc`](https://github.com/eslint/eslint/issues/11035#issuecomment-435079819)


## Open Questions

## Help Needed
