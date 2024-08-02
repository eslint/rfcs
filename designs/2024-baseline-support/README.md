- Repo: eslint/eslint
- Start Date: 2024-04-20
- RFC PR: (leave this empty, to be filled in later)
- Authors: [Iacovos Constantinou](https://github.com/softius)

# Introduce a way to suppress violations

## Summary

<!-- One-paragraph explanation of the feature. -->

Suppress existing violations, so that they are not being reported in subsequent runs. It allows developers to enable one or more lint rules and be notified only when new violations show up.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Enabling a new lint rule as `error` can be painful when the codebase has many violations and the rule isn't auto-fixable. A good example is [`@typescript-eslint/no-explicit-any`](https://typescript-eslint.io/rules/no-explicit-any/). Unless the rule is enabled during the early stages of the project, it becomes harder and harder to enable it as the codebase grows. Existing violations must be resolved before enabling the rule, but while doing that other violations might creep in.

This can be counterintuitive for enabling new rules as `error`, since the developers need to address the violations before-hand in one way or another. The suggested solution suppress existing violations, allowing the developers to address these at their own pace. It also reports any new violations making it easier to identify and address them.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

To keep track of the all the violations that we would like to suppress, we are storing these into a separate file; A JSON file containing the number of errors that must be ignored for each rule in each file. By design, no violations are suppressed - in other words, this feature doesn't affect existing or new projects, unless the developers explicitly suppress one or more violations.

### File format

The JSON file includes details about the file where the violations found, the rule name and the number of violations. As an example, the following indicates that the file `"src/app/components/foobar/foobar.component.ts"` has one violation for the rule `@typescript-eslint/no-explicit-any` that we want to suppress. All paths are relative to CWD, for portability reasons.

```
{
  "src/app/components/foobar/foobar.component.ts": {
    "@typescript-eslint/no-explicit-any": {
        count: 1
    }
  }
}
```

The file is stored in `.eslint-suppressions.json` , unless otherwise specified.

### Suppressing all violations

A new option `--suppress-all` wil be introduced to ESLint CLI. When provided, the JSON file is generated and saved in `.eslint-suppressions.json`. If the file already exists, it gets over-written. Note that this is a boolean flag option (no values are accepted).

``` bash
eslint --suppress-all ./src
```

### Suppressing violations of a specific rule

A new option `--suppress-rule [RULE1]` will be introduced to ESLint CLI. When provided, the existing suppressions file will be updated to include any existing violation of the provided rule. The suppressions file will be created if not already exists. Note that this is a string flag option (value is required).

``` bash
eslint --suppress-rule '@typescript-eslint/no-explicit-any' ./src
```

### Changing the location of the suppressions file

A new option `--suppression-location` will be introduced to ESLint CLI. When provided, the suppressions file will be loaded and saved to the provided location. Note that this is a string flag option (value is required).

``` bash
eslint --suppress-all --suppressions-location /home/user/project/mycache ./src
```

### Maintaining a lean suppressions file

When working with suppressed violations, there is a chance that a violation is addressed but the suppressions file is not updated. This might allow new violations to creep in without noticing. To ensure that the suppressions file is always up to date, `eslint` can exit with an error code when there are suppressed violations that do not occur anymore.

Consider the following scenario:

* The developer executes `eslint --supress-all ./src` which creates the suppressions file.
* Then `eslint ./src` is executed which reports no violations, with an exit status of 0.
* The developer addresses an error violation. While the violation is addressed is still part of the suppressions file.
* The developer then executes `eslint ./src` again. While it still reports no violations, it exits with a non-zero status code. That is to indicate that the suppressions file needs to be updated.

To address this, a new option `--prune-suggestions` will be introduced to ESLint. Note that this is a boolean flag option (no values are accepted). When provided, violations in suppressions file that no longer occur will be removed, but no new violations will be added (in contrary to when executing `--suppress-all`).

``` bash
eslint --prune-suppressions ./src
eslint --prune-suppressions --suppressions-location /home/user/project/mycache ./src
```

### Execution details

The suggested solution always compares against the suppressions file, given that the file already exists. By default the file is picked up from `.eslint-suppressions.json`, unless the option `--suppressions-location` is provided. This makes it easier for existing and new projects to adopt this feature without the need to adjust scripts in `package.json` and CI/CD workflows. 

This will go through each result item and message, and check each rule giving an error (`severity == 2`) against the suppressions. By design, we do not take warnings into consideration (regardless of whether quite mode is enabled or not), since warnings do not cause eslint to exit with an error code and they already serve a different purpose. If the file and rule are part of the suppressions file, it means that we can remove and ignore the result message. 

To implement this, we will need to adjust mainly `cli.js` to adopt the following operations:

* Check if the `--suppress-all` or `--suppress-rule` option is passed
  * If both are passed exit with an error, since the two options are mutually exclusive.
  * If either option was passed, we need to update the suppressions file based on `results`. 
  * If none option was passed, we need to check if the suppressions file already exists taking `--suppressions-location` into consideration
* Assuming that a suppressions file was found or generated, we need to match the errors found against the violations included in the suppressions file. In particular, for each error found:
  * We need to check whether both file and error are part of the suppressions file.
  * If yes, we reduce the `count` by 1 and ignore the current error. If `count` has already reach zero we keep the error.
  * If no, we keep the error.
* Exit with a non-zero status code if there are unmatched violations. Depending on the verbose mode we can display the list of errors that were left unmatched.
* Otherwise list the remaining errors as usual.

Note that the error detection in `cli.js` happens before the error counting. This allow us to update the suppressions file and modify the errors, before it is time to count errors. Please refer to the last example of the "Implementation notes" for more details.

Furthermore, ESLint cache (`--cache`) must contain the full list of detected violations, even those matched against the suppressions file. This approach has the following benefits:

- Generating the suppressions file can be based on the cache file and should be faster when the cache file is used.
- Allows developers to re-generate the suppressions file or even adjust it manually and re-lint still taking the same cache into consideration.
- It even allows developers to delete the suppressions file and still take advantage of the cached file in subsequent runs. 

# Implementation notes

To introduce the above-mentioned options, we will need to: 

* add the new options in `default-cli-options.js`.
* adjust the config for optionator.
* add the new options as comments and arguments for eslint.
* update documentation to explain the newly introduced option

A new type must be created:

``` js
/**
 * @typedef {Record<string, { count: number }>} SuppressedViolation
 */
```

``` js
class SuppressedViolationsManager {
    /**
     * Creates a new instance of SuppressedViolationsManager.
     * @param {string} suppressionsLocation The location of the suppressions file.
     */
    constructor(suppressionsLocation) {}

    /**
     * Updates the suppressions file based on the current violations
     * 
     * @param {LintResult[]} results The lint results.
     * @returns SuppressedViolation[]
     */
    suppressAll(results) 

    /**
     * Updates the suppressions file based on the current violations and the provided rule
     * 
     * @param {LintResult[]} results The lint results.
     * @param {string} rule The rule to suppress.
     */
    suppressByRule(results, rule)

    /**
     * Removes old suppressions that do not occur anymore.
     * @returns void
     */
    prune()

    /**
     * Checks the provided suppressions against the lint results.
     * It filters and returns the lint results that are not in the suppressions file.
     * It also returns the unmatched suppressions.
     * 
     * @param {LintResult[]} results The lint results.
     * @param {SuppressedViolation[]} suppressions The suppressions.
     * @return {
     *   results: LintResult[],
     *   unmatched: SuppressedViolation[]
     * }
     */
    applySuppressions(results, suppressions)
    
    /**
     * Loads the suppressions file.
     * @returns SuppressedViolation[]
     */
    load()

    /**
     * Updates the suppressions file.
     * @param {SuppressedViolation[]}
     * @returns void
     * @private
     */
    save()
}
```

The resolution of the suppressions file must happen outside of the above class. An idea is to make `getCacheFile` in `lib/eslint/eslint-helpers.js` a bit more abstract so that we can inject the prefix i.e. `.cache_` or `.suppressions_` when a directory is provided. This way both `cache-location` and `suppressions-location` are consistent and following the same pattern.

Once the above are in place, `cli.js` should look something like:

``` js
// ...
if (options.fix) {
    debug("Fix mode enabled - applying fixes");
    await ActiveESLint.outputFixes(results);
}

const suppressionsFileLocation = getCacheFile(options.suppressionsLocation, cwd, 'suppressions_');
if (options.suppressAll || options.suppressRule || options.pruneSuppressions || fs.existsSync(suppressionsFileLocation)) {
    const suppressionsManager = new SuppressedViolationsManager(suppressionsFileLocation);
    if (options.suppressAll) {
        results = suppressionsManager.suppressAll(results);
    } else if (options.suppressRule) {
        results = suppressionsManager.suppressByRule(results, options.suppressRule);
    } else if (options.pruneSuppressions) {
        results = suppressionsManager.prune();
    } else {
        const suppressionResults = suppressionsManager.applySuppressions(results, suppressionsManager.load());
        if (suppressionResults.unmatched.length > 0) {
            // exit with a non-zero code
        }

        results = suppressionResults.results;
    }
}

let resultsToPrint = results;

if (options.quiet) {
    debug("Quiet mode enabled - filtering out warnings");
    resultsToPrint = ActiveESLint.getErrorResults(resultsToPrint);
}

//...
```

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

We should update [Command Line Interface Reference](https://eslint.org/docs/latest/use/command-line-interface) to document the newly introduced options. A dedicated section should be added in Documentation to explain how the new suppression system works.

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

The suggested solution can be used only when linting files. It can not be leveraged when using `stdin` since it relies on file paths.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

If the suppressions file does not exist, ESLint CLI behavior will not change. This change is therefore backwards compatible to start with.

If the suppressions file is already generated, ESLint CLI will compare the errors against the violations included in the suppressions file. Hence it might report less errors than before and someone might argue that this is not backwards compatible since the behavior changes for them. However, as discussed earlier this should facilitate the adoption of the suggested solution without worrying about adjusting scripts in `package.json` and CI/CD workflow. Plus, the suppressions file can be easily deleted and cancel the new behavior.

Furthermore, we are adding one more reason to exit with an error code (see "Maintaining a lean suppressions file"). This might have some negative side-effects to wrapper scripts that assume that error messages are available when that happens. We could introduce a different exit code, to differentiate between exiting due to unresolved errors or ignored errors that do not occur anymore.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

Unfortunately existing approaches do not address the issue at its core and come with their own set of drawbacks. It is worth mentioning that the suggested solution is based on [how baseline works in PHPStan](https://phpstan.org/user-guide/baseline) and bulk suppressions from [@rushstack/eslint-bulk](https://www.npmjs.com/package/@rushstack/eslint-bulk).

The following sections are extracted from [Change Request: Introduce a system to suppress existing errors](https://github.com/eslint/eslint/issues/16755) where [@jfmengels](https://github.com/jfmengels) did a detailed analysis about existing approaches and their drawbacks.

### Using warnings

This use-case is apparently what the "warn" severity level is for.

A large problem with warnings is that as soon as there are more than a few warnings, you don't notice new ones showing up. A common practice I've seen quite often is to avoid warnings altogether, and to only use errors to avoid new problems creeping in. But that doesn't solve the problem of all the existing errors.

Also, users can too easily ignore the new errors, so in a way, the rule is enabled without being enforced when IMO the point of a linter is to enforce rules.

### Using disable comments

One can use disable comments to temporarily suppress errors, by adding a comment like `// eslint-disable rule-name  -- FIX THIS LATER`

"Disable comments" can be used to enable a rule as an error early, by adding them everywhere where an error is currently reported (and that is actually something that can be automated by some linters).

But "disable comments" have the tendency to be hard to distinguish from other "disable comments" created for reasons such as false positives or disagreements on the rule, especially when there is no enforced need to add a message on the comment. Meaning that once you decide to tackle the existing errors, they can be hard to detect (or to distinguish from ones that are disabled for good reasons).

They also "pollute" the codebase in a way that is quite visible, and makes users numb to the fact of using "disable comments".

### Ignoring parts of the project

It is also possible to simply disable the rule in each file that is currently reporting errors, either through manually configuring the rule in the ESLint config, or by adding a disable comment at the top of the file that disables the rule for the entire file.

This has multiple downsides:

* While new errors are enforced in the other files, new errors can creep in the ignored files
* If/when the errors in the ignored files get removed, the user has to remember to re-enable the rule on this file. Otherwise new errors can creep in.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

None so far.

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I expect to implement this change.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

### Does this count warnings?

No, we are only counting errors when updating the suppressions file. Also only errors are considered when checking against the suppressions file.

## Related Discussions

* [Change Request: Introduce a system to suppress existing errors](https://github.com/eslint/eslint/issues/16755)
* [PHPStan - The Baseline](https://phpstan.org/user-guide/baseline)
* [@rushstack/eslint-bulk](https://www.npmjs.com/package/@rushstack/eslint-bulk)

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
