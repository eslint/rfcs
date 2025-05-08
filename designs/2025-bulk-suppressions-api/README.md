- Repo: eslint/eslint
- Start Date: 2025-05-06
- RFC PR: https://github.com/eslint/rfcs/pull/133
- Authors: [Kentaro Suzuki](https://github.com/sushichan044)

# Add support for bulk suppressions in `ESLint` and `LegacyESLint` classes

## Summary

<!-- One-paragraph explanation of the feature. -->

This RFC proposes extending the bulk suppressions feature, introduced in ESLint 9.24.0 for CLI usage, to be available through the Node.js API via the `ESLint` and `LegacyESLint` classes.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Currently, the bulk suppression feature introduced in ESLint 9.24.0 is only available via the CLI.
This leads to inconsistencies when ESLint is used programmatically via its Node.js API, such as in IDE integrations.

This leads to inconsistencies when ESLint is used programmatically. Violations suppressed using `eslint-suppressions.json` (especially when using a custom location via the CLI) might not be recognized when using the Node.js API, leading to incorrect error reporting in environments like IDEs.

This RFC aims to resolve this discrepancy by adding support for bulk suppressions to the `ESLint` and `LegacyESLint` Node.js APIs, ensuring consistent linting results and configuration capabilities across both interfaces.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

This proposal integrates the existing bulk suppression functionality into the `ESLint` and `LegacyESLint` Node.js API classes by leveraging the internal `SuppressionsService`. A new `suppressionsLocation` option is introduced in the constructors.

1. New Constructor Option (`suppressionsLocation`)
    - Both `ESLintOptions` and `LegacyESLintOptions` will accept a new optional property: `suppressionsLocation`.
    - `suppressionsLocation: string | undefined`: Specifies the path to the suppressions file (`eslint-suppressions.json`). This path can be absolute or relative to the `cwd`.
    - If `suppressionsLocation` is provided, ESLint will attempt to load suppressions from that specific file path.
    - If `suppressionsLocation` is not provided (or is `undefined`), ESLint will default to looking for `eslint-suppressions.json` in the `cwd`.

2. Service Instantiation and Configuration
    - Upon instantiation, the `ESLint` and `LegacyESLint` constructors will create an instance of `SuppressionsService`.
    - A new constructor option, `suppressionsLocation` (string, optional), will be added to both classes.
    - If provided, this path (relative to `cwd`) specifies the suppression file.
    - If not provided, ESLint defaults to searching for `eslint-suppressions.json` in the `cwd`.
    - The constructor will resolve the final absolute path to the suppression file (using `suppressionsLocation` or the default) and pass it to the `SuppressionsService` constructor.

3. Applying Suppressions
    - Within the `lintFiles()` and `lintText()` methods of both classes, *after* the initial linting results are obtained, the `applySuppressions` method of the instantiated `SuppressionsService` will be called.
    - This method takes the raw linting results and the loaded suppressions (from the resolved file path) and returns the results with suppressions applied, along with any unused suppressions.
    - The final, suppression-applied results will be returned to the user.

4. Changes to ESLint CLI
    - With the integration of suppression handling into the `ESLint` and `LegacyESLint` APIs, the ESLint CLI (`lib/cli.js`) will be updated.
    - Specifically, direct calls to `SuppressionsService` within the CLI will be removed. The CLI will now leverage the updated API methods to handle bulk suppressions, ensuring that the CLI's behavior is consistent with the API's new capabilities. This change simplifies the CLI's implementation by delegating suppression logic to the core API.

### Example code of `lib/eslint/eslint.js`

```javascript
import fs from "node:fs";
import { getCacheFile } from "./eslint-helpers.js";
import { SuppressionsService } from "../services/suppressions-service.js";

class ESLint {
    constructor(options = {}) {
        const processedOptions = processOptions(options);

        // ... existing constructor logic to initialize options, linter, cache, configLoader ...

        const suppressionsFilePath = getCacheFile(
            processedOptions.suppressionsLocation,
            processedOptions.cwd,
            { prefix: "suppressions_" }
         );

        privateMembers.set(this, {
            options: processedOptions,
            linter: /* ... */,
            lintResultCache: /* ... */,
            configLoader: /* ... */
            suppressionsFilePath,
        });
    }

    async lintFiles(patterns) {
        const { options, suppressionsFilePath, lintResultCache, /* ... other needed members */ } = privateMembers.get(this);
        let suppressionResults = null;

        // Existing lint logic to get initial `results` (LintResult[])
        const results = /* LintResult[] */

        // Persist the cache to disk before applying suppressions.
        if (lintResultCache) {
            lintResultCache.reconcile();
        }

        const finalResults = results.filter(result => !!result);

         if (!fs.existsSync(suppressionsFilePath)) {
            return finalResults;
         }

        const suppressions = new SuppressionsService({
            filePath: suppressionsFilePath,
            cwd: options.cwd,
        });

        return suppressions.applySuppressions(
            finalResults,
            await suppressions.load(),
        );
    }

    async lintText(code, options = {}) {
        const {
            options: eslintOptions, // Renamed to avoid conflict with method options
            suppressionsFilePath,
            linter,
            configLoader
        } = privateMembers.get(this);

        const { filePath: providedFilePath, warnIgnored, ...otherOptions } = options || {};

        const results = [];

        // --- Existing lintText logic to determine config, etc. ---

        // Do lint.
        results.push(
            verifyText({
                text: code,
                filePath: resolvedFilename.endsWith("__placeholder__.js")
                    ? "<text>"
                    : resolvedFilename,
                configs,
                cwd,
                fix: fixer,
                allowInlineConfig,
                ruleFilter,
                stats,
                linter,
            }),
        );

        if (!fs.existsSync(suppressionsFilePath)) {
            return processLintReport(this, { results });
        }

        const suppressions = new SuppressionsService({
            filePath: suppressionsFilePath,
            cwd: eslintOptions.cwd,
        });

        const suppressedResults = suppressions.applySuppressions(
            results,
            await suppressions.load(),
        );

        // Return the suppression-applied results
        return processLintReport(this, { results: suppressedResults });
    }
}
```

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The documentation updates will reflect that this change aligns the Node.js API behavior with the existing CLI functionality.

1. API Documentation (`ESLint`/`LegacyESLint` classes):
    - Add the new `suppressionsLocation` option to the constructor options documentation for both `ESLint` and `LegacyESLint`, explaining its purpose (specifying the suppression file path) and behavior (relative to `cwd`, default lookup).
    - Add a note to the descriptions of `lintText()` and `lintFiles()` methods stating that suppressions are automatically applied based on the resolved suppression file path (either from `suppressionsLocation` or the default `eslint-suppressions.json` in `cwd`). Example: "Applies suppressions from the resolved suppression file (`suppressionsLocation` option or `eslint-suppressions.json` in `cwd`), if found."

2. Bulk Suppressions User Guide Page:
    - Update the existing user guide page for Bulk Suppressions.
    - Add a section or note clarifying that the feature is now also available when using the `ESLint` and `LegacyESLint` Node.js APIs.
    - Explicitly mention how the suppression file is located when using the API: "Note: When using the Node.js API, ESLint searches for the suppression file specified by the `suppressionsLocation` constructor option. If this option is not provided, it defaults to looking for `eslint-suppressions.json` in the `cwd` (current working directory)."

3. Release Notes:
    - Include an entry in the release notes announcing the availability of bulk suppressions in the Node.js API, **highlighting the new `suppressionsLocation` option**.

## Drawbacks

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think
    about any opposing viewpoints that reviewers might bring up.

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

- API Complexity: Introduces a new option (`suppressionsLocation`) to the constructor API surface for both `ESLint` and `LegacyESLint`, slightly increasing complexity compared to only supporting the default file location.
- Performance: The overhead of potentially resolving `suppressionsLocation` and then searching for/parsing the suppression file is introduced. However, this aligns the API\'s behavior and capabilities with the CLI.
- Complexity: Introduces `SuppressionsService` interaction into `ESLint`/`LegacyESLint`, but reuses existing internal logic.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

This change is designed to be backward-compatible.

- New Option is Optional: The new `suppressionsLocation` option is optional. Existing code that does not provide this option will continue to work, defaulting to the behavior of looking for `eslint-suppressions.json` in the `cwd`.
- Automatic Application: By integrating bulk suppression handling directly into the existing `lintText()` and `lintFiles()` methods, users who utilize a suppression file (either at the default location or specified via the new option) will automatically benefit simply by updating their ESLint version.
- Alignment with CLI: This approach aligns the Node.js API behavior *and configuration options* more closely with the established CLI behavior.
- Non-Breaking: Since the core change only alters behavior when a suppression file is found (based on the new option or default location), it is considered a non-breaking change for existing API consumers.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

- No `suppressionsLocation` API Option: The initial consideration was to *not* add the `suppressionsLocation` option to the API, only implementing the default lookup in `cwd`. This was simpler but rejected because it lacked full consistency with the CLI\'s `--suppressions-location` flag, preventing API users from specifying a custom file path. Adding the option provides greater flexibility and closer parity with the CLI.
- Separate API Method for Applying Suppressions: Introducing new methods like `lintTextWithSuppressions()` was rejected as inconsistent and burdensome for users compared to automatic application within existing methods.
- Including Suppression File Manipulation Options in `lintText`/`lintFiles`: The CLI includes flags like `--suppress-all`, `--suppress-rule <rule-name>`, and `--prune-suppressions` which generate, update, or prune the suppression file based on lint results. Adding corresponding options to the `lintText` and `lintFiles` API methods was considered. However, this approach was rejected because:
  - It would introduce file-writing side effects into API methods primarily designed for linting (reading and analyzing code). This could lead to unexpected behavior for API consumers.
  - It blurs the responsibility of the `lintText`/`lintFiles` methods.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

- Specific implementation examples for the `LegacyESLint` class.
- Should functionality equivalent to CLI flags like `--suppress-all` or `--suppress-rule` be supported via separate API methods in the future?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I intend to implement this feature.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

Potential questions regarding alternative design approaches are addressed in the `Alternatives` section.

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

### Current Usage of ESLint Node.js API

While improving IDE support is not the primary goal of this RFC, the following investigation provides context on how various tools integrate with ESLint\'s Node.js API.

#### IDE Integrations

- VSCode: Uses [`ESLint.lintText()` / `LegacyESLint.lintText()`](https://github.com/microsoft/vscode-eslint/blob/c0e753713ea9935667e849d91e549adbff213e7e/server/src/eslint.ts#L1192-L1243) for validation.

- Zed: Interacts with the `vscode-eslint` server via `--stdio`. No specific changes needed beyond updating `vscode-eslint`.
  - <https://github.com/zed-industries/zed/blob/d1ffda9bfeccfdf9bea3f76251350bf9cf7f6e1b/crates/languages/src/typescript.rs#L332-L354>
  - <https://github.com/zed-industries/zed/blob/d1ffda9bfeccfdf9bea3f76251350bf9cf7f6e1b/crates/languages/src/typescript.rs#L59-L65>

- JetBrains IDEs: Implementation details are unknown (closed-source).

- Neovim: Integrations like [nvim-eslint](https://github.com/esmuellert/nvim-eslint) typically use `vscode-eslint`. No specific changes needed beyond updating `vscode-eslint`.

#### ESLint Wrappers / Utilities

- xo: Uses [`ESLint.lintFiles()` / `ESLint.lintText()`](https://github.com/xojs/xo/blob/529e6c4ac75f6165044f7ea87bad0b9831803efd/lib/xo.ts#L333-L395) for linting.

- eslint-interactive: Uses [`ESLint.lintFiles()`](https://github.com/mizdra/eslint-interactive/blob/96f3fa34eb6fa150056aa48c0bc2c3e322ef3549/src/core.ts#L85-L93) for linting.
