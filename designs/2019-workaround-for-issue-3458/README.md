- Start Date: 2019-11-04
- RFC PR: (leave this empty, to be filled in later)
- Authors: Pete Gonzalez ([@octogonz](https://github.com/octogonz))

# Add resolveRelativeToConfigFile setting

## Summary

RFC #14 proposes a way for a shared ESLint config package to supply its own plugin dependencies, rather than imposing
that responsibility on every consumer of the package.  But because that RFC seeks to design a comprehensive solution,
its discussion has been open for a long time without encouraging progress.  In the interim, this RFC proposes
a temporary workaround, that allows users opt-in to a resolution behavior that works well in practice (despite having
some known limitations).  This feature would be designated as experimental, with the intent to remove it when/if
the ideal solution finally ships.

## Motivation

For the more narrow (and perhaps more common) scenario tackled by this RFC, please refer to
[this comment](https://github.com/eslint/eslint/issues/3458#issuecomment-516666620) from ESLint issue 3458.

The more general set of requirements is already spelled out in RFC #14.

## Detailed Design

Change [line 841 of config-array-factory.js](
https://github.com/eslint/eslint/blob/586855060afb3201f4752be8820dc85703b523a6/lib/cli-engine/config-array-factory.js#L845)
from this:

```js
        try {
            filePath = ModuleResolver.resolve(request, relativeTo);
        } catch (resolveError) {
```

...to this:

```ts
        try {
            filePath = ModuleResolver.resolve(request, importerPath);
        } catch (resolveError) {
```

Gate this change behind a new option `resolveRelativeToConfigFile` in `.eslintrc.js`.  It is an optional boolean
value that is `false` by default.  Thus this behavior will be off by default.

A complete implementation is already provided in [ESLint PR 12460](https://github.com/eslint/eslint/pull/12460).

## Documentation

The PR will include documentation for the new option.

## Drawbacks

The `resolveRelativeToConfigFile` feature does not consider all possible design considerations, such as naming
conflicts between plugins.  The PR also assumes that `resolveRelativeToConfigFile` is assigned by the main project,
not by an extended config.  This is okay, because it's a temporary workaround.  It's not meant to be an ideal design.

## Backwards Compatibility Analysis

No impact, because the option is off by default.

## Alternatives

If RFC #14 can be completed and implemented within a reasonable timeframe, then this workaround would not be needed.

## Open Questions

None.

## Help Needed

None.

## Frequently Asked Questions

### In order to modify one line of logic, do we really need to wait an entire month (21 + 7 days) for an "RFC"?

Yes, the ESLint maintainers have insisted on this formalism.

### Why should we accept a solution with known limitations?

Back in July, the workaround was presented as a
[monkey patch](https://github.com/eslint/eslint/issues/3458#issuecomment-516716165).
Several people are using it in large multi-project repos for serious shipping applications,
and they've reported that it works correctly.  Thus, it's "good enough" for many people who would be otherwise blocked.

### Why can't everyone just use a monkey patch then?

Monkey patching is awkward and brittle.  An .eslintrc.js file should not probe into the ESLint binary
and overwrite its module objects at runtime.  It may be acceptable for small projects, but at a large company,
project admins would be rightly concerned about the supportability of such a solution.

## Related Discussions

None.
