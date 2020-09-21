- Repo: eslint/eslint
- Start Date: 2020-09-21
- RFC PR: (leave this empty, to be filled in later)
- Authors: [CoryDanielson](https://github.com/corydanielson)

# `RuleTester` debugger improvements

## Summary

Add a `before` hook to the valid/invalid code configuration, so that developers can inject a debugger into RuleTester before code is passed into the rule being tested.

## Motivation

Giving developers a mechanism to add a debugger before specific code is passed into their rule, would streamline and improve the developer experience when writing rules. Currently, when a developer wants to debug their rule against specific test code they will most likely do one of the following:
    1. Comment out the other test code blocks - the developer will be unable to see if they have broken an existing test while changing code.
    2. Rearrange the test code blocks so that the one they wish to debug comes first - this discourages or breaks organization
    3. Press the "Play/Resume" button while debugging X times until the debugger is paused at the code block they want to debug - tedious, especially when there are lots of tests or add new ones (X may change)
    4. Some clever conditional debugging - the developer is spending time writing conditional debugger logic instead of the eslint rule.

## Detailed Design

1. Add an optional `before` function property to the config for valid/invalid code.
2. Update RuleTester to call this function (if it exists) just prior to the valid/invalid code being passed into the rule.
3. If the autofix code is tested as well, the before hook should be called before that as well.
4. Because of the potential to call the `before` hook twice - it may make sense to pass in an argument that defines when this code is being called. (ie a string: 'create' | 'fix') This would enable a developer to conditionally debug whichever code they want.
5. Possibly export methods from RuleTester that can be used as `before` values, to debug code
    -- RuleTester.debugRule = () => debugger;
    -- RuleTester.debugCreate = (step) => step === 'create' && debugger;
    -- RuleTester.debugAutofix = (step) => step === 'autofix' && debugger;
6. In order to avoid debuggers reaching production code, RuleTester should fail test cases when a before hook is specified. Once the before hooks are removed from all valid/invalid code, the automatic test failure would be avoided.

## Documentation

[RuleTester](https://eslint.org/docs/developer-guide/nodejs-api#ruletester) should be updated.

## Drawbacks

The `before` hook could be seen as ambiguous. It's possible that users may also expect an `after` hook to be available. Users may confuse this with hooks provided by test frameworks. Users might use these hooks as an opportunity to introduce solutions to solved problems such [logging performance](https://eslint.org/docs/1.0.0/developer-guide/working-with-rules#performance-testing-3).


## Backwards Compatibility Analysis

Since this change only introduces an addition to configuration, there should be no backwards compatibility issues.

## Alternatives

The alternative to a `before` hook, would be a debugger that lives in RuleTester or elsewhere, but that would introduce a maintenance issues especially if users decide to uncomment it or unwrap any if-statement that protect it. My [initial idea](https://github.com/eslint/eslint/issues/13625) was to wrap the `create` and `fix` methods for a rule with a function that adds a debugger before they are called.

```
let _create = create;
if (code.debug) {
    _create = (...args) => {
        debugger;
        return create(...args)
    }
}
```

## Open Questions

- Which approach is preferred? Inserting a debugger into the codebase, or a `before` hook that users can provide themselves.
- If a `before` hook is chosen, should RuleTester export functions (`debugRule`, `debugCreate`, `debugAutofix`) to easily allow developers to debug a rule's create/fix?

## Help Needed

I would enjoy making this contribution into RuleTester, but I might not be available if/when the RFC is approved. I will let you know if I am unavailable at that time.

## Frequently Asked Questions

Would any arguments be passed into the `before` hook?
- Yes possibly a value to indicate if the before call is made before `create` or `fix`. If other use cases are identified for this hook, we can pass in the full arguments that would be passed into `create`/`fix` (or other args?)

If there is a `before` hook - is there also going to be an `after` hook?
- Possibly - If there is a valid use case. This may be out of scope for this specific RFC?

## Related Discussions

https://github.com/eslint/eslint/issues/13625
