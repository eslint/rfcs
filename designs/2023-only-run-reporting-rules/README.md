- Repo: eslint/eslint
- Start Date: 2023-01-16
- RFC PR: <https://github.com/eslint/rfcs/pull/104>
- Authors: Maddy Miller

# Only run reporting lint rules

## Summary

Currently, ESLint will run all rules that are not marked as `off` in the configuration.
This RFC proposes adding a way to configure which rules are actually run, to reduce linting
time and better match the reporting outcome. Currently, when a rule is marked as `warn`,
ESLint will still run the rule but not report the results when run under `--quiet`. The
intended outcome of this RFC is to allow users to not run these rules when unnecessary.

## Motivation

A common use-case of the `warn` level is to report errors to the developer in their IDE,
but to completely ignore it when running ESLint on the command line via the `--quiet` flag.
When running with many rules set to the `warn` level, this can cause a significant amount
of overhead on ESLint runtime. On an example large codebase with 154988 lines of TypeScript
code, I found that by disabling all rules set to warn, I was able to reduce ESLint runtime
from 4 minutes and 39 seconds to just 22 seconds. As the `warn` rules are entirely ignored
in this situation, this is a substantial amount of wasted time and resources that could be
avoided.

## Detailed Design

The proposed implementation is to modify the `--quiet` flag so that ESLint will skip
running any rules that are set to the `warn` level.

From an API perspective, this would be implemented by a filter function that filters down to
which rules should be run. The function would take `(ruleId, severity)` and return true to
include a rule or false to exclude it. For simplicity, this function should not be
given rules marked as `off`, as if this function handled existing behavior then users of the
API would have to mimic that when attempting to extend it.

In `cli.js`'s `translateOptions` function, the `ruleFilter` option should be assigned to
a function that filters to rules with a `severity` of 2 (error) when the `--quiet` flag is applied,
otherwise always returns true. In `eslint/flat-eslint.js`, the `ruleFilter` should be taken from
the `eslintOptions` object, and passed down to the `linter.verifyAndFix` call.

Within `linter.js`, the API should be added to `VerifyOptions`, and will be passed down into and
utilized within the `runRules` function during the `configuredRules` loop, after the check
for disabled rules and the rule existing. The `ruleId`, and `severity` should be passed into the
predicate function as an object. `normalizeVerifyOptions` should verify that the `ruleFilter`
option is a function, and replace it with an always-true function if not. `processOptions` in
`eslint-helpers.js` should also perform a validation check that the `ruleFilter` option is a function.

The new `ruleFilter` function when implemented would look like this, using the `--quiet` flag
rules outlined in this RFC as an example:

```typescript
linter.verifyAndFix(text, configs, {
  ruleFilter: ({ ruleId: string, severity: number }) => {
    return severity === 2;
  },
});
```

The function acts as a filter on the rules to enable, where removal from the list disables a rule.
This function would return true for all rules which have a severity of 2 (error), effectively filtering out
all other rules.

The check for unused `eslint-disable` directives (`--report-unused-disable-directives`)
should continue to mark `warn` rules as used, even when running with `--quiet`. As the
rules are not actually run, an assumption would have to be made that all directives
on rules present only in the un-filtered list are used when in this mode. This is a reasonable
assumption, as the user likely does not expect `warn` flags to be touched at all in this mode.
This would also apply to blanket `eslint-disable` directives that disable all rules, which
should always be assumed to be used when the filter function has not changed the length of the list.

In cases of conflicting flags such as the `--max-warnings` flag, this altered `--quiet` flag
behavior should be disabled while it is in use. This is to ensure that the `--max-warnings`
flag continues to work as expected. The filter API should not be set, and existing
behavior of filtering the output should be used. This altered behavior should be documented
to clarify that it causes warnings to be run despite the `--quiet` flag.

## Documentation

Both the behaviour modification of impacted flags and the API would make sense to document.
As they are relatively simple additions, they would be documented inline with the rest
of the CLI and API documentation. Due to the potentially major performance improvements for
many setups, I feel mentioning it in the release blog post, alongside the potential benefits,
would be a good idea to spread awareness.

## Drawbacks

This change modifies a command line flag, as well as a new API method. This adds further
maintenance burden. However, the implementation is relatively simple, and the benefits
are significant.

As this flag has many interactions with other systems such as `--max-warnings`, further
maintenance overhead is introduced by having to ensure it behaves correctly between
the different flags. This would require updating documentation across all flags and relevant
settings.

## Backwards Compatibility Analysis

This is a breaking change as it alters the behaviour of the `--quiet` flag.
While the alterations to the quiet flag should not affect the actual outcome of the command,
as cases where it would are covered by this RFC, it is still worth noting as the behaviour
is changing.

For the case of rules with side effects, such as the `react/jsx-uses-vars` rule, these side
effects will no longer run in quiet mode when set to `warn`. Due to this, users will need to
update their eslint configuration to ensure that any rules with side effects are set to `error`.

## Alternatives

### Implement this via a separate `--skip-warnings` flag

This alternative would be to implement this via a separate `--skip-warnings` flag, rather
than into the `--quiet` flag. The same API changes would still be required. The benefit of
this alternative is that it doesn't break behavior of the `--quiet` flag, but it also can
cause confusion with a new flag.

### Expose more power via the CLI flag

Given the API takes a filter, another alternative would be to also expose this power
to the CLI. This would allow users to specify their own filter function, which would
allow for more complex filtering. The downside to this change is that it's vastly more
complex for both the end user and to implement. As the API side of this would be done
anyway, this could be implemented alongside the current proposal in the future if necessary.
However, I feel the simple option as outlined in this proposal that covers the majority
use-case is ideal to keep as a flag either way.

## Open Questions

### Are there any other considerations I've missed

In the proposal I've outlined two checks that require warnings to run, however I'm not
familiar with all internal and external features of ESLint, so might have missed some other
checks that we need to ensure still function correctly with this flag.

## Help Needed

I'm not entirely familiar with the ESLint internals, but would be willing to at least
partially implement this if given guidance on the specific areas of code that would need
to be changed.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

https://github.com/eslint/eslint/issues/16450
