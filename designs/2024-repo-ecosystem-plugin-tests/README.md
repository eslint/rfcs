- Repo: eslint/eslint
- Start Date: 2024-11-25
- RFC PR: <https://github.com/eslint/rfcs/pull/127>
- Authors: [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Introduce ecosystem tests for popular plugins

## Summary

Adding an CI job to the `eslint/eslint` repository that checks changes against `@eslint/*` plugins as well as a small selection of third-party plugins.

## Motivation

Changes in ESLint occasionally break downstream plugins in unexpected ways.
Those changes might be unintentional breaking changes, or even non-breaking changes that happen to touch edge case behaviors relied on by plugins.

[Bug: Error while loading rule '@typescript-eslint/no-unused-expressions'](https://github.com/eslint/eslint/issues/19134) reports an example change in ESLint that caused downstream breakages in third-party plugins.
At least two popular plugins -[`eslint-plugin-unicorn`](https://github.com/sindresorhus/eslint-plugin-unicorn/issues/2496) and [`typescript-eslint`](https://github.com/typescript-eslint/typescript-eslint/issues/10338)- were broken by that change.

The plugins broke because they were relying on non-public implementation details of ESLint rules per [Docs: Formalize recommendation against plugins calling to rules via use-at-your-own-risk](https://github.com/eslint/eslint/issues/19169).
ESLint core's [`eslint-config-eslint`](https://github.com/eslint/eslint/tree/main/packages/eslint-config-eslint) does not use all rules of downstream plugins and is not always up-to-date with their latest versions, so its internal usage of plugins is not sufficient to flag all high visibility compatibility issues.
When the root cause is a bug in the downstream plugins, an "early warning" system would help them fix their issues before the incompatible changes to ESLint are published.

## Detailed Design

This RFC proposes creating a small list of popular third-party plugins that will be tested as part of ESLint's CI.
Each plugin will have a `test:eslint-compat` script in their `package.json` that runs lint rule tests.

See [Plugin Selection](#plugin-selection) below for specifics on which plugins will be included.

> ⚠️ Plugins are currently being asked for feedback on the `test:eslint-compat` script.

### CI Job

The new CI job will, for each plugin:

1. Clone the plugin into a directory named `test/ecosystem/${plugin}`
2. Run the plugin's package installation and build commands with [ni](https://github.com/antfu-collective/ni)
3. Run the plugin's `test:eslint-compat` script with [ni](https://github.com/antfu-collective/ni)

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

For now, it is assumed each plugin that needs to be built before testing does so with a script named `build`.
The CI job could be given overrides in the `matrix.plugin` to override the name of the builder script(s) as needed.

### Failure Handling

It is theoretically possible that the ecosystem CI job will occasionally be broken by updates to plugins.
However, this RFC believes that case will be exceedingly rare and short-lived:

- Per [Plugin Selection](#plugin-selection), only very stable plugins that test on multiple ESLint versions including the latest will be selected
- Today, plugin breakages are typically resolved within a week - even without this RFC's proposed "early warning" detection
  - Example: [typescript-eslint#10191](https://github.com/typescript-eslint/typescript-eslint/issues/10191) was reported on October 21st, 2024 and a fix published on October 28th, 2024
  - Example: [typescript-eslint#10338](https://github.com/typescript-eslint/typescript-eslint/issues/10338) was reported on November 15th, 2024 and a fix published on November 18th, 2024
  - Example: [eslint-plugin-unicorn#2496](https://github.com/sindresorhus/eslint-plugin-unicorn/issues/2496) was reported on November 15th, 2024 and a fix published on November 19th, 2024

If a breakage occurs on the `main` branch of ESLint, it will be assumed a plugin has introduced a compatibility bug and should be fixed.
This RFC proposes the following process:

1. An ESLint team member should file issues tracking fixing the breakage:
   - A bug report on the plugin's repository if it doesn't yet exist
   - An issue on `eslint/eslint` linking to that bug report
2. If the issue isn't resolved by the next day, an ESLint team member should:
   1. Send a PR to the ESLint repository to remove the plugin from ESLint's ecosystem CI job
   2. File a followup issue to re-add it once the breakage is fixed

In the case of a breakage being discovered on a PR branch, this RFC proposes the following process:

1. If the failure is an indication of an issue in the PR, the PR should be updated as usual
2. Otherwise, if the failure is an indication the plugin needs to be updated, the PR's author should drive filing issues to update the plugin:
   1. The PR author should file a bug report on the plugin's repository - if it doesn't yet exist
   2. If the issue isn't resolved within two weeks:
      1. The PR's author should remove the plugin from ESLint's ecosystem CI job in the PR
      2. The PR's author should file a followup issue on ESLint, initially labeled as `blocked`, to re-add it once the breakage is fixed
      3. Once the breakage is fixed, a team member should replace the issue's `blocked` label with `accepted`

### Major Releases

Upcoming new major versions of ESLint are an expected failure case for ecosystem plugins.
The ecosystem CI job will skip running any plugin that doesn't explicitly support the version of ESLint being tested.

Plugin version support will be determined by the maximum `eslint` peer dependency range in the plugin's published `package.json`, if it exists.
Otherwise the ESLint repository will assume only supporting up to the currently stable version of ESLint.

### Plugin Selection

The plugins that will be included to start will be:

- All `@eslint/*` plugins, including [`@eslint/css`](https://www.npmjs.com/package/@eslint/css), [`@eslint/json`](https://www.npmjs.com/package/@eslint/json), and [`@eslint/markdown`](https://www.npmjs.com/package/@eslint/markdown)
- [`eslint-plugin-eslint-comments`](https://github.com/eslint-community/eslint-plugin-eslint-comments): to capture an `eslint-community` project and AST edge cases around comments
- [`eslint-plugin-unicorn`](https://github.com/sindresorhus/eslint-plugin-unicorn): to capture a large selection of miscellaneous rules
- [`eslint-plugin-vue`](https://github.com/vuejs/eslint-plugin-vue): to capture support for a framework with nested parsing of a non-JavaScript/TypeScript-standard syntax
- [`typescript-eslint`](https://github.com/typescript-eslint/typescript-eslint): to capture testing TypeScript APIs and intricate uses of parsing in general

Third-party plugins will be selectively added if they meet all of the following criteria:

- &gt;1 million npm downloads a week: arbitrary large size threshold to avoid small packages
- Adding a notable new API usage not yet covered: to avoid duplicate equivalent plugins
- Has had a breakage reported on ESLint: to be cautious in adding to the list
- Is under active maintenance and has taken a week or less to fix any ESLint breakages within the last year: to avoid packages that won't be updated quickly on failures
- Add a `test:eslint-compat` script that exclusively runs lint rule tests

The number of third-party plugins should remain small.
Each added plugin adds a risk of breakage, so plugins will only be added after filing a new issue and gaining team consensus.

### Rollout

This RFC expects the added ecosystem CI job to _likely_ consistently pass.
A CI job will be added to the `eslint/eslint` repository, but will not immediately be a part of `main` branch or PR branch builds.
To be safe, this RFC proposes rolling out CI job in three steps:

1. On a CI cron job once a day, targeting the `main` branch but not blocking its builds
2. On the `main` branch only, with failures showing as failures in its builds
3. On all PRs targeting the `main` branch, alongside existing CI jobs

Each step will replace the previous step.
Once all three are done, running ecosystem tests will be a standard part of `main` branch and pull request CI along with existing tasks like linting and testing.

Starting with a job separately from `main` ensures that unexpectedly high frequencies of breakages are caught early, without blocking `main` branch builds.
At least one month should be held between those steps to make sure the job is consistently passing.

## Out of Scope

Automation could be added for at least the filing of issues on plugin failures.
That does not seem worth the time expenditure given how rarely plugins are expected to fail.
This RFC's discussion settled on it not being worth it.

Plugins using internal/private ESLint APIs are one of the canonical examples of what this process is meant to flag.
However, this process intentionally does not include processes for making code changes in those plugins.
For downstream repositories, this process only proposes how the ESLint team or PR contributors may file issues.
This RFC's intent is that those repositories will drive changing their uses of ESLint APIs.

## Open Questions

Are there other plugins we should include that satisfy the criteria?

## Help Needed

I expect to implement this change.

## Frequently Asked Questions

### Given ESLint respects semver, why add tests for plugins that are relying on internals?

It's exceedingly difficult to be sure when changes to a large published package break contracts with downstream consumers.
Even when all packages in an ecosystem are well-tested the way ESLint and its major plugins are, the sheer project size and duration of maintenance make unfortunate edge cases likely to happen.

> [Venerable xkcd "Workflow" comic](https://xkcd.com/1172)

### What if a breakage causes rules to report incorrectly, but doesn't cause `npm test:eslint-compat` to crash?

Checking for incorrect rule reports is not handled by this RFC.
All recent significant downstream breakages caused rules to fully crash.

Any kind of rule report verification would necessitate ecosystem tests taking a dependency on the specific reports from downstream plugins.
This RFC does not believe the effort of keeping snapshots of those reports up-to-date as worthwhile.

## Related Discussions

- [Repo: add end-to-end/integration tests for popular 3rd party plugins](https://github.com/eslint/eslint/issues/19139)
