- Repo: eslint/eslint
- Start Date: 2026-05-03
- RFC PR: https://github.com/eslint/rfcs/pull/147
- Authors: xbinaryx

# Report Malformed Rule Deprecation Metadata

## Summary

`RuleTester` should validate the shape of `meta.deprecated` on the rule under test and throw an assertion error when the value does not match the documented `boolean | DeprecatedInfo` contract.

## Motivation

Rule deprecation metadata is used by ESLint to populate `LintResult#usedDeprecatedRules` and by integrations to explain replacement rules. Today, malformed `meta.deprecated` values can pass through rule tests unnoticed, which allows invalid metadata to reach users and downstream tools. `RuleTester` already catches common rule authoring mistakes such as invalid option schemas, missing `meta.fixable`, and missing `meta.hasSuggestions`; deprecation metadata should receive the same early feedback.

## Detailed Design

When `RuleTester#run()` is called, `RuleTester` will validate `rule.meta.deprecated` at the very beginning of the execution, before any valid or invalid test cases are run. This will be implemented as part of the existing `assertRule()` validation function, which runs early in the process. If `meta.deprecated` is omitted or set to `undefined`, no validation error is reported.

Examples of accepted values are:

```js
meta: {
    deprecated: true
}

meta: {
    deprecated: false
}

meta: {
    deprecated: {
        message: "Use eslint-plugin-example/example instead.",
        url: "https://example.com/deprecations/example",
        deprecatedSince: "9.0.0",
        availableUntil: "10.0.0",
        replacedBy: [
            {
                message: "Replacement rule.",
                url: "https://example.com/replacements/example",
                plugin: {
                    name: "eslint-plugin-example",
                    url: "https://example.com/plugin"
                },
                rule: {
                    name: "example",
                    url: "https://example.com/rules/example"
                }
            }
        ]
    }
}
```

The object form must match `DeprecatedInfo`:

- `message`, `url`, and `deprecatedSince` must be strings when present.
- `availableUntil` must be a string or `null` when present.
- `replacedBy` must be an array when present.
- No other top-level `DeprecatedInfo` properties are allowed.

Each `replacedBy` entry must be a `ReplacedByInfo` object:

- `message` and `url` must be strings when present.
- `plugin` and `rule` must be `ExternalSpecifier` objects when present.
- No other `ReplacedByInfo` properties are allowed.

Each `ExternalSpecifier` object may contain:

- `name`, which must be a string when present.
- `url`, which must be a string when present.
- No other `ExternalSpecifier` properties are allowed.

Malformed values cause `RuleTester` to throw before any valid or invalid test case is executed. Error messages should include the invalid metadata path, such as:

```text
Rule's `meta.deprecated` must be a boolean or object.
Rule's `meta.deprecated.replacedBy` must be an array.
Rule's `meta.deprecated.replacedBy[0].plugin.name` must be a string.
```

## Documentation

No updates to the existing documentation are required, as the [Rule Deprecation](https://eslint.org/docs/latest/extend/rule-deprecation) page already defines the accepted runtime shape for rule deprecation metadata. However, because this is a breaking change for rule tests, it will be noted in the migration guide for the next major release.

## Drawbacks

This is a breaking change for rule packages whose tests currently pass with malformed deprecation metadata. Some projects may also be storing custom, nonstandard fields under `meta.deprecated`; those projects will need to remove those fields or move them elsewhere.

## Backwards Compatibility Analysis

Existing rules with valid deprecation metadata are unaffected. Rules with malformed metadata will start failing in `RuleTester`. This disruption is intentional so rule authors discover metadata problems before publishing.

## Alternatives

One alternative is to validate only the top-level `meta.deprecated` type and ignore nested fields. That would catch the most severe malformed values, but it would still allow invalid replacement metadata to reach `usedDeprecatedRules`.

Another alternative is to rely on TypeScript definitions. That helps TypeScript rule authors, but many ESLint rules are written in JavaScript and can still publish malformed metadata.

## Open Questions

None at this time.

## Help Needed

No additional help is needed for the initial implementation.

## Frequently Asked Questions

### Does this affect end users running ESLint?

No. This proposal changes `RuleTester`, so it affects rule authors and maintainers running rule tests.

## Related Discussions

eslint/eslint#20603
