- Start Date: 2018/11/10
- RFC PR: #1
- Authors: Toru Nagashima

# Making Plugin's Processor API chainable

## Summary

This proposal makes the processors being possible to combinate by that those preprocess can return the pair of code and filename. If the returned filename requires another processor, ESLint applies the processor recursively.

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

We have [Processor API in plugins](https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins).
Every processor is mapped to a file extension by one-to-one and we cannot combinate multiple processors per a file extension.

But there are some file types that can contain multiple code blocks in various types such as Markdown and HTML. For example:

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

If some of the blocks require another processor (in this case `vue` block requires the processor of `eslint-plugin-vue`), we cannot make it work easily.

This proposal lets us work it by allowing chaining multiple processors.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

```ts
type VirtualFile = { filePath: string; code: string }
type Options = { [key: string]: any }

interface Processor {
    readonly supportsAutofix?: boolean
    preprocess(code: string, filePath: string, options: Options): (VirtualFile | string)[]
    postprocess(messages: Problem[][], filePath: string, options: Options): Problem[]
}
```

- This proposal changes the return type of `Processor#preprocess` to containing the filename of each block. And if the returned filename requires another processor, ESLint applies the processor recursively except when the extension is the same as the original file.

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

## Documentation

It updates the following two sections:

- https://eslint.org/docs/developer-guide/nodejs-api#linterverify
- https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins

## Drawbacks

- The matter this feature is solving may be rare.
- This feature can increase the maintenance cost of plugins for the configuration that returns other blocks than JavaScript.

## Backwards Compatibility Analysis

- To delete `message.fix` from the lint results that a certain plugin which doesn't support autofix returned. But I think no problem because those `message.fix` is broken. And this will solve another problem on editor integrations (https://github.com/Microsoft/vscode-eslint/issues/185#issuecomment-267570350).

## Alternatives

### 1. [`processor` option in `.eslintrc`](https://github.com/eslint/eslint/issues/11035#issuecomment-435079819)

Specifying an arbitrary processor in `.eslintrc` will solve the problem. But the solution is on the end-user layer, they have to implement a processor, and a combinatorial explosion can be in a processor because the processor is mapped to a file extension by one-to-one. On the other hand, the recursive applying of processors solves the problem on the plugin layer, and end-users have less configuration.

<!--
## Open Questions

    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

<!--
## Help Needed

    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

<!--
## Frequently Asked Questions

    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

- https://github.com/eslint/eslint/issues/11035
