- Repo: eslint/markdown
- Start Date: 2025-08-15
- RFC PR: https://github.com/eslint/rfcs/pull/138
- Authors: 루밀LuMir(lumirlumir)

# Support GFM Alert syntax parsing

## Summary

This RFC proposes adding support for GFM Alert syntax parsing to the `@eslint/markdown` package by implementing custom `micromark-extension-gfm-alert` and `mdast-util-gfm-alert` logic, and also includes the formal specification based on the [CommonMark spec](https://spec.commonmark.org/) and the [GitHub Flavored Markdown spec](https://github.github.com/gfm/).

## Motivation

The motivation for this RFC comes from [eslint/markdown#294](https://github.com/eslint/markdown/issues/294) and [eslint/markdown#449](https://github.com/eslint/markdown/issues/449).

Currently, GitHub supports a special syntax for alerts that isn't mentioned in the [GitHub Flavored Markdown spec](https://github.github.com/gfm/) and isn't parsed by the existing [`micromark-extension-gfm`](https://github.com/micromark/micromark-extension-gfm?tab=readme-ov-file#when-to-use-this) or [`mdast-util-gfm`](https://github.com/syntax-tree/mdast-util-gfm?tab=readme-ov-file#when-to-use-this) packages, which are used internally by `@eslint/markdown` to parse Markdown content and create AST nodes.

This RFC aims to address that gap by introducing the necessary parsing logic and formal specification for GFM Alert syntax.

GFM Alert syntax is a way to create attention-grabbing blocks in Markdown content. It uses a specific syntax to denote different types of alerts, such as notes, tips, warnings, and more. The basic structure of a GFM Alert is as follows:

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

TODO

### Type Interface

```ts
TODO
```

### Valid Syntax

#### Labels are case-insensitive

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

</details>

### Invalid Syntax

#### `!` prefix should not be surrounded by any spaces (U+0020) or tabs (U+0009).

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

#### Alert syntax should not be enclosed in HTML opening or closing tags.

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

1. It's not an official GitHub Flavored Markdown specification.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

Adding the `alert` option to the [Language Options](https://github.com/eslint/markdown?tab=readme-ov-file#language-options) field will not affect existing functionality in `@eslint/markdown@7`.

This option can be enabled by default when `@eslint/markdown@8` is released.

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
+           alert: true // Enable GFM Alert syntax parsing
        },
        rules: {
            "markdown/no-html": "error"
        }
    }
]);
```

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

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- The reasons why GitHub Alert syntax is not included in Mdast: 
    - https://github.com/syntax-tree/mdast-util-gfm/issues/2
    - https://github.com/orgs/syntax-tree/discussions/144

- `micromark-util` library reference:
    - TODO

- `mdast-util` library reference:
    - `mdast-util-math`: https://github.com/syntax-tree/mdast-util-math

- `rehype-github-alert` library reference:
    - https://github.com/rehypejs/rehype-github/tree/main/packages/alert#rehype-github-alert
