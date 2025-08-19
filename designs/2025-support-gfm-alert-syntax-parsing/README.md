- Repo: eslint/markdown
- Start Date: 2025-08-19
- RFC PR: https://github.com/eslint/rfcs/pull/138
- Authors: 루밀LuMir(lumirlumir)

# Support GFM alert syntax parsing

## Summary

This RFC proposes adding support for GFM alert syntax parsing to the [`@eslint/markdown`](https://github.com/eslint/markdown) package by implementing `micromark-extension-gfm-alert` and `mdast-util-gfm-alert` custom plugin logic, and also includes the formal specification based on the [CommonMark spec](https://spec.commonmark.org/) and the [GitHub Flavored Markdown spec](https://github.github.com/gfm/).

## Motivation

The motivation for this RFC comes from [eslint/markdown#294](https://github.com/eslint/markdown/issues/294) and [eslint/markdown#449](https://github.com/eslint/markdown/issues/449).

Currently, GitHub supports a special syntax for alerts that isn't mentioned in the [GitHub Flavored Markdown spec](https://github.github.com/gfm/) and isn't parsed by the existing [`micromark-extension-gfm`](https://github.com/micromark/micromark-extension-gfm?tab=readme-ov-file#when-to-use-this) and [`mdast-util-gfm`](https://github.com/syntax-tree/mdast-util-gfm?tab=readme-ov-file#when-to-use-this) packages, which are used internally by `@eslint/markdown` to parse Markdown content and create AST nodes.

Inconsistencies between GitHub's GFM alert syntax and existing Markdown parsers can cause false positives and false negatives in some cases, especially when writing rules that detect error-prone syntax and when the syntax overlaps with GitHub's alert syntax.

This RFC aims to address that gap by introducing the necessary parsing logic and formal specification for GFM alert syntax.

GFM alert syntax is a way to create attention-grabbing blocks in Markdown content. It uses a specific syntax to denote different types of alerts, such as notes, tips, warnings, and more. The basic structure of a GFM alert is as follows:

```md
> [!NOTE]
> Useful information that users should know, even when skimming content.

> [!TIP]
> Helpful advice for doing things better or more easily.

> [!IMPORTANT]
> Key information users need to know to achieve their goal.

> [!WARNING]
> Urgent info that needs immediate user attention to avoid problems.

> [!CAUTION]
> Advises about risks or negative outcomes of certain actions.
```

Which is being rendered on GitHub as:

> [!NOTE]
> Useful information that users should know, even when skimming content.

> [!TIP]
> Helpful advice for doing things better or more easily.

> [!IMPORTANT]
> Key information users need to know to achieve their goal.

> [!WARNING]
> Urgent info that needs immediate user attention to avoid problems.

> [!CAUTION]
> Advises about risks or negative outcomes of certain actions.


## Detailed Design

### Terms

Before explaining the design, it's important to define some key terms:

- ***closing angle bracket***: The `>` (U+003E) character that denotes a block quote in Markdown.
- ***opening square bracket***: The `[` (U+005B) character that starts the alert syntax.
- ***closing square bracket***: The `]` (U+005D) character that ends the alert syntax.
- ***exclamation mark***: The `!` (U+0021) character is used as a prefix to denote the alert type.
- ***space***: The ` ` (U+0020) character that separates the block quote marker from the alert syntax.
- ***tab***: The `\t` (U+0009) character that represents a tab.
- ***line ending***: According to the [CommonMark spec](https://spec.commonmark.org/0.31.2/#line-ending), A line ending is a line feed (U+000A), a carriage return (U+000D) not followed by a line feed, or a carriage return and a following line feed.
- ***label***: The text that appears inside the ***opening square bracket*** and ***closing square bracket*** and is prefixed by an ***exclamation mark***. It determines the style and semantics of the alert.

### Type Interface

Since GFM alert syntax is part of blockquote syntax from Markdown, `Alert` interface extends [`Blockquote`](https://github.com/syntax-tree/mdast?tab=readme-ov-file#blockquote) node type of Mdast.

```ts
import type { Blockquote } from "mdast";

interface Alert extends Blockquote {
    /**
     * Node type of mdast GFM alert.
     */
    type: "alert",

    /**
     * The label of the alert. It determines the style and semantics of the alert.
     * 
     * Its value should remain in its parsed form and should not be normalized.
     *
     * You can normalize its value by converting it to lowercase.
     * The normalized value should be one of the following: `"note"`, `"tip"`, `"important"`, `"warning"`, `"caution"`
     */
    label: string,
}
```

### Valid Syntax

Here are some examples of valid GFM alert syntax:

#### `NOTE`, `TIP`, `IMPORTANT`, `WARNING` and `CAUTION` are valid ***label***s

```md
> [!NOTE]
> Useful information that users should know, even when skimming content.
```

> [!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [!TIP]
> Helpful advice for doing things better or more easily.
```

> [!TIP]
> Helpful advice for doing things better or more easily.

---

```md
> [!IMPORTANT]
> Key information users need to know to achieve their goal.
```

> [!IMPORTANT]
> Key information users need to know to achieve their goal.

---

```md
> [!WARNING]
> Urgent info that needs immediate user attention to avoid problems.
```

> [!WARNING]
> Urgent info that needs immediate user attention to avoid problems.

---

```md
> [!CAUTION]
> Advises about risks or negative outcomes of certain actions.
```

> [!CAUTION]
> Advises about risks or negative outcomes of certain actions.

#### ***Label***s are case-insensitive

```md
> [!NOTE]
> Useful information that users should know, even when skimming content.
```

> [!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [!note]
> Useful information that users should know, even when skimming content.
```

> [!note]
> Useful information that users should know, even when skimming content.

---

```md
> [!Note]
> Useful information that users should know, even when skimming content.
```

> [!Note]
> Useful information that users should know, even when skimming content.

---

```md
> [!NoTe]
> Useful information that users should know, even when skimming content.
```

> [!NoTe]
> Useful information that users should know, even when skimming content.

---

```md
> [!nOtE]
> Useful information that users should know, even when skimming content.
```

> [!nOtE]
> Useful information that users should know, even when skimming content.

#### Upto 4 indented ***space*** is allowed before the ***opening square bracket***

```md
>[!NOTE]
> Useful information that users should know, even when skimming content.
```

>[!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [!NOTE]
> Useful information that users should know, even when skimming content.
```

> [!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
>  [!NOTE]
> Useful information that users should know, even when skimming content.
```

>  [!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
>   [!NOTE]
> Useful information that users should know, even when skimming content.
```

>   [!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
>    [!NOTE]
> Useful information that users should know, even when skimming content.
```

>    [!NOTE]
> Useful information that users should know, even when skimming content.

#### Upto 3 indented ***space*** is allowed before the ***closing angle bracket***

```md
> [!NOTE]
> Useful information that users should know, even when skimming content.
```

> [!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
 > [!NOTE]
 > Useful information that users should know, even when skimming content.
```

 > [!NOTE]
 > Useful information that users should know, even when skimming content.

---

```md
  > [!NOTE]
  > Useful information that users should know, even when skimming content.
```

  > [!NOTE]
  > Useful information that users should know, even when skimming content.

---

```md
   > [!NOTE]
   > Useful information that users should know, even when skimming content.
```

   > [!NOTE]
   > Useful information that users should know, even when skimming content.

### Invalid Syntax

Here are some examples of invalid GFM alert syntax:

#### ***exclamation mark*** should not be surrounded by any ***space***s or ***tab***s

```md
> [ !NOTE]
> Useful information that users should know, even when skimming content.
```

> [ !NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [    !NOTE]
> Useful information that users should know, even when skimming content.
```

> [    !NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [! NOTE]
> Useful information that users should know, even when skimming content.
```

> [! NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [!    NOTE]
> Useful information that users should know, even when skimming content.
```

> [!    NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [ ! NOTE]
> Useful information that users should know, even when skimming content.
```

> [ ! NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [    !    NOTE]
> Useful information that users should know, even when skimming content.
```

> [    !    NOTE]
> Useful information that users should know, even when skimming content.

#### ***Label***s should not end with ***space***s or ***tab***s

```md
> [!NOTE ]
> Useful information that users should know, even when skimming content.
```

> [!NOTE ]
> Useful information that users should know, even when skimming content.

---

```md
> [!NOTE    ]
> Useful information that users should know, even when skimming content.
```

> [!NOTE    ]
> Useful information that users should know, even when skimming content.

#### Only ***space***s and ***tab***s are allowed before the ***opening square bracket***

```md
> hi[!NOTE]
> Useful information that users should know, even when skimming content.
```

> hi[!NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> hi [!NOTE]
> Useful information that users should know, even when skimming content.
```

> hi [!NOTE]
> Useful information that users should know, even when skimming content.

#### Only ***space***s, ***tab***s, and ***line ending***s are allowed after the ***closing square bracket***

```md
> [!NOTE]hi
> Useful information that users should know, even when skimming content.
```

> [!NOTE]hi
> Useful information that users should know, even when skimming content.

---

```md
> [!NOTE] hi
> Useful information that users should know, even when skimming content.
```

> [!NOTE] hi
> Useful information that users should know, even when skimming content.

#### GFM alert syntax without content isn't working

```md
> [!NOTE]
```

> [!NOTE]

---

```md
> [!NOTE]
>
```

> [!NOTE]
>

#### Multi-line GFM alert syntax isn't working

```md
> [
> !NOTE]
> Useful information that users should know, even when skimming content.
```

> [
> !NOTE]
> Useful information that users should know, even when skimming content.

---

```md
> [!NOTE
> ]
> Useful information that users should know, even when skimming content.
```

> [!NOTE
> ]
> Useful information that users should know, even when skimming content.

#### Nested GFM alert syntax isn't working

```md
> > [!NOTE]
> > Useful information that users should know, even when skimming content.
```

> > [!NOTE]
> > Useful information that users should know, even when skimming content.

---

```md
- > [!NOTE]
- > Useful information that users should know, even when skimming content.
```

- > [!NOTE]
- > Useful information that users should know, even when skimming content.

#### GFM alert syntax should not be enclosed in HTML opening or closing tags

```md
<div>
> [!NOTE]
> Useful information that users should know, even when skimming content.
</div>
```

<div>
> [!NOTE]
> Useful information that users should know, even when skimming content.
</div>

---

```md
<div>

> [!NOTE]
> Useful information that users should know, even when skimming content.

</div>
```

<div>

> [!NOTE]
> Useful information that users should know, even when skimming content.

</div>

---

```md
<details>
<summary>Details</summary>
> [!NOTE]
> Useful information that users should know, even when skimming content.
</details>
```

<details>
<summary>Details</summary>
> [!NOTE]
> Useful information that users should know, even when skimming content.
</details>

---

```md
<details>
<summary>Details</summary>

> [!NOTE]
> Useful information that users should know, even when skimming content.

</details>
```

<details>
<summary>Details</summary>

> [!NOTE]
> Useful information that users should know, even when skimming content.

</details>

## Documentation

We need to update the [`README.md`](https://github.com/eslint/markdown?tab=readme-ov-file#eslint-markdown-language-plugin) for `@eslint/markdown` to include details about the new GFM alert syntax parsing feature. In particular, we should document the new `alert` option under the [Language Options](https://github.com/eslint/markdown?tab=readme-ov-file#language-options) section.  

## Drawbacks

1. This is not part of the official CommonMark or GitHub Flavored Markdown specifications.
2. It may not be widely adopted outside of GitHub, which could lead to inconsistencies in Markdown parsing across different platforms.

## Backwards Compatibility Analysis

Adding the `alert` option to the [Language Options](https://github.com/eslint/markdown?tab=readme-ov-file#language-options) field will not affect existing functionality in `@eslint/markdown@7`.

If desired, this option could be enabled by default in `@eslint/markdown@8`.

```diff
// eslint.config.js
import { defineConfig } from "eslint/config";
import markdown from "@eslint/markdown";

export default defineConfig([
    {
        files: ["**/*.md"],
        plugins: {
            markdown
        },
        language: "markdown/gfm",
        languageOptions: {
            frontmatter: "yaml", // Or pass `"toml"` or `"json"` to enable TOML or JSON front matter parsing.
+           alert: true // Enable GFM alert syntax parsing
        },
        rules: {
            "markdown/no-html": "error"
        }
    }
]);
```

## Alternatives

I cannot find any significant alternatives to this proposal. but I am open to suggestions.

### Pre-existing Implementations

There are existing implementations of GFM alert syntax parsing in other Markdown and HTML parsers:

- `rehype-github-alert`: https://github.com/rehypejs/rehype-github/tree/main/packages/alert#rehype-github-alert
- `markdown-it-github-alert`: https://github.com/antfu/markdown-it-github-alerts

## Open Questions

1. Will the `micromark-extension-gfm-alert` and `mdast-util-gfm-alert` plugins be published as separate packages?

## Help Needed

I've been reviewing the `micromark-extension` and `mdast-util` guides and implementations, but I'm not sure how best to implement the `micromark-extension-gfm-alert` and `mdast-util-gfm-alert` plugins.

I'd appreciate any guidance while I work on this, or help from anyone willing to take it on.

## Frequently Asked Questions

N/A

## Related Discussions

### The reasons why GitHub alert syntax is not included in Mdast

- https://github.com/syntax-tree/mdast-util-gfm/issues/2
- https://github.com/orgs/syntax-tree/discussions/144

### `micromark-extension` references

- Creating a micromark extension: https://github.com/micromark/micromark?tab=readme-ov-file#creating-a-micromark-extension

### `mdast-util` references
    
- `mdast-util-math`: https://github.com/syntax-tree/mdast-util-math
- `mdast-util-gfm-task-list-item`: https://github.com/syntax-tree/mdast-util-gfm-task-list-item