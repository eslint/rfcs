- Repo: eslint/eslint
- Start Date: 2021-04-11
- RFC PR: (leave this empty, to be filled in later)
- Authors: [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Fixable Disable Directives

## Summary

Inline disable directives such as `/* eslint-disable */` that do not suppress any violations may be detected with the `--report-unused-disable-directives` CLI option, but there is no way to automatically remove those comments.
This RFC proposes adding removal of those directives to the `--fix` CLI option as a new [`--fix-type`](https://eslint.org/docs/user-guide/command-line-interface#-fix-type): _**`meta`**_.

## Motivation

Manually deleting comments from `--report-unused-disable-directives` is cumbersome, especially in large repositories with many directives.
Users would benefit from a quick way to fix these without having to manually map between CLI output lines and files.

## Detailed Design

Recapping the existing flow of code:

- When the `--fix-type` CLI option is specified, `options.fix` is patched to filter on the `rule.meta` of each fix's corresponding `ruleId`
- Unused disable directives are calculated by an `applyDirectives` function within the `applyDisableDirectives` function called in the linter's `_verifyWithoutProcessors`
  - These problems have a `ruleId` of `null`
- `_verifyWithoutProcessors` is called within the call stack of each of the 1-10 passes in the linter's `verifyAndFix`

This RFC proposes making three changes:

- In the patched `options.fix`, consider a problem's meta type to be `"meta"` if its `ruleId` is `null`
- Pass the `SourceCode` being linted as a parameter to `applyDisableDirectives`
- Add a `fix` method to the unused directive problems returned by `applyDirectives` that uses the `SourceCode` to compute:
  - For comments that are the only non-whitespace on their line, delete that line
  - Otherwise, delete just the comment and any now-unnecessary surrounding whitespace

### Fix Behavior Examples

```diff
- /* eslint-disable */
```

```diff
- // eslint-disable-next-line -- related explanation
```

```diff
- before /* eslint-disable-next-line -- related explanation */ after
+ before after
```

```diff
- before // eslint-disable-next-line -- related explanation
+ before
```

```diff
  // before
- // eslint-disable-next-line
  // after
```

Multiline block comments are already not allowed to be disable directives. [eslint/eslint#10334](https://github.com/eslint/eslint/issues/10334)

## Documentation

- [Command Line Interface > Fixing Problems](https://eslint.org/docs/user-guide/command-line-interface#fixing-problems)'s `--fix` documentation should mention the added directives fixing and the new `--fix-type`
- [Rules > Report Unused ESLint Disable Comments](https://eslint.org/docs/user-guide/configuring/rules#report-unused-eslint-disable-comments) should link to that documentation

## Drawbacks

Like any new feature, this flag will slightly increase the complexity and maintenance costs of ESLint core.

This RFC's implementation would lock in the name for a new `--fix-type` even though we only have one concrete use case for it.

## Backwards Compatibility Analysis

It is unlikely but not impossible that some `--fix` dependant users would rely on unused disable directives remaining in code.

Otherwise, this change is additive in behavior.

## Alternatives

- Applying directive fixes after the `verifyAndFix` passes
  - Not ideal: deleting a comment could introduce new rule violations that would warrant running >=1 other pass
- Writing an external tool to apply these options
  - Not ideal: the internal implementation is simple; external would have to deal with variant formatters, etc and suffer from the same introduced rule violations

## Open Questions

- Is `"meta"` a good name for the fix type?
  - I'd considered `"directive"`, but that felt like it could be too specific to be extended for other meta/configuration in the future.

## Help Needed

I'm looking forward to implementing this if approved! 🙌

## Frequently Asked Questions

> Will these be autofixed by default?

Yes: problems reported by directive usage checking are joined with remaining rule violation problems in a single array.
This should allow directive fixing to seamlessly act similarly to rule fixing.

> Can I opt out?

Yes.
If you are in the peculiar situation of needing to enable `--fix` and `--report-unused-disable-directives` _without_ fixing those directives _(why?)_, you can use `--fix-type` with all types except `meta`.

## Related Discussions

- [eslint/eslint#10334](https://github.com/eslint/eslint/issues/10334) - Report an error for eslint-disable-line comments that span multiple lines #1033)
- [eslint/eslint#9249](https://github.com/eslint/eslint/issues/9249) - Add CLI option to report unused eslint-disable directives
- [eslint/eslint#11815](https://github.com/eslint/eslint/issues/11815) - --report-unused-disable-directives should be autofixable
