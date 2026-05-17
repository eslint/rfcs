- Repo: eslint/eslint
- Start Date: 2026-03-27
- RFC PR: eslint/rfcs#146
- Authors: sethamus

# Respect Bulk Suppressions During Autofix

## Summary

ESLint should stop applying autofixes for a rule in any file where that rule is suppressed via the suppressions file. This should depend only on whether the suppressions data contains a matching file-and-rule entry, not on the stored suppression count.

For example, if `eslint-suppressions.json` indicates that `semi` is suppressed for `src/a.js`, then `semi` fixes are disabled for `src/a.js`. Other rules in `src/a.js` may still be autofixed, and `semi` may still be autofixed in files where it is not suppressed.

## Motivation

Bulk suppressions are meant to separate an existing backlog from new work. That separation breaks down when `--fix` rewrites code that the suppressions file is explicitly marking as out of scope.

This is especially painful in large or inherited codebases, where a developer may make a small change, run `eslint --fix`, and end up with a large unrelated diff because ESLint also fixed suppressed rules in the same files. Even when those fixes are technically safe, they create churn in code that teams have deliberately deferred.

## Detailed Design

### High-Level Behavior

When bulk suppressions are active for a run, ESLint should treat each suppressed file-and-rule pair as ineligible for autofix.

Examples:

- If `semi` is suppressed for `src/a.js`, then `semi` fixes are skipped in `src/a.js`.
- If `semi` is not suppressed for `src/b.js`, then `semi` fixes may still be applied in `src/b.js`.

The stored suppression count is ignored for autofix purposes. A single suppression entry for a file and rule is enough to disable autofix for that rule in that file.

### Why Count Is Ignored

Bulk suppressions are stored as counts, not as identities for specific lint messages. Because ESLint cannot reliably tell which current violations correspond to historical suppressed ones, this RFC does not try to autofix only the "new" violations.

Instead, once a rule is suppressed for a file, autofix for that rule is disabled for the entire file until the suppression entry is removed. This is intentionally coarse:

- `eslint-suppressions.json` says `src/a.js -> semi -> count: 3`
- `src/a.js` currently has 5 `semi` errors

For reporting, ESLint can continue to use the existing count-based suppression behavior. In this example, because the current count exceeds the stored count, none of the `semi` errors are suppressed and ESLint reports all 5. For autofix, however, all `semi` fixes in `src/a.js` are still skipped because `semi` is suppressed for that file.

### CLI Behavior

For the CLI, this behavior should happen automatically whenever the run uses bulk suppressions.

That means a command such as:

```bash
eslint --fix
```

should respect the active suppressions file for that invocation.

The same should apply to:

```bash
eslint --fix-dry-run
```

Likewise:

```bash
eslint --fix --suppress-rule semi
```

should use the suppressions file that exists at the start of the run when deciding which fixes to skip. Newly added suppressions from the same invocation should not retroactively change which fixes were already eligible earlier in the run.

### Node.js API Behavior

This RFC does not add a separate API option. When using the Node.js API, autofix should respect suppressions whenever `applySuppressions` is enabled for the run. For example:

```js
const eslint = new ESLint({
  applySuppressions: true,
  fix: true,
});
```

In this configuration:

- suppressed file-and-rule pairs are not autofixed
- unsuppressed file-and-rule pairs remain autofixable

For `lintText()`, this behavior only applies when `filePath` is provided, because suppressions are keyed by file path.

### Reporting Semantics Remain Separate

This RFC only changes autofix eligibility. ESLint may continue to use counts for reporting and pruning exactly as it does today.

As a result, reporting and autofix operate at different levels of precision:

- reporting remains count-based
- autofix becomes file-and-rule based

That difference is intentional.

### Implementation Approach

The implementation would likely touch the following files:

A **suppressions snapshot** is the in-memory representation of the `eslint-suppressions.json` file contents, read once at the start of the lint run. By capturing the suppressions data before linting begins, autofix decisions are based on a stable, point-in-time view of the file rather than a version that may change during the run (for example, if `--suppress-rule` writes new entries to the file during the same invocation).

- `lib/cli.js`: for CLI runs with `--fix` or `--fix-dry-run`, resolve the active suppressions file before linting, load its current contents, and pass that suppressions snapshot into the `ESLint` instance used for fix generation. This makes autofix depend on the suppressions file contents loaded before linting begins.
- `lib/shared/translate-cli-options.js`: pass that internal suppressions snapshot through CLI option translation so it reaches the `ESLint` instance.
- `lib/eslint/eslint.js`: initialize `SuppressionsService` when suppressions are needed for fix filtering as well as reporting, and load suppressions early enough in `lintFiles()`. In `lintText()`, compute the suppressed rules for the current file and build the suppressions-aware fixer before passing it to `verifyText()`.
- `lib/eslint/eslint-helpers.js`: extend the fixer-building logic used by `lintFile()` so it composes the existing fix predicate with a suppressions-aware predicate. For the current file, if `message.ruleId` appears in that file's suppressions entry, return `false` so the fix is skipped.
- `lib/services/suppressions-service.js`: add a helper that maps a file path to the set of suppressed rule IDs for that file, reusing the existing relative-path normalization logic.
- `lib/linter/linter.js`: no new suppression logic should be necessary. `verifyAndFix()` already passes `options.fix` through to `SourceCodeFixer.applyFixes()`, so the composed predicate from the higher layers should be enough.

### Example

Suppose `eslint-suppressions.json` contains:

```json
{
  "src/a.js": {
    "semi": {
      "count": 3
    }
  }
}
```

And suppose `src/a.js` currently has:

- 5 `semi` errors
- 2 `indent` errors

With this RFC's behavior:

- `semi` fixes are skipped in `src/a.js`
- `indent` fixes may still be applied in `src/a.js`

## Documentation

This RFC requires updates to:

- [CLI options documentation](https://eslint.org/docs/latest/use/command-line-interface#options): update the `--fix` and `--fix-dry-run` documentation to explain that fixes are skipped for rules suppressed via the suppressions file.
- [Bulk suppressions documentation](https://eslint.org/docs/latest/use/suppressions): explain that suppressions affect autofix eligibility, and that users must remove a suppression entry before intentionally bulk-fixing that rule in that file.
- [Node.js API reference](https://eslint.org/docs/latest/integrate/nodejs-api): document that when `applySuppressions` is enabled, autofix also respects suppressions.

## Drawbacks

The main drawback is that this behavior is intentionally coarse. If a rule is suppressed for a file and a new violation of that rule is later introduced in the same file, ESLint will still skip autofixing that new violation.

Reporting and autofix also use different matching rules. If the current count exceeds the stored suppression count, ESLint reports all violations for that rule in the file, but this RFC would still skip autofixes for that rule in that file.

## Backwards Compatibility Analysis

This proposal changes behavior for autofix runs where bulk suppressions are active. In the CLI, rules suppressed via the suppressions file would no longer be autofixed in those files. In the Node.js API, the same behavior would apply when `applySuppressions` is enabled. This is a behavior change, but it is consistent with how other effectively ignored problems already behave: problems ignored by `eslint-disable` comments are not autofixed, and warnings ignored by `--quiet` are also not autofixed.

Users who want to bulk-fix a suppressed rule can still do so by removing the relevant suppression entry before running autofix.

## Alternatives

### Make the Behavior Opt-In

Another alternative is to keep the current behavior by default and make this opt-in through a CLI flag and a Node.js API option. This was rejected because the current behavior is better understood as unexpected, and the same principle already applies elsewhere in ESLint: effectively ignored lint problems are not autofixed.

## Open Questions

Should ESLint provide any user-facing indication that fixes were skipped because the corresponding file-and-rule pairs are suppressed?

## Help Needed

I can implement this RFC.

## Related Discussions

- eslint/eslint#20062
