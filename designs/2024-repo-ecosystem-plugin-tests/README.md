- Repo: eslint/eslint
- Start Date: 2024-11-25
- RFC PR: <https://github.com/eslint/rfcs/pull/127>
- Authors: [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Introduce ecosystem tests for popular third-party plugins

## Summary

Adding an CI job to the `eslint/eslint` repo that checks changes against a small selection of third-party plugins.

## Motivation

Changes in ESLint occasionally break downstream plugins in unexpected ways.
Those changes might be unintentional breaking changes, or even non-breaking changes that happen to touch edge case behaviors relied on by plugins.

[Bug: Error while loading rule '@typescript-eslint/no-unused-expressions](https://github.com/eslint/eslint/issues/19134) is an example change in ESLint's that caused downstream breakages in third-party plugins.
At least two popular plugins -[`eslint-plugin-unicorn`](https://github.com/sindresorhus/eslint-plugin-unicorn/issues/2496) and [`typescript-eslint`](https://github.com/typescript-eslint/typescript-eslint/issues/10338)- were broken by that change.

The plugins broke because they were relying on non-public implementation details of ESLint rules per [Docs: Formalize recommendation against plugins calling to rules via use-at-your-own-risk](https://github.com/eslint/eslint/issues/19169).
When the root cause is a bug in the downstream plugins, an "early warning" system would help them fix their issues before the incompatible changes to ESLint are published.

## Detailed Design

### CI Job

The new CI job will, for each plugin:

1. Create a new directory containing a `package.json`, `eslint.config.js`, and small set of files known to be parsed and not cause lint reports with the plugin
2. Run a lint command (i.e. `npx eslint .`) in that directory
3. Assert that the lint command passed with 0 lint reports.

This will all be runnable locally with a `package.json` script like `npm run test:ecosystem --plugin eslint-plugin-unicorn`.

An addition to `.github/workflows/ci.yml` under `jobs` would approximately look like:

```yml
test_ecosystem:
  name: Test Ecosystem Plugins
  runs-on: ubuntu-latest
  strategy:
    matrix:
      plugin:
        - eslint-plugin-unicorn
        - eslint-plugin-vue
        - typescript-eslint
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: "lts/*"
    - name: Install Packages
      run: npm install
    - name: Test ${{ matrix.plugin }}
      run: npm run test:ecosystem --plugin ${{ matrix.plugin }}
```

A `test/ecosystem` directory will be created with a directory for each plugin.
The `test:ecosystem` script will copy the contents of the provided `--plugin` directory into a clean `test/${plugin}-scratch` directory.

Asserting that plugins successfully produce reports will not be part of this job.
Depending on specifics of plugin rule reports would make the job prone to failure on arbitrary plugin rule updates.

### Failure Handling

It is theoretically possible that the ecosystem CI job will occasionally be broken by updates to ecosystem plugins.
However, this RFC believes that case will be exceedingly rare and short-lived:

- Per [Plugin Selection](#plugin-selection), only very stable plugins that test on multiple ESLint versions including the latest will be selected
- Today, plugin breakages are typically resolved within a week - even without this RFC's proposed "early warning" detection
  - Example: [typescript-eslint#10191](https://github.com/typescript-eslint/typescript-eslint/issues/10338) was reported on October 21st, 2024 and a fix published on October 28th, 2024
  - Example: [typescript-eslint#10338](https://github.com/typescript-eslint/typescript-eslint/issues/10338) was reported on November 15th, 2024 and a fix published on November 18th, 2024
  - Example: [eslint-plugin-unicorn#10191](https://github.com/sindresorhus/eslint-plugin-unicorn/issues/2496) was reported on November 15th, 2024 and a fix published on November 19th, 2024

In the case of a breakage being discovered on the `main` branch, this RFC proposes the following process:

1. An ESLint team member should file a bug report on the plugin's repository -if it doesn't yet exist-, as well as an issue on `eslint/eslint` linking to that bug report
2. If the issue isn't resolved within two weeks:
   1. An ESLint team member should send a PR to resolve the issue that removes the plugin from ESLint's ecosystem CI job
   2. An ESLint team member should file a followup issue to re-add it once the breakage is fixed

In the case of a breaking being discovered on a PR branch, this RFC proposes the following process:

1. If the failure is an indication of an issue in the PR, the PR should be updated as usual
2. Otherwise, if the failure is an indication the plugin needs to be updated, the PR's author should file a bug report on the plugin's repository - if it doesn't yet exist
3. If the issue isn't resolved within two weeks:
   1. The PR's author should remove the plugin from ESLint's ecosystem CI job in the PR
   2. The PR's author should file a followup issue to re-add it once the breakage is fixed

### Major Releases

Upcoming new major versions of ESLint are an expected failure case for ecosystem plugins.
The ecosystem CI job will skip running any plugin that doesn't explicitly support the version of ESLint being tested.

Plugin version support will be determined by the maximum `eslint` peer dependency range in the plugin's published `package.json`, if it exists.
Otherwise the ESLint repository will assume only supporting up to the currently stable version of ESLint.

### Plugin Selection

The plugins that will be included to start will be:

- [`eslint-plugin-eslint-comments`](https://github.com/eslint-community/eslint-plugin-eslint-comments): to capture an `eslint-community` project and AST edge cases around comments
- [`eslint-plugin-unicorn`](https://github.com/sindresorhus/eslint-plugin-unicorn): to capture a large selection of miscellaneous rules
- [`eslint-plugin-vue`](https://github.com/vuejs/eslint-plugin-vue): to capture support for a framework with nested parsing of a non-JavaScript/TypeScript-standard syntax
- [`typescript-eslint`](https://github.com/typescript-eslint/typescript-eslint): to capture testing TypeScript APIs and intricate uses of parsing in general

Plugins will be selectively added if they meet the following criteria:

- &gt;1 million npm downloads a week: arbitrary large size threshold to avoid small packages
- Adding a notable new API usage not yet covered: to avoid duplicate equivalent plugins
- Has had a breakage reported on ESLint: to be cautious in adding to the list
- Is under active maintenance and has taken a week or less to fix any ESLint breakages within the last year: to avoid packages that won't be updated quickly on failures

The number of plugins should remain small.
Each added plugin brings adds the risk of third-party breakage, so plugins will only be added after filing a new issue and gaining team consensus.

### Rollout

This RFC expects the added ecosystem CI job to _likely_ consistently pass.
However, to be safe, this RFC proposes adding a CI job in three steps:

1. On a branch that and updated from `main` several times a week
2. On the `main` branch only
3. On all PRs targeting the `main` branch, alongside existing CI jobs

At least one month should be held between steps to make sure the job is consistently passing.

## Out of Scope

Automation could be added for at least the filing of issues on plugin failures.
That does not seem worth the time expenditure given how rarely plugins are expected to fail.
This RFC's discussion settled on it not being worth it.

## Open Questions

Are there other plugins we should include that satisfy the criteria?

## Help Needed

I expect to implement this change.

## Frequently Asked Questions

### Given ESLint respects semver, why add tests for plugins that are relying on internals?

It's exceedingly difficult to be sure when changes to a large published package break contracts with downstream consumers.
Even when all packages in an ecosystem are well-tested the way ESLint and its major plugins are, the sheer project size and duration of maintenance make unfortunate edge cases likely to happen.

> [Venerable xkcd "Workflow" comic](https://xkcd.com/1172)

## Related Discussions

- [Repo: add end-to-end/integration tests for popular 3rd party plugins](https://github.com/eslint/eslint/issues/19139)
