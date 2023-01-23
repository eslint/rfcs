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
of overhead on ESLint runtime. On an example large codebase, I found that by disabling all
rules set to warn, I was able to reduce ESLint runtime from 4 minutes and 39 seconds to just
22 seconds. As the `warn` rules are entirely ignored in this situation, this is a substantial
amount of wasted time and resources that could be avoided.

## Detailed Design

The proposed implementation is to modify the `--quiet` flag so that ESLint will skip
running any rules that are set to the `warn` level.

From an API perspective, this would be implemented by a predicate function that acts as
a filter on which rules should be run. The predicate would take the rule configuration,
and return whether the rule should be run. For simplicity, this function should not be
given rules marked as `off`, as if this function handled existing behavior then users of the
API would have to mimic that when attempting to extend it.

The flag would be implemented by passing a predicate function that only returns true for
rules that have been set to `error`, thus filtering out any that are marked as `warn`.

The check for unused `eslint-disable` directives (`--report-unused-disable-directives`)
should continue to mark `warn` rules as used, even when running with `--skip-warnings`.
As the rules are not actually run, an assumption would have to be made that all directives
on rules marked `warn` are used when in this mode. This is a reasonable assumption, as
the user likely does not expect `warn` flags to be touched at all in this mode.

The `--max-warnings` flag should be disabled when the `--quiet` flag is in use, as it requires that
`warn` rules are run to know the current number of reported warnings. This should provide an
error when the two flags are used together.

## Documentation

Both the flag and API would make sense to document. As they are relatively simple additions,
they would be documented inline with the rest of the CLI and API documentation. Due to the
potentially major performance improvements for many setups, I feel mentioning it in the
release blog post, alongside the potential benefits, would be a good idea to spread awareness.

## Drawbacks

This change adds a new command line flag, as well as a new API method. This adds further
maintenance burden. However, the implementation is relatively simple, and the benefits
are significant.

As this flag has many interactions with other systems such as `--max-warnings`, further
maintenance overhead is introduced by having to ensure it behaves correctly between
the different flags. This would require updating documentation across all flags and relevant
settings, rather than just the added flag.

## Backwards Compatibility Analysis

The proposed solution does not cause any compatibility issues, as all functionality is
implemented via new APIs and command line flags. Existing users will be unaffected
by this change.

## Alternatives

### Implement this via a separate `--skip-warnings` flag

This alternative would be to implement this via a separate `--skip-warnings` flag, rather
than into the `--quiet` flag. The same API changes would still be required. The benefit of
this alternative is that it doesn't break behavior of the `--quiet` flag, but it also can
cause confusion with a new flag.

### Expose more power via the CLI flag

Given the API takes a predicate, another alternative would be to also expose this power
to the CLI. This would allow users to specify their own predicate function, which would
allow for more complex filtering. The downside to this change is that it's vastly more
complex for both the end user and to implement. As the API side of this would be done
anyway, this could be implemented alongside the current proposal in the future if necessary.
However, I feel the simple option as outlined in this proposal that covers the majority
use-case is ideal to keep as a flag either way.

## Open Questions

### How to handle conflicting flags

This proposal notes that the `--max-warnings` flag would be disabled under `--quiet`, however
this might not be the best option. Rather than preventing it working, it might make sense to
warn the user that this mode will still run the `warn` rules despite being in quiet mode,
so that a setting like `--max-warnings` can still co-exist with `--quiet`.

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
