- Repo: eslint/eslint
- Start Date: 2025-05-06
- RFC PR: <https://github.com/eslint/rfcs/pull/142>
- Authors: [Kentaro Suzuki](https://github.com/sushichan044) [Blake Sager](https://github.com/adf0nt3s)

# Consider bulk suppressions when running Lint via the Node.js API

## Summary

This RFC proposes integrating bulk suppressions support into the Node.js API via the `ESLint` class, specifically focusing on considering existing bulk suppressions when linting files or text through the API. This change ensures that suppression files (`eslint-suppressions.json`) created via CLI commands are automatically respected when using the programmatic API, maintaining consistency between CLI and API behavior.

The scope is limited to applying existing suppressions during linting and does not include suppression file manipulation features (such as `--suppress-all`, `--suppress-rule`, or `--prune-suppressions`), which remain CLI-exclusive functionalities.

## Motivation

Currently, the bulk suppression feature introduced in ESLint 9.24.0 is only available via the CLI.

This leads to inconsistencies when ESLint is used programmatically via its Node.js API, such as in IDE integrations. Violations suppressed using `eslint-suppressions.json` (especially when using a custom location via the CLI) might not be recognized when using the Node.js API, leading to incorrect error reporting in environments like IDEs.

This RFC aims to resolve this discrepancy by adding support for bulk suppressions to the `ESLint` Node.js API, ensuring consistent linting results and configuration capabilities across both interfaces.

## Detailed Design

This proposal integrates the existing bulk suppression functionality into the `ESLint` Node.js API class by leveraging the internal `SuppressionsService`. A new `suppressionsLocation` option is introduced in the constructor.

1. New Constructor Options 
    - `suppressionsLocation`
        - `ESLintOptions` will accept a new optional property: `suppressionsLocation`.
        - `suppressionsLocation: string | undefined`: Specifies the path to the suppressions file (`eslint-suppressions.json`). This path can be absolute or relative to the `cwd`.
        - If `suppressionsLocation` is provided, ESLint will attempt to load suppressions from that specific file path.
        - If `suppressionsLocation` is not provided (or is `undefined`), ESLint will default to looking for `eslint-suppressions.json` in the `cwd`.
    - `applySuppressions`
        - Controls whether suppressions are automatically applied to results from `lintText()` and `lintFiles()`.
        - If `true`, suppressions are automatically applied to lint results before returning.
        - If `false`, results are returned as-is without applying suppressions.
        - If not provided (or is `undefined`), defaults to `false`. (This is to preserve backwards compatibility for existing API users who may not expect suppressions to be applied automatically. We may consider changing this default to `true` in a future major release.)

2. Service Instantiation and Configuration
    - Upon instantiation, the `ESLint` constructor will create an instance of `SuppressionsService`.
    - 2 new constructor options will be added to the `ESLint` class.
        - `suppressionsLocation` (string, optional)
            - If provided and relative, this path (relative to `cwd`) specifies the suppression file.
            - If provided and absolute, this path (relative to `/`) specifies the suppression file.
            - If not provided, ESLint defaults to searching for `eslint-suppressions.json` in the `cwd`.
            - The constructor will resolve the final absolute path to the suppression file (using `suppressionsLocation` or the default) and pass it to the `SuppressionsService` constructor.
        - `applySuppressions` (boolean, optional)
            - If `true`, suppressions will be applied automatically in `lintText()` and `lintFiles()`.
            - If `false`, suppressions will not be applied.
            - If not provided, defaults to `false`.

3. Applying Suppressions
    - When `applySuppressions` is `true`, the `lintFiles()` and `lintText()` methods will automatically apply suppressions to results before returning.
    - Within these methods, *after* the initial linting results are obtained, the `applySuppressions` method of the instantiated `SuppressionsService` will be called.
    - This method takes the raw linting results and the loaded suppressions (from the resolved file path) and returns the results with suppressions applied.
    - When caching is enabled, the **original lint results** are stored in the cache. Suppressions are applied after cache retrieval, so that changes to the suppression file take effect without needing to bust the cache.
    - When `applySuppressions` is `false`, the raw linting results are returned without applying suppressions, allowing consumers to handle suppressions themselves.

4. Changes to ESLint CLI
    - The CLI will instantiate `ESLint` with `applySuppressions: false` to receive raw linting results.
    - The CLI will continue to call `SuppressionsService` directly to handle suppression-related flags (`--suppress-all`, `--suppress-rule`, `--prune-suppressions`) and to apply suppressions before output.

### Example code of `lib/eslint/eslint.js`

```javascript
import fs from "node:fs";
import { getCacheFile } from "./eslint-helpers.js";
import { SuppressionsService } from "../services/suppressions-service.js";

class ESLint {
    /**
    * The suppressions service to use for suppressing messages.
    * @type {SuppressionsService}
    */
    #suppressionsService;

    constructor(options = {}) {
        // options includes: suppressionsLocation, applySuppressions, cwd, ...
        const processedOptions = processOptions(options);

        // ... existing constructor logic to initialize options, linter, cache, configLoader ...

        const suppressionsFilePath = getCacheFile(
            processedOptions.suppressionsLocation,
            processedOptions.cwd,
            { prefix: "suppressions_" }
         );

        this.#suppressionsService = new SuppressionsService({
            filepath: suppressionsFilePath,
            cwd: processedOptions.cwd,
        })
    }

    async lintFiles(patterns) {
        const { options, lintResultCache, /* ... other needed members */ } = privateMembers.get(this);

        const { filePath...otherOptions } = options || {};

        // Existing lint logic to get initial `results` (LintResult[])
        const results = /* LintResult[] */

        // Persist the cache to disk before applying suppressions.
        if (lintResultCache) {
            lintResultCache.reconcile();
        }

        const finalResults = results.filter(result => !!result);

        if (options.applySuppressions === false) {
            return finalResults;
        }

        return this.#suppressionsService.applySuppressions(
            finalResults,
            await this.#suppressionsService.load(),
        );
    }

    async lintText(code, options = {}) {
        const {
            options: eslintOptions, // Renamed to avoid conflict with method options
            linter,
            configLoader
        } = privateMembers.get(this);

        const { filePath, warnIgnored, ...otherOptions } = options || {};

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

        if (!filePath || eslintOptions.applySuppressions === false) {
            return processLintReport(this, { results });
        }

        const suppressedResults = this.#suppressionsService.applySuppressions(
            results,
            await this.#suppressionsService.load(),
        );

        return processLintReport(this, { results: suppressedResults });
    }
}
```

## Documentation

The documentation updates will reflect that this change aligns the Node.js API behavior with the existing CLI functionality.

1. API Documentation (`ESLint` class):
    - Add the new `suppressionsLocation` and `applySuppressions` options to the constructor options documentation for `ESLint`.
    - Document `suppressionsLocation`: its purpose (specifying the suppression file path) and behavior (relative to `cwd`, default lookup).
    - Document `applySuppressions`: controls whether suppressions are automatically applied (defaults to `true`).
    - Add a note to the descriptions of `lintText()` and `lintFiles()` methods stating that suppressions are automatically applied when `applySuppressions` is `true` (the default), based on the resolved suppression file path.

2. Bulk Suppressions User Guide Page:
    - Update the existing user guide page for Bulk Suppressions.
    - Add a section or note clarifying that the feature is now also available when using the `ESLint` Node.js API.
    - Explicitly mention how the suppression file is located when using the API: "Note: When using the Node.js API, ESLint searches for the suppression file specified by the `suppressionsLocation` constructor option. If this option is not provided, it defaults to looking for `eslint-suppressions.json` in the `cwd` (current working directory)."

3. Release Notes:
    - Include an entry in the release notes announcing the availability of bulk suppressions in the Node.js API, **highlighting the new `suppressionsLocation` and `applySuppressions` options**.

## Drawbacks

- API Complexity: Introduces two new options (`suppressionsLocation` and `applySuppressions`) to the constructor API surface for `ESLint`, slightly increasing complexity.
- Performance: The overhead of potentially resolving `suppressionsLocation` and then searching for/parsing the suppression file is introduced. However, this aligns the API\'s behavior and capabilities with the CLI.
- Complexity: Introduces `SuppressionsService` interaction into `ESLint`, but reuses existing internal logic.

## Backwards Compatibility Analysis

This change is designed to be backward-compatible.

- New Options are Optional: The new `suppressionsLocation` and `applySuppressions` options are optional. Existing code that does not provide these options will continue to work, with `applySuppressions` defaulting to `false` and `suppressionsLocation` defaulting to looking for `eslint-suppressions.json` in the `cwd`.
- Automatic Application: By integrating bulk suppression handling directly into the existing `lintText()` and `lintFiles()` methods, users who utilize a suppression file (either at the default location or specified via the new option) will automatically benefit simply by updating their ESLint version.
- Alignment with CLI: This approach aligns the Node.js API behavior *and configuration options* more closely with the established CLI behavior.
- Non-Breaking: Since the core change only alters behavior when a suppression file is found (based on the new option or default location), it is considered a non-breaking change for existing API consumers.

## Alternatives

- No `suppressionsLocation` API Option: The initial consideration was to *not* add the `suppressionsLocation` option to the API, only implementing the default lookup in `cwd`. This was simpler but rejected because it lacked full consistency with the CLI\'s `--suppressions-location` flag, preventing API users from specifying a custom file path. Adding the option provides greater flexibility and closer parity with the CLI.
- Separate API Method for Applying Suppressions: Introducing new methods like `lintTextWithSuppressions()` was rejected as inconsistent and burdensome for users compared to automatic application within existing methods.
- Including Suppression File Manipulation Options in `lintText`/`lintFiles`: The CLI includes flags like `--suppress-all`, `--suppress-rule <rule-name>`, and `--prune-suppressions` which generate, update, or prune the suppression file based on lint results. Adding corresponding options to the `lintText` and `lintFiles` API methods was considered. However, this approach was rejected because:
  - It would introduce file-writing side effects into API methods primarily designed for linting (reading and analyzing code). This could lead to unexpected behavior for API consumers.
  - It blurs the responsibility of the `lintText`/`lintFiles` methods.

## Open Questions

- Should functionality equivalent to CLI flags like `--suppress-all` or `--suppress-rule` be supported via separate API methods in the future?

## Help Needed

I intend to implement this feature.

## Related Discussions

* Original RFC PR: https://github.com/eslint/rfcs/pull/133/files

### Current Usage of ESLint Node.js API

While improving IDE support is not the primary goal of this RFC, the following investigation provides context on how various tools integrate with ESLint\'s Node.js API.

#### IDE Integrations

- VSCode: Uses [`ESLint.lintText()`](https://github.com/microsoft/vscode-eslint/blob/c0e753713ea9935667e849d91e549adbff213e7e/server/src/eslint.ts#L1192-L1243) for validation.

- Zed: Interacts with the `vscode-eslint` server via `--stdio`. No specific changes needed beyond updating `vscode-eslint`.
  - <https://github.com/zed-industries/zed/blob/d1ffda9bfeccfdf9bea3f76251350bf9cf7f6e1b/crates/languages/src/typescript.rs#L332-L354>
  - <https://github.com/zed-industries/zed/blob/d1ffda9bfeccfdf9bea3f76251350bf9cf7f6e1b/crates/languages/src/typescript.rs#L59-L65>

- JetBrains IDEs: Implementation details are unknown (closed-source).

- Neovim: Integrations like [nvim-eslint](https://github.com/esmuellert/nvim-eslint) typically use `vscode-eslint`. No specific changes needed beyond updating `vscode-eslint`.

#### ESLint Wrappers / Utilities

- xo: Uses [`ESLint.lintFiles()` / `ESLint.lintText()`](https://github.com/xojs/xo/blob/529e6c4ac75f6165044f7ea87bad0b9831803efd/lib/xo.ts#L333-L395) for linting.

- eslint-interactive: Uses [`ESLint.lintFiles()`](https://github.com/mizdra/eslint-interactive/blob/96f3fa34eb6fa150056aa48c0bc2c3e322ef3549/src/core.ts#L85-L93) for linting.
