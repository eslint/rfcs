- Repo: eslint/eslint
- Start Date: 2026-07-31
- RFC PR: https://github.com/eslint/rfcs/pull/151
- Authors: [Norbiros](https://github.com/Norbiros)

# Link Rule IDs to Their Documentation in the `stylish` Formatter

## Summary

The `stylish` formatter should render each rule ID as a terminal hyperlink (OSC 8) pointing at that rule's documentation URL, so users can open the docs for a reported problem directly from their terminal. The visible text of the output does not change, but it becomes clickable.

The rollout happens in two stages:

- In ESLint v10, hyperlinks are opt-in and are emitted only when the `FORCE_HYPERLINK` env variable is set.
- In ESLint v11, hyperlinks are emitted by default when ESLint is writing to a terminal, and users can opt out with `FORCE_HYPERLINK=0`.

## Motivation

Most popular ESLint plugins have good documentation, and rules already expose it through `meta.docs.url`. That metadata is unreachable from the default CLI output, though. Given this:

```
/home/user/project/src/index.js
  1:10  error  'foo' is defined but never used  no-unused-vars

✖ 1 problem (1 error, 0 warnings)
```

reading the rule's documentation means either switching to the `json-with-metadata` formatter, which prints a lot of unrelated information to surface one URL, searching for the rule manually online, which works poorly for plugin rules, or installing a third-party formatter.

Linking the rule ID reuses metadata ESLint already has, works the same for plugin rules as for core rules, and does not change what the output looks like. It is a common practice among other linters, and it is already what the VS Code ESLint extension does in the Problems panel.

## Detailed Design

### Terminology

A **terminal hyperlink** is an [OSC 8 escape sequence](https://gist.github.com/egmontkob/eb114294efbcd5adb1944c9f3cb5feda) that associates a URL with a run of text. Its form is:

```
ESC ] 8 ; ; <URL> BEL <text> ESC ] 8 ; ; BEL
```

or, as a JavaScript string:

```js
`\u001B]8;;${url}\u0007${text}\u001B]8;;\u0007`;
```

Terminals that support OSC 8 render `<text>` as a clickable link. Terminals that do not support OSC 8, but have a conformant OSC parser, consume and discard the sequence and render `<text>` unchanged.

### Which Text Is Linked

Only the rule ID column of the `stylish` formatter is linked. Message text, file paths, line/column positions, severity labels, and the summary line are unchanged.

The URL for a message with rule ID `ruleId` is `context.rulesMeta[ruleId].docs.url`. The `stylish` formatter already receives `rulesMeta` as part of its second `context` argument, so no new data needs to be plumbed through the CLI engine.

A rule ID remains plain text when it or its corresponding `rulesMeta` entry or `docs.url` is missing. This covers parsing errors, unmatched-pattern warnings, unresolved rules, and rules without documentation metadata.

The URL is used from the rule metadata without validation or rewriting, as it is in the `json-with-metadata` and `html` formatters. Control characters are removed as described under "Escaping".

### Detecting Support

Hyperlink support is decided once per formatter invocation. `FORCE_HYPERLINK` takes precedence: `0` and `false` disable links, while any other value enables them, including for piped or redirected output. The name and semantics follow `supports-hyperlinks`, `terminal-link`, and the established `FORCE_COLOR` convention.

Without the environment variable, hyperlinks are disabled in v10. In v11, the CLI enables them only when stdout is a TTY and `--output-file` is not set. It passes this decision to the formatter through an internal context field so the formatter does not mistake a file for terminal output. Programmatic formatter use remains default-off because ESLint cannot know where the returned string will be written; consumers can opt in with `FORCE_HYPERLINK`.

The v11 default is deliberately not tied to color support. Emitting a hyperlink is not a question of whether the user wants color, it is a question of whether the receiving terminal has a conformant OSC parser. Those are different properties: a user who sets `NO_COLOR` or passes `--no-color` is asking for output without color, not for output without links, and their terminal parses OSC 8 either way.

ESLint does not detect specific terminals. Supporting terminals render the link, while conformant non-supporting terminals discard the OSC sequence. Users with a non-conformant terminal can set `FORCE_HYPERLINK=0`.

### Escaping

The URL is embedded inside an escape sequence, so a URL containing a `BEL` (`\u0007`) or `ESC` (`\u001B`) character would terminate the sequence early and leak raw bytes into the terminal. Before building the sequence, ESLint removes all C0 control characters from the URL. In practice `meta.docs.url` values never contain them, so this is purely defensive.

`BEL` (`\u0007`) is used as the string terminator rather than `ESC \` (`ST`), because `BEL` is the more widely supported of the two terminators and, as described below, is recognized by Node.js's `util.stripVTControlCharacters()`.

### Column Alignment

`stylish` lays out each file's problems with ESLint's shared `text-table` implementation, passing a custom `stringLength` implementation that measures `util.stripVTControlCharacters(str).length`. Node.js recognizes OSC sequences terminated by `BEL`, so a hyperlink-wrapped rule ID measures the same width as the bare rule ID and the existing column alignment continues to work with no changes to the layout code. The implementation should include a test asserting both this assumption and that the rendered column positions are identical with and without hyperlinks enabled.

### Example

With `FORCE_HYPERLINK=1` (v10) or when writing to a terminal (v11), the rule ID cell for a `no-unused-vars` message is emitted as:

```
\u001B]8;;https://eslint.org/docs/latest/rules/no-unused-vars\u0007no-unused-vars\u001B]8;;\u0007
```

which renders in the terminal as the unchanged output with `no-unused-vars` clickable.

### Scope

This RFC covers only the `stylish` formatter, which is ESLint's default. The other built-in formatters are out of scope:

- `json` and `json-with-metadata` are machine-readable; `json-with-metadata` already exposes rule metadata.
- `html` already links rule IDs using anchors.

Nothing in this design prevents a follow-up from applying the same helper to another formatter, and the support-detection helper should be written so it can be reused.

### Implementation Approach

The formatter wraps rule IDs that have a `docs.url` using a small internal helper for support detection, escaping, and sequence construction. In v11, the CLI passes whether its actual output destination is a terminal through the formatter context.

Tests should cover the environment variable, missing metadata, URL escaping, column alignment, programmatic use, and CLI output to a TTY, pipe, redirect, and `--output-file`.

The v11 stage changes the value supplied by the CLI for terminal output, plus documentation and test updates.

A proof of concept of the core idea (without the staged rollout described here) exists in [eslint/eslint#21072](https://github.com/eslint/eslint/pull/21072).

## Documentation

This feature should be documented in:

- [Formatters](https://eslint.org/docs/latest/use/formatters/): note that `stylish` links rule IDs to their documentation, that this requires `FORCE_HYPERLINK` in v10, and that it is enabled by default in v11 for terminal output.
- [Command Line Interface](https://eslint.org/docs/latest/use/command-line-interface): document `FORCE_HYPERLINK`, including that it is independent of `--color`/`--no-color`.
- [Custom Rules](https://eslint.org/docs/latest/extend/custom-rules) and [Plugins](https://eslint.org/docs/latest/extend/plugins): mention that setting `meta.docs.url` now also makes the rule ID clickable in the default CLI output. This is a good incentive for plugin authors to populate the field.

The generated formatter examples in the documentation are produced without a TTY and without `FORCE_HYPERLINK`, so they are unaffected and require no regeneration.

## Drawbacks

- **Escape sequences in output that is not a terminal.** If a user forces hyperlinks on and then redirects output to a file or into a tool that does not strip ANSI sequences, the file contains escape sequences. This is already true of ESLint's color output and is mitigated by tying the v11 default to a TTY check.
- **Terminals and multiplexers with non-conformant OSC parsers.** A small number of environments (notably older `screen` configurations, and `tmux` before OSC 8 support landed) can mangle the sequence rather than discarding it. Affected users need to set `FORCE_HYPERLINK=0`. The staged rollout exists specifically so that this risk is only taken in a major release, after a full v10 cycle of opt-in usage has surfaced problem environments.

## Backwards Compatibility Analysis

**ESLint v10 (opt-in).** No behavior changes for any user who does not set `FORCE_HYPERLINK`. Default output is byte-for-byte identical to today, so snapshot tests, CI logs, and integrations are unaffected.

**ESLint v11 (default on).** Output changes only when ESLint writes to a TTY. The rendered text is unchanged, and nothing changes for output that is piped or redirected, which covers CI systems and integrations that consume `stylish` output programmatically. The populations that could be affected are:

- Users in a terminal with a non-conformant OSC parser, who see the raw sequence. They can set `FORCE_HYPERLINK=0`.
- Users who run ESLint in a TTY and assert against the captured output, for example in an interactive test harness. They can set `FORCE_HYPERLINK=0`.

## Alternatives

### Depend on `supports-hyperlinks`

The `supports-hyperlinks` package encodes per-terminal knowledge (`TERM_PROGRAM` and its version, VTE version, `WT_SESSION`, CI detection, and so on) to decide whether hyperlinks are safe. It was suggested during the issue discussion as a way to avoid rough heuristics.

This RFC rejects it because it adds a runtime dependency to ESLint in order to encode a table that goes stale in the direction of false negatives: terminals gain OSC 8 support and the table does not know about them, so users get no links and no way to understand why. Combined with the fact that non-supporting terminals discard the sequence, the detection problem is not worth a dependency. The explicit environment variable gives users deterministic control in both directions, which a heuristic cannot.

If the team disagrees, vendoring a minimal version of the detection logic (rather than depending on the package) would be the next-best option, and it can be added later behind the same predicate without any change to this design's user-facing behavior.

### Add a CLI flag instead of an environment variable

A `--hyperlinks`/`--no-hyperlinks` flag pair was considered. An environment variable is a better fit because the setting is a property of the user's terminal, not of an individual invocation: users want to set it once in their shell profile rather than on every command, and it needs to apply to ESLint invocations made by scripts and editors they do not control. It also matches the established convention of `FORCE_COLOR`/`NO_COLOR`. A CLI flag could be added later if there is demand, without conflicting with this design.

### Use a third-party formatter

`eslint-formatter-pretty` already offers hyperlinked rule IDs. It has a fraction of a percent of ESLint's download count, so in practice nearly all users never encounter this feature. Rule documentation URLs are first-class metadata that ESLint itself defines and that plugin authors are asked to provide; making use of it should not require discovering and installing a separate formatter.

## Open Questions

1. **Environment variable naming.** This RFC uses `FORCE_HYPERLINK`, matching `supports-hyperlinks` and `terminal-link`. The plural `FORCE_HYPERLINKS` was suggested in the issue discussion; does the team prefer that spelling instead?
2. **A `NO_HYPERLINK` counterpart.** `FORCE_HYPERLINK=0` already disables the feature. Should ESLint additionally honor a bare `NO_HYPERLINK` variable, mirroring the `NO_COLOR` convention?

## Help Needed

I can implement this RFC, both the v10 stage and the v11 stage.

## Related Discussions

- [eslint/eslint#21073](https://github.com/eslint/eslint/issues/21073) - Change Request: link rule IDs to docs URLs in stylish formatter
- [eslint/eslint#21072](https://github.com/eslint/eslint/pull/21072) - proof-of-concept implementation
- [OSC 8 hyperlink specification](https://gist.github.com/egmontkob/eb114294efbcd5adb1944c9f3cb5feda)
