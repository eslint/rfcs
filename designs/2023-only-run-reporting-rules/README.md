- Repo: eslint/eslint
- Start Date: 2023-01-16
- RFC PR: (leave this empty, to be filled in later)
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

The proposed implementation is to add a `--skip-warnings` flag to ESLint that will skip
running any rules that are set to the `warn` level. This will be a CLI flag only, as it
doesn't make sense to have this flag in a configuration file. This flag would implicitly
imply the `--quiet` flag, given if the `warn` rules are not run, they will not be outputted
to reporters.

From an API perspective, this would be implemented by a predicate function that acts as
a filter on which rules should be run. The predicate would take the rule configuration,
and return whether the rule should be run. For simplicity, this function should not be
given rules marked as `off`, as if this function handled existing behavior then users of the
API would have to mimic that when attempting to extend it.

The flag would be implemented by passing a predicate function that only returns true for
rules that have been set to `error`, thus filtering out any that are marked as `warn`.

The check for unused `eslint-disable` directives (`--report-unused-disable-directives`)
should continue to mark `warn` rules as used, even when running with `--skip-warnings`.

The `--max-warnings` flag should disable the `--skip-warnings` flag, as it requires that
`warn` rules are run to know the current number of reported warnings.

## Documentation

Both the flag and API would make sense to document. As they are relatively simple additions,
they would be documented inline with the rest of the CLI and API documentation. Due to the
potentially major performance improvements for many setups, I feel mentioning it in the
release blog post, alongside the potential benefits, would be a good idea to spread awareness.

## Drawbacks

This change adds a new command line flag, as well as a new API method. This adds further
maintenance burden. However, the implementation is relatively simple, and the benefits
are significant.

Another potential drawback around this specific implementation is that the existence of
both `--quiet` and `--skip-warnings` might be confusing to some users, however this can be
partially alleviated by updating the `--quiet` docs to mention that warnings will still be
run, and to use `--skip-warnings` instead to not run them.

## Backwards Compatibility Analysis

The proposed solution does not cause any compatibility issues, as all functionality is
implemented via new APIs and command line flags. Existing users will be unaffected
by this change.

## Alternatives

### Implement this into `--quiet` instead of a new flag

This alternative would still require the same API changes, but would instead implement
this behavior into the `--quiet` flag. This would be a breaking change, as it would
change the behavior of `--quiet` to not run `warn` rules. The upside of this alternative
would be that it's less confusing to users. This would presumably need to wait for ESLint 9.

Another downside of this alternative is that it would introduce confusion around the
`--max-warnings` flag and other checks that require warnings to run. Currently the quiet
flag works fine alongside these. If we were to implement this into the quiet flag, we would
have to either break that functionality, or have the quiet flag only sometimes stop actual
running depending on what other flags are in use, which is confusing and unclear.

### Expose more power via the CLI flag

Given the API takes a predicate, another alternative would be to also expose this power
to the CLI. This would allow users to specify their own predicate function, which would
allow for more complex filtering. The downside to this change is that it's vastly more
complex for both the end user and to implement. As the API side of this would be done
anyway, this could be implemented alongside the current proposal in the future if necessary.
However, I feel the simple option as outlined in this proposal that covers the majority
use-case is ideal to keep as a flag either way.

## Open Questions

### Flag naming

Whether `--skip-warnings` is the best name for the flag is still an open question. I feel it's
the most clear around the base purpose, however I'm open to hearing other suggestions.

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
