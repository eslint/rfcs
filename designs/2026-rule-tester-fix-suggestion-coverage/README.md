- Repo: eslint/eslint
- Start Date: 2026-09-16
- RFC PR: https://github.com/eslint/rfcs/pull/154
- Authors: morgan-coded

# Require Tests for Declared Fixes and Suggestions

## Summary

`RuleTester` gains opt-in assertions requiring a rule that declares `meta.fixable` to produce at least one autofix and a rule that declares `meta.hasSuggestions` to produce at least one suggestion across the test cases that actually run.

## Motivation

The existing checks enforce behavior-to-declaration consistency, but do not enforce declaration-to-behavior consistency.
At the ESLint source pin `24310e3a0e22b3c086ca402f88448676f2e1cfcd`, `lib/linter/linter.js:589` rejects a reported fix without `meta.fixable`; lines 595-612 reject suggestions without `meta.hasSuggestions`.
The fix error at line 591 is:

```text
Fixable rules must set the `meta.fixable` property to "code" or "whitespace".
```

The suggestion branches at lines 605-607 and 609-611 distinguish the obsolete `meta.docs.suggestion` declaration from an absent declaration.

```text
Rules with suggestions must set the `meta.hasSuggestions` property to `true`. `meta.docs.suggestion` is ignored by ESLint.
Rules with suggestions must set the `meta.hasSuggestions` property to `true`.
```

`lib/rule-tester/rule-tester.js` contains no `fixable` reference, and its direct `rule.meta` reads access `messages`.
Its three `hasSuggestions` references concern a local value derived from the reported message, beginning at `lib/rule-tester/rule-tester.js:1637`.
RFC 147's Motivation already places metadata checks in RuleTester:

> `RuleTester` already catches common rule authoring mistakes such as invalid option schemas, missing `meta.fixable`, and missing `meta.hasSuggestions`; deprecation metadata should receive the same early feedback.

That statement motivates the reverse-direction check; it does not establish that declaration-without-behavior is checked today.
The metadata also matters to documentation generation, `--fix` filtering, and editor integrations, as DMartens described in the issue discussion.
The survey posted on 2026-08-16 covers 351 published rules across seven plugins the author contributes to: 178 declare `meta.fixable`, seven declare `meta.hasSuggestions`, and one active fixable rule has no fix-producing test.
These are survey-wide counts; the population reached by ESLint's RuleTester is smaller.

## Detailed Design

### API and typing

The third argument of `run()` already contains `valid`, `invalid`, and `assertionOptions`; the proposal adds two default-false members to that existing bag.
RFC 137 introduced `requireMessage` and `requireLocation`, and `requireData` was subsequently added without a new RFC.
The proposed call shape is:

```js
ruleTester.run("no-foo", rule, {
    assertionOptions: {
        /**
         * Require an autofix when the rule declares meta.fixable.
         * @default false
         */
        requireFix: false,
        /**
         * Require a suggestion when the rule declares meta.hasSuggestions.
         * @default false
         */
        requireSuggestions: false,
    },
    valid: [],
    invalid: [],
});
```

Both members extend the inline anonymous `assertionOptions` type at `lib/types/index.d.ts:1471`, using its existing TSDoc style.
No exported interface, constructor argument, static method, `namespace RuleTester` change, test-case type change, or `@types/eslint` change is proposed.
The runtime JSDoc bag begins at `lib/rule-tester/rule-tester.js:1028`; option handling remains on `run()`.

### Accumulation and signals

A module-level `WeakMap` keys state on the original rule object passed to `run()`, never the label or the internal rule wrapper.
The existing `forbiddenMethodCalls` map at `lib/rule-tester/rule-tester.js:154` provides an in-file precedent for module-level state.
In the census, 38 of 432 lexical `run()` calls use a drifted label, and five of those label a different published rule.
Each entry stores armed flags, registered and finished run counters, registered and executed invalid-case counts, and independent observed-fix and observed-suggestion flags.
The `requireFix` and `requireSuggestions` flags merge with logical OR across runs, so one armed run arms that rule object in the current module scope.
The fix signal reuses `messages.some(m => m.fix)` at `lib/rule-tester/rule-tester.js:1320`, with its result returned to the invalid-case path.
The suggestion signal is `messages.some(m => Array.isArray(m.suggestions) && m.suggestions.length > 0)`; an empty suggestions array does not count.
Only invalid-case execution records signals; valid cases do not contribute.
A suggestion's fix is applied separately at `lib/rule-tester/rule-tester.js:1815`, so suggestion output cannot satisfy `requireFix`.
Four suggestion-only rules assert suggestion `output`, while zero fixable rules depend on that shape for fix evidence.

### Delivery and its execution assumptions

Each armed run with invalid cases registers its coverage callback as the last `it()` inside the run's `invalid` suite.
Mocha runs a suite's own tests before its child suites, so a sibling callback after the invalid suite executes before the invalid cases it is supposed to follow.
The `invalid` suite starts at `lib/rule-tester/rule-tester.js:1944`; the coverage callback belongs after the case-registration loop ends at line 1987 and before the suite closes at line 1988.
The trailing callback increments `finished` and renders a verdict only when `finished === registered` and `invalidExecuted === invalidRegistered`.
The verdict is only rendered for a rule object if at least one invalid test case for that rule actually executed in the current scope.
The corpus contains 44 empty-invalid runs and 36 source shapes without an invalid key; 24 of those runs concern fixable rules.
A missing invalid key remains an API error: `assertTest()` requires an array at `lib/rule-tester/rule-tester.js:543`; wrapper or fixture source shapes do not relax that requirement.
Excluding zero-invalid execution takes measured per-file false positives from four to zero.
Suite registration before test execution makes the registered counter stable in the describe-based runners considered by the dossier.
The coverage verdict is rendered only from inside the run's `invalid` suite, after every invalid test case registered for that rule object in the current module scope has executed and every armed run's coverage callback has finished; because the verdict requires that every registered invalid case actually ran, any execution that is filtered, skipped, or reordered ahead of the callback yields no verdict rather than a false one - and the check sees only one module scope: process-wide under mocha, per test file under vitest, jest and node:test.
If filtering suppresses any registered coverage callback, the counter cannot complete and the aggregate check stays silent.
The accumulator sees one module scope, so a rule whose only fix-producing case lives in another test file is a false positive under vitest, jest, and `node:test`.
The RuleTester-visible corpus contains 0 cross-file fixable rules; all 29 cross-file rules live in `@stylistic`, outside RuleTester's reach.

The delivery prototype's 116-run matrix covers mocha 12.0.1, vitest 5.0.1, jest 30.5.1, and `node:test` on Node v26.3.0, using `eslint@10.10.0` with `lib/rule-tester/rule-tester.js` byte-identical to the RFC's pin.
The prototype wraps `context.report` and tests `typeof descriptor.fix === "function"` as a stand-in for `messages.some(m => m.fix)`; no fixture exercises the suggestion path, and the ordering and filtering results transfer because both predicates fire inside the same invalid-case body.
On those runners, inner placement and the executed-equals-registered condition produced correct verdicts or silence in default, concurrent, and shuffled modes within one module scope, while filtered or skipped executions yielded no verdict.
Across 12 `jest --randomize` seeds, the filter-aware condition produced 0 false positives and went silent on 5.
Those five silent seeds miss detections on a rule with no fix-producing case; a missed verdict is recoverable, while a failing test on a correctly covered rule is not.

When an armed declaration lacks its corresponding observed signal, the check uses these messages:

```text
The rule declares `meta.fixable` but no test case produced a fix. Please add an invalid test case with an 'output' property.
The rule declares `meta.hasSuggestions` but no test case produced a suggestion. Please add an invalid test case with a 'suggestions' property.
```

The messages omit the rule name because the enclosing `describe(ruleName)` supplies it.

### Framework scope

The dossier records these module-isolation scopes; changing a runner's isolation settings can change the accumulation boundary.

| Runner | Accumulation scope under the recorded configuration |
| --- | --- |
| Mocha | Process-wide for the surveyed single-process module registry, with registration before execution. |
| Vitest | Per file with the default forks pool and `isolate: true`. |
| Jest | Per file because each test file has an independent module registry. |
| node:test | Per file with default process isolation; shared under isolation `none`. |

The isolation references are [Vitest parallelism](https://vitest.dev/guide/parallelism), [Jest configuration](https://jestjs.io/docs/configuration), and [Node test isolation](https://nodejs.org/api/test.html).
Both per-file and per-rule scopes have zero measured false positives after the carve-out; the runner experiment separately measures callback delivery.
The corpus uses three Mocha, two Vitest, two Jest, and zero node:test setups.

### Behavior matrix

| Case | Proposed result or limit |
| --- | --- |
| Empty invalid array | No verdict unless another run of the same object executes an invalid case. |
| Missing invalid key | Existing ESLint input validation rejects it; wrappers may supply arrays from other source shapes. |
| `only`, name filtering, or skipped suites | No verdict on all four measured runners, including name filtering that retains every coverage callback. |
| Both metadata flags | Independent fix and suggestion requirements; neither substitutes for the other. |
| `output: null` only | No fix evidence; existing assertions require unchanged output. |
| Fix only in a later run | Passes within one module scope when the verdict is rendered; both measured examples are outside ESLint RuleTester reach. |
| Never passed to `run()` | Not checked; finding unregistered rules is a non-goal. |
| Alias or re-export | Identity is the object supplied to `run()`, irrespective of its published names. |
| No test framework | Immediate default handlers interleave registration and execution, reducing scope to per run; both measured variants produced a false positive. |

Three surveyed rules declare both flags, and all three have evidence on both axes.
The two later-fix examples are `stylistic/type-generic-spacing` and `stylistic/indent`.
Six of 351 published rule names are never passed to `run()`: four deprecated re-exports, `n/hashbang` tested through its `shebang` alias, and `n/process-exit-as-throw` tested with a raw Linter.
The unchanged-output check is at `lib/rule-tester/rule-tester.js:1870`; immediate default handlers are at lines 865-885.
The cost is one extra test per armed run and zero extra tests with both defaults false.
The census counts lexical calls, not a promise of the same runtime test-count increase.

## Documentation

The existing `docs/src/integrate/nodejs-api.md:865` RuleTester section is the documentation surface; no new page is proposed.
Document the options under `RuleTester#run()` at `docs/src/integrate/nodejs-api.md:964` and its assertion-options list at line 1010.
Explain signal separation under Testing Fixes at `docs/src/integrate/nodejs-api.md:1044` and Testing Suggestions at line 1073.
Extend Enforcing assertionOptions Globally at `docs/src/integrate/nodejs-api.md:1176` with the existing wrapper or subclass approach.
No blog post accompanies the default-false option; a migration-guide entry belongs with any future default change.

## Drawbacks

At least one observed fix is a metadata smoke test, not coverage of every fixer branch; it can give false confidence, as aladdin-add argued in the issue.
The extra trailing test adds reporter noise to every armed run.
The assertion-options surface grew from two members to three within a year, and this proposal takes it to five.
Rules never passed to RuleTester and harnesses that bypass it remain invisible.
Immediate default handlers reduce scope to per run and produced a false positive on both measured variants; the filter-aware gate can yield silence under reordered execution.

## Backwards Compatibility Analysis

ESLint's Semantic Versioning Policy puts new public capabilities in a minor release:

> New capabilities to the public API are added (new classes, new methods, new arguments to existing methods, etc.).

The policy puts incompatible API changes in a major release:

> Part of the public API is removed or changed in an incompatible way.

The research dossier records RFC 137's implementation and `requireData` as `feat:` changes shipped in v10 minors.
Its release baseline is v10.10.0, published 2026-09-04; this proposal therefore targets an opt-in v10 minor.
With both defaults false, existing suites gain no new failure or extra test, and end users running ESLint are unaffected.
Opting in affects rule authors' test suites.

The check governs tests that run through ESLint's RuleTester.
The RuleTester-visible subset contains 75 fixable rules and four suggestion rules, with zero measured missing-behavior cases.
`eslint-vitest-rule-tester`, used by `@stylistic/eslint-plugin`, and the `@typescript-eslint/rule-tester` fork, used by `eslint-plugin-import-x`, drive Linter directly and are out of scope until they port the option.
The typescript-eslint fork already carries its own assertion-options type, demonstrating the separate porting surface.
The survey's single true positive is `import-x/no-import-module-exports`, which runs through that fork and is not caught by this proposal today.
If equivalent checks reached the entire survey, the measured missing-behavior cost would be one of 351 rules and zero of seven suggestion rules; the directly reachable subset contributes zero.

A default flip in v11 is a candidate for a future TSC decision, not a commitment of this RFC.
The measured cost is small, but seven plugins the author contributes to do not establish ecosystem-wide impact.
The v10 migration guide supplies the `@eslint/v9-to-v10-ruletester` codemod for RuleTester breaking changes at `docs/src/use/migrate-to-10.0.0.md:20`, with references at lines 293 and 444.
A codemod here could insert opt-outs; it cannot author the rule-specific fix-producing test that is missing.
RFC 137 explicitly did not plan to enable its assertions by default, so a promised flip would create a new commitment rather than follow an established assertion-options pattern.

## Alternatives

### Assert on finished runs alone

This variant renders a verdict after every armed run's coverage callback finishes, without requiring every registered invalid case to execute.
It is rejected because it produced false positives in 6 measured cells: all four runners under name filtering, plus 2 of 12 `jest --randomize` seeds.

### Assert in the last invalid case of each run

Per-run accumulation and an assertion in the last invalid `it()` remove module state and the extra test, with a smaller implementation.
The price is narrower scope: zero fixable and one suggestion false positive in the RuleTester-visible subset.
Across the full corpus the counts are nine fixable and three suggestion false positives, dropping to three and three after the zero-invalid carve-out.
The affected run can disable its corresponding option; this remains the fallback if aggregate delivery is rejected.

### A static lint rule

`eslint-plugin-eslint-plugin` already has `require-meta-fixable` with `catchNoFixerButFixableProperty`, default false.
Its schema explains the limitation at `lib/rules/require-meta-fixable.ts:36` in that repository's `eda5e47` pin:

> This option is off by default because it increases the chance of false positives since fixers can't always be detected when helper functions are used.

In eslint-plugin-yml, 28 of 30 test files use a fixture loader, and 19 fixable rules have no inline output assertions.
Its loader can generate a missing output fixture from current behavior, allowing a broken expectation to be recorded without an independently authored expected output.
eslint-plugin-jsdoc routes 76 rules through a shared runner; 12 use shared metadata builders and nine have no separate rule file.
These indirections limit a single-file static check; a per-run approximation produces nine full-corpus fixable false positives, versus zero at rule-object scope.
This addresses nzakas' linting-versus-testing objection through observation scope rather than treating the venue as the distinction.

### Explicit groups

JoshuaKGoldberg's [2024-07-09 sketch](https://github.com/eslint/eslint/issues/18008#issuecomment-2216503793) proposes a `runAll` method accepting test groups or a `RuleTesterGroup` class with `runAll`.
Explicit complete groups avoid the measured grouping false positives by construction.
The RuleTester-visible subset has two single-file multi-run rules and zero cross-file rules, so the measured need for call-site restructuring is small.
Both group APIs add a second entry point and require adopters to restructure existing tests.
The 75 fixable and four suggestion rules across jsdoc, eslint-plugin, n, yml, and promise have zero cross-file fixable rules and zero rule-object false positives; the reach caveat above defines that subset.

### Other delivery and policy choices

A process-exit assertion loses ordinary test attribution and does not follow test filtering; it also cannot combine isolated module registries.
A global assertion-options static was declined in issue 20613 in favor of documentation, and PR 20752 closed unmerged; an explicit finish static would add another manual hook.
RFC 122's hooks belong to individual test cases and do not provide an aggregate suite-completion hook.
Coverage tooling answers a broader question but requires separate setup; this assertion only observes whether declared behavior occurs.
A warning-only result leaves the missing behavior non-failing, weakening the proposed invariant.
A combined `strict` option is outside scope; the proposal retains separate assertions, consistent with RFC 137's decision not to add `strict`.

## Open Questions

None at this time.

## Help Needed

I can implement the RuleTester change, tests, types, and documentation after the design is agreed.

## Frequently Asked Questions

### Does this prove fixer coverage?

No; it establishes that at least one invalid case produced the declared kind of behavior, not that every fixer path is tested.
The corpus's nine per-run fixable false positives fall to zero at rule-object scope, which measures grouping effects rather than branch coverage.

### Can a suggestion satisfy the fix assertion?

No; the top-level fix and suggestion-array signals remain independent.

### Does this change end-user linting?

No; these options affect RuleTester test execution.

## Related Discussions

- [eslint/eslint issue 18008](https://github.com/eslint/eslint/issues/18008), including JoshuaKGoldberg's 2024-07-09 grouping sketch.
- [RFC 137: assertion options](https://github.com/eslint/rfcs/pull/137).
- [RFC 122: test-case hooks](https://github.com/eslint/rfcs/pull/122).
- [RFC 101: suggestion parse errors](https://github.com/eslint/rfcs/pull/101).
- [RFC 103: rule errors](https://github.com/eslint/rfcs/pull/103).
- [RFC 147: deprecation metadata](https://github.com/eslint/rfcs/pull/147).
- [eslint/eslint issue 20613: global assertion options](https://github.com/eslint/eslint/issues/20613).
- [JoshuaKGoldberg's grouping sketch](https://github.com/eslint/eslint/issues/18008#issuecomment-2216503793).
- [nzakas' clearance to write the RFC](https://github.com/eslint/eslint/issues/18008#issuecomment-2286484089).
- [nzakas' invitation to pick up the RFC](https://github.com/eslint/eslint/issues/18008#issuecomment-5667171467).
