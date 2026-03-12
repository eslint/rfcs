- Repo: eslint/yaml
- Start Date: 2026-02-23
- RFC PR: https://github.com/eslint/rfcs/pull/144
- Authors: André Ahlert (@andreahlert)

# Official @eslint/yaml Language Plugin

## Summary

This RFC proposes creating an official ESLint language plugin for YAML, published as `@eslint/yaml`, under the ESLint organization. The plugin would allow users to lint YAML files (e.g. GitHub Actions workflows, Kubernetes manifests, Docker Compose, CI configs) using ESLint's flat config and the language plugin API established in [RFC #99 (2022-languages)](https://github.com/eslint/rfcs/blob/main/designs/2022-languages/README.md). It follows the same patterns as other official language plugins: `@eslint/json`, `@eslint/markdown`, `@eslint/css`, and `@html-eslint/eslint-plugin`.

## Motivation

- **Demand:** YAML was one of the most requested languages alongside CSS (which now has official `@eslint/css` support). Many developers work with YAML daily for GitHub Actions, Kubernetes, Docker Compose, Ansible, Helm charts, and other tooling.
- **Consistency:** ESLint has expanded into JSON, Markdown, and CSS with official plugins. An official YAML plugin completes the story for common config/data formats and establishes a reference implementation maintained by the team.
- **Existing ecosystem:** Community plugins such as [eslint-plugin-yml](https://github.com/ota-meshi/eslint-plugin-yml) (by @ota-meshi, v3 supports the language plugin API) and [eslint-yaml](https://github.com/43081j/eslint-yaml) already exist and work well. The TSC has expressed interest in a team-maintained plugin (see [Discussion #20482](https://github.com/eslint/eslint/discussions/20482)) and @nzakas has welcomed an RFC to kickstart this effort. An official plugin would provide long-term maintenance, alignment with ESLint's release cycle, and a single recommended solution for YAML linting.
- **Reference implementation:** A team-maintained plugin serves as a reference for the language plugin API applied to YAML and encourages best practices for parsing, rules, and editor integration.

## Detailed Design

### Scope

- **Package name:** `@eslint/yaml`
- **Repository:** New repository at [github.com/eslint/yaml](https://github.com/eslint/yaml), following the pattern of `eslint/json`, `eslint/markdown`, `eslint/css`.
- **Module format:** ESM-only, in line with the TSC decision that new packages should be ESM-only (see [2026-01-08 TSC meeting](https://github.com/eslint/tsc-meetings/blob/main/notes/2026/2026-01-08.md)).
- **Compatibility:** ESLint v9.15.0+ (flat config only). No eslintrc support.
- **TypeScript types:** Built-in type definitions shipped with the package from the initial release, following the direction the team has taken with `espree`, `eslint-scope`, and other packages.

### Plugin export shape

The plugin follows the same structure as `@eslint/json`:

```js
const plugin = {
  meta: {
    name: "@eslint/yaml",
    version: "0.1.0", // x-release-please-version
  },
  languages: {
    yaml: new YAMLLanguage({ mode: "yaml" }),     // YAML 1.2 (default)
    yaml11: new YAMLLanguage({ mode: "yaml11" }), // YAML 1.1 compatibility
  },
  rules,
  configs: {
    recommended: {
      name: "@eslint/yaml/recommended",
      plugins: {},
      rules: recommendedRules,
    },
  },
};
```

Key points:
- **`meta`** with `name` and `version` (aligned with all official plugins).
- **Two language modes:**
  - `yaml/yaml` — YAML 1.2 (the current spec; strict mode, `yes`/`no` are strings, not booleans).
  - `yaml/yaml11` — YAML 1.1 compatibility (legacy; `yes`/`no`/`on`/`off` are booleans, octal `0777` syntax, etc.). Many real-world files (older Docker Compose, Ansible) still rely on 1.1 behavior.
- **`configs.recommended`** — a recommended config that enables sensible default rules (e.g. `no-duplicate-keys`), so users can adopt with minimal configuration.

### Language plugin contract

The language definition implements the interface from [RFC #99](https://github.com/eslint/rfcs/blob/main/designs/2022-languages/README.md#language-definition-object):

| Property | Value | Notes |
|---|---|---|
| `fileType` | `"text"` | YAML is a text format |
| `lineStart` | `1` | 1-based line numbers |
| `columnStart` | `1` | 1-based columns |
| `nodeTypeKey` | `"type"` | Node type property in the AST |
| `visitorKeys` | `{ YAMLStream: [...], YAMLDocument: [...], YAMLMapping: [...], ... }` | Traversal keys for each node type |
| `validateOptions(options)` | Validates `languageOptions` | e.g. reject unknown keys |
| `matchesSelectorClass(className, node, ancestry)` | Optional selector shortcuts | e.g. `"value"` matches all value nodes |
| `parse(file, context)` | Parse YAML text → `ParseResult` | Never throws; parse errors returned in result |
| `createSourceCode(file, input, context)` | Build `SourceCode` from parse result | Used by ESLint for reporting and fixes |

### AST node types

The AST should represent the YAML structure with at least the following node types:

```
YAMLStream          — root node representing the entire file (contains one or more documents)
  YAMLDocument      — a single document (files may contain multiple documents separated by ---)
    YAMLMapping     — a mapping (object / key-value collection)
      YAMLPair      — a single key-value pair within a mapping
    YAMLSequence    — a sequence (array / list)
    YAMLScalar      — a scalar value (string, number, boolean, null)
    YAMLAlias       — an alias reference (*anchor); the anchor name is stored as a property
    YAMLComment     — a comment (# ...)
    YAMLDirective   — a YAML directive (%YAML 1.2, %TAG, etc.)
```

Note: anchors (`&name`) are **not** separate node types. An anchor is a property on the node it is attached to (e.g. `YAMLScalar.anchor = "name"`), following how YAML parsers represent them. Only alias references (`*name`) are distinct `YAMLAlias` nodes.

The `YAMLStream` root node is necessary because YAML allows multiple documents in a single file. Even for single-document files, the stream node provides a consistent root for traversal.

Each node must include:
- `type` — the node type string (e.g. `"YAMLMapping"`)
- `range` — `[startOffset, endOffset]` in the source text
- `loc` — `{ start: { line, column }, end: { line, column } }` for reporting

The exact AST shape will depend on the chosen parser, but should be documented in the plugin's README and type definitions.

### Parser choice

The RFC does not prescribe a specific parser. Candidates include:

| Parser | YAML version | Comments | Location info | Notes |
|---|---|---|---|---|
| [yaml (eemeli/yaml)](https://github.com/eemeli/yaml) | 1.1 + 1.2 | Yes (CST) | Yes | Mature, actively maintained, CST → AST |
| [yaml-eslint-parser](https://github.com/ota-meshi/yaml-eslint-parser) | 1.2 | Yes | Yes | ESTree-compatible AST on top of eemeli/yaml; used by eslint-plugin-yml |

The parser must:
1. Produce a traversable AST with location info for every node.
2. Support comments (required for disable directives and style rules).
3. Not throw on parse errors (return them as part of `ParseResult`).
4. Be actively maintained and have a permissive license (MIT or Apache 2.0).

The final choice will be made during implementation and documented in the repo.

### Configuration comments and disable directives

YAML supports line comments with `#`. The plugin will support ESLint configuration comments and disable directives:

```yaml
# eslint yaml/no-duplicate-keys: "error"

name: CI
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "hello" # eslint-disable-line yaml/some-rule -- reason here
```

Both `eslint-disable`, `eslint-disable-next-line`, `eslint-enable`, and inline rule configuration will be supported, following the same patterns as `@eslint/json` in JSONC/JSON5 mode.

### Rules (initial set)

Following the direction of the project — which is [moving away from formatting rules](https://github.com/eslint/eslint/issues/20097) and toward correctness / best-practice rules — the initial set focuses on catching real problems:

**Recommended (enabled in `configs.recommended`):**

| Rule | Description | Fixable |
|---|---|---|
| `no-duplicate-keys` | Disallow duplicate keys in mappings | No |
| `no-empty-keys` | Disallow empty string keys | No |
| `no-empty-values` | Disallow empty mapping values | No |
| `no-empty-sequence-entry` | Disallow empty sequence entries | No |
| `no-unsafe-values` | Disallow unquoted values that would be interpreted differently between YAML 1.1 and 1.2 (e.g. `yes`, `no`, `on`, `off` are booleans in 1.1 but plain strings in 1.2) | No |

**Additional (not in recommended, opt-in):**

| Rule | Description | Fixable |
|---|---|---|
| `sort-keys` | Require mapping keys to be sorted | Yes |
| `no-duplicate-anchors` | Disallow duplicate anchor names in a document | No |
| `require-document-start` | Require explicit `---` document start marker | Yes |

Formatting-related rules (indentation, quoting style, trailing newlines) are explicitly **out of scope** for v1, consistent with the project's direction toward delegating formatting to tools like Prettier.

Additional correctness rules can be proposed and added in subsequent minor versions.

### Configuration example

```js
// eslint.config.js
import { defineConfig } from "eslint/config";
import yaml from "@eslint/yaml";

export default defineConfig([
  {
    plugins: {
      yaml,
    },
  },

  // Lint YAML 1.2 files
  {
    files: ["**/*.yml", "**/*.yaml"],
    language: "yaml/yaml",
    extends: ["yaml/recommended"],
  },

  // Lint legacy YAML 1.1 files (optional, e.g. older Ansible playbooks)
  {
    files: ["**/ansible/**/*.yml"],
    language: "yaml/yaml11",
    extends: ["yaml/recommended"],
  },
]);
```

### Collaboration with existing plugins

- **eslint-plugin-yml (by @ota-meshi):** v3 already works as a language plugin. The official plugin may take inspiration from its rule set and AST design. Any code in the new repo will be written from scratch under the eslint org; collaboration and ideas from the community are welcome.
- **eslint-yaml (by @43081j):** Another community plugin for reference.
- No obligation to fork or replace community plugins. The goal is to provide an official, team-maintained option. Community plugins remain valid choices.

### Repository and release process

- New repo at `github.com/eslint/yaml`.
- Release process aligned with [ESLint's release process](https://eslint.org/docs/latest/contribute/release-process): semver, release-please, provenance-signed publishes.
- CI: lint, test, type-check, and integration tests against ESLint main.

## Documentation

- **README:** Installation, flat config usage with `defineConfig`, list of rules with recommended markers, language modes, editor setup.
- **Rule docs:** One doc per rule with description, options, examples (valid/invalid), and fixable status.
- **Editor setup:** Instructions for VS Code (`eslint.validate` with `yaml`) and JetBrains (ESLint scope including `*.yml`, `*.yaml`).
- **eslint.org:** Link from the "language plugins" or "integrations" section to `@eslint/yaml` once published.

## Drawbacks

- **Maintenance burden:** Another official package for the team to maintain. The TSC has limited bandwidth and has recently implemented [spending reductions](https://github.com/eslint/tsc-meetings/blob/main/notes/2026/2026-01-08.md) due to budget constraints. **Mitigation:** The RFC author volunteers as primary champion for implementation and ongoing maintenance, starting with a minimal rule set. The goal is to reduce, not increase, the TSC's workload.
- **Overlap with community plugins:** Some users are happy with eslint-plugin-yml; an official plugin could be perceived as fragmenting the ecosystem. **Mitigation:** Position as the recommended official option; document a clear migration path; avoid breaking existing community plugins.
- **YAML complexity:** YAML has many edge cases (anchors, aliases, multi-document, custom tags, merge keys). **Mitigation:** Scope v1 to the most common use cases (GitHub Actions, K8s manifests, Docker Compose) and clearly document what is and isn't supported. Advanced features can be added incrementally.

## Backwards Compatibility Analysis

- **ESLint core:** No changes to `eslint` or `@eslint/js`. This is a new, standalone package.
- **For users of community YAML plugins:** Fully opt-in. Users of eslint-plugin-yml or eslint-yaml can continue using them. Migration to `@eslint/yaml` would be config-based; a migration guide with rule name mapping (if applicable) will be provided.

## Alternatives

1. **Rely only on community plugins.** Community plugins already cover YAML linting well. However, the TSC has expressed interest in a team-maintained plugin to serve as a reference implementation aligned with ESLint's release cycle ([Discussion #20482](https://github.com/eslint/eslint/discussions/20482), [2024-09-05 TSC meeting](https://github.com/eslint/tsc-meetings/blob/main/notes/2024/2024-09-05.md#do-we-want-to-create-official-language-plugins-for-yaml-and-css)).
2. **Adopt or fork eslint-plugin-yml under the eslint org.** This would require agreement with @ota-meshi and a potentially different maintenance model. This RFC proposes a new repo; collaboration with existing plugins is left open and encouraged.
3. **Only document how to use community YAML plugins on eslint.org.** This satisfies discoverability but not the goal of having an official, team-maintained reference implementation.

## Open Questions

1. **Language modes:** Is `yaml/yaml` (1.2) + `yaml/yaml11` (1.1) the right split, or should there be a single mode with a version option in `languageOptions`?
2. **Parser choice:** Which YAML parser best balances maintainability, comment support, location accuracy, and license compatibility? (See parser comparison table above.)
3. **Initial rule set:** Are the proposed rules the right starting point? Should any be added or removed for v1?
4. **Collaboration model:** Should the team formally invite @ota-meshi or other community maintainers as collaborators on the new repo?
5. **Anchor / alias handling:** How deeply should v1 resolve anchors and aliases in the AST? Full resolution enables more rules but adds complexity. One approach is a `sourceCode.resolveAnchor(aliasNode)` method that keeps resolution opt-in for rule authors without complicating the AST.

## Help Needed

- TSC feedback on language modes, rule scope, and parser choice.
- Volunteers for code review and ongoing maintenance (the RFC author will champion implementation).
- Input from users of eslint-plugin-yml and eslint-yaml on must-have rules and config patterns.

## Frequently Asked Questions

**Q: Do I have to migrate from eslint-plugin-yml?**
A: No. The official plugin is an additional option. You can migrate when and if you want. A migration guide will be provided.

**Q: Will this support eslintrc?**
A: No. Like other official language plugins, this targets flat config only (ESLint v9.15.0+).

**Q: Can I use this with Prettier or other formatters?**
A: Yes. This plugin focuses on correctness and best-practice rules, not formatting. You can use Prettier or another tool for YAML formatting alongside `@eslint/yaml` for linting.

**Q: What about YAML 1.1 vs 1.2?**
A: The plugin will provide two language modes — `yaml/yaml` for YAML 1.2 (default, current spec) and `yaml/yaml11` for YAML 1.1 compatibility. This follows the same pattern as `@eslint/json` providing `json/json`, `json/jsonc`, and `json/json5`.

**Q: How does this relate to eslint-plugin-yml and eslint-yaml?**
A: This plugin uses the official language plugins API, which is the recommended way to support non-JavaScript languages in ESLint. Community plugins may use different approaches (parsers, processors) or offer a different rule set. Both can coexist; users choose what fits their needs.

## Related Discussions

- [Interest in an official @eslint/yaml plugin? (Discussion #20482)](https://github.com/eslint/eslint/discussions/20482) — Community interest and TSC welcome for an RFC.
- [RFC #99 — ESLint Language Plugins (2022-languages)](https://github.com/eslint/rfcs/blob/main/designs/2022-languages/README.md) — Language plugin API that this plugin will implement.
- [eslint-plugin-yml](https://github.com/ota-meshi/eslint-plugin-yml) — Community YAML plugin (v3 supports the language plugin API).
- [eslint-yaml](https://github.com/43081j/eslint-yaml) — Another community YAML plugin.
- [2024-09-05 TSC meeting notes](https://github.com/eslint/tsc-meetings/blob/main/notes/2024/2024-09-05.md#do-we-want-to-create-official-language-plugins-for-yaml-and-css) — TSC discussed creating official language plugins for YAML and CSS; CSS was prioritized first.
- [2026-01-08 TSC meeting notes](https://github.com/eslint/tsc-meetings/blob/main/notes/2026/2026-01-08.md) — ESM-only decision for new packages; budget and spending context.
