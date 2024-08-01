- Repo: eslint/eslint
- Start Date: 2024-04-20
- RFC PR: (leave this empty, to be filled in later)
- Authors: [Iacovos Constantinou](https://github.com/softius)

# Introduce baseline system to suppress existing errors

## Summary

<!-- One-paragraph explanation of the feature. -->

Declare currently reported errors as the "baseline", so that they are not being reported in subsequent runs. It allows developers to enable one or more rules as `error` and be notified when new ones show up.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

Enabling a new lint rule as `error` can be painful when the codebase has many violations and the rule isn't auto-fixable. A good example is [`@typescript-eslint/no-explicit-any`](https://typescript-eslint.io/rules/no-explicit-any/). Unless the rule is enabled during the early stages of the project, it becomes harder and harder to enable it as the codebase grows. Existing violations must be resolved before enabling the rule, but while doing that other violations might creep in.

This can be counterintuitive for enabling new rules as `error`, since the developers need to address the violations before-hand in one way or another. The suggested solution suppress existing violations, allowing the developers to address these at their own pace. It also reports any new violations making it easier to identify and address future violations.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

To keep track of the all the errors that we would like to ignore, we are introducing the concept of the baseline file; A JSON file containing the number of errors that must be ignored for each rule in each file. By design, the baseline is disabled by default and it doesn't affect existing or new projects, unless the baseline file is generated.

### Baseline format

The baseline files includes details about the file where the error found, the rule name and the number of errors. Here is what the baseline file looks like. This indicates that the file `"src/app/components/foobar/foobar.component.ts"` has one error for the rule `@typescript-eslint/no-explicit-any` that is acceptable to be ignored. All paths are relative to CWD, for portability reasons.

```
{
  "src/app/components/foobar/foobar.component.ts": {
    "@typescript-eslint/no-explicit-any": {
        count: 1
    }
  }
}
```

### Generating the baseline

A new option `--baseline` wil be introduced to ESLint CLI. When provided, the baseline is generated and saved in `.eslint-baseline.json`. If the file already exists, it gets over-written. Note that this is a boolean flag option (no values are accepted). Here is an example how to generate the baseline:

``` bash
eslint --baseline ./src
```

The above goes through each result item and messages, and counts the number of errors (`severity == 2`). The necessary details are stored in the baseline file. By default, the baseline file is saved at `.eslint-baseline.json` . To control where the baseline is saved, another option will be introduced `--baseline-location`. That is a string argument specifying where the baseline must be saved. Here is an example 
how to generate the baseline and save the results to `/home/user/project/mycache`.

``` bash
eslint --baseline --baseline-location /home/user/project/mycache
```

To introduce the above-mentioned options, we will need to: 

 * add the new options in `default-cli-options.js`.
 * adjust the config for optionator.
 * add the new options as comments and arguments for eslint.
 * update documentation to explain the newly introduced options.

On top of that, we will need to adjust `cli.js` to check if `--baseline` was provided and set to true, right after the fixes are done and the warnings are filtered out, to avoid counting errors that are about to be fixed. A new method `generateBaseline` will be introduced, which must be called only and only if `--baseline` was provided and set to true. Please refer to "Implementation Details" for more details on this.

### Matching against the baseline

The suggested solution always compares against the baseline, given that the baseline file exists. By default the baseline file is picked up from `.eslint-baseline.json`, unless the option `--baseline-location` is provided. This makes it easier for existing and new projects to adopt the baseline without the need to adjust scripts in `package.json` and CI/CD workflows. As mentioned above, `--baseline-location` is a string argument specifying where the baseline was previously saved.

This will go through each result item and message, and check each rule giving an error (`severity == 2`) against the baseline. By design, we do not take warnings into consideration (regardless of whether quite mode is enabled or not), since warnings do not cause eslint to exit with an error code and they already serve a different purpose. If the file and rule are part of the baseline, means that we can remove and ignore the result message. 

To implement this, we will need to adjust further `cli.js` to adopt the following operations:

* Check if the `--baseline` option is passed
> * If yes, we need to generate the baseline based on `resultsToPrint`. 
> * If no, we need to check if the baseline file already exists taking `--baseline-location` into consideration
* Assuming that a baseline was found or generated, we need to match the errors found against the baseline. In particular, for each error found:
> * We need to check whether both file and error are part of the baseline
> * If yes, we reduce the `count` by 1 and ignore the current error. If `count` has already reach zero we keep the error.
> * If no, we keep the error.

Note that `cli.js` the error detection in `cli.js` happens quite earlier before the error counting. This allow us to create the baseline, before it is time to count errors. Please refer to the last example of the "Implementation details".

We can also keep track of which errors from baseline were not matched, that is useful for the next section.

### Maintaining a lean baseline

When using the baseline, there is a chance that an error is resolved but the baseline file is not updated. This might allow new errors to creep in without noticing. To ensure that the baseline is always up to date, eslint can exit with an error code when there are ignored errors that do not occur anymore. To fix this, the developer can regenerate the baseline file.

Consider the following scenario:

* The developer executes `eslint --baseline ./src` which updates the baseline file.
* Then `eslint ./src` is executed which gives no violations, with an exit status of 0.
* The developer addresses an error violation. While the error is addressed is still part of the baseline.
* The developer then executes `eslint ./src` again. While it still gives no violations, it exits with a non-zero status code. That is to indicate that the baseline needs to be re-generated by executing `eslint --baseline ./src`.

To implement this, we will need to extend `applyBaseline` to return the unmatched rules. In particular, we will need to check if one or more rules are left unmatched, and exit with an error code. Depending on the verbose mode we can display the list of errors that were left unmatched.

### ESLint Cache

ESLint cache (`--cache`) must contain the full list of detected errors, even those matched against the baseline. This approach has the following benefits:

- Generating the baseline can be based on the cache file and should be faster when the cache file is used.
- Allows developers to re-generate the baseline or even adjust it manually and re-lint still taking the same cache into consideration.
- It even allows developers to delete the baseline and still take advantage of the cached file in subsequent runs. 

### Implementation Details

First of all we need to introduce a new type to hold the individual baseline result:

``` js
/**
 * @typedef {Record<string, { count: number }>} BaselineResult
 */
```

One way to approach this is to introduce a Manager class handling the baseline results - in particular, both generating a baseline and validating the results against a baseline.

``` js
class BaselineResultManager {
    /**
     * Creates a new instance of BaselineResultManager.
     * @param {string} baselineLocation The location of the baseline file.
     */
    constructor(baselineLocation) {}

    /**
     * Generates the baseline from the provided lint results.
     * @param {LintResult[]} results The lint results.
     * @returns BaselineResult[]
     */
    generateBaseline(results) 

    /**
     * Checks the provided baseline results against the lint results.
     * It filters and returns the lint results that are not in the baseline.
     * It also returns the unmatched baseline results.
     * 
     * @param {LintResult[]} results The lint results.
     * @param {BaselineResult[]} baseline The baseline.
     * @return {
     *   results: LintResult[],
     *   unmatched: BaselineResult[]
     * }
     */
    applyBaseline(results, baseline)
    
    /**
     * Loads the baseline from the file.
     * @returns BaselineResult[]
     */
    loadBaseline()

    /**
     * Saves the baseline to the file.
     * @param {BaselineResult[]}
     * @returns void
     * @private
     */
    saveBaseline(baseline)
}
```

The resolution of the baseline location must happen outside of the above class. An idea is to make `getCacheFile` in `lib/eslint/eslint-helpers.js` a bit more abstract so that we can inject the prefix i.e. `.cache_` or `.baseline` when a directory is provided. This way both `cache-location` and `baseline-location` are consistent and following the same pattern.

Once the above are in place, `cli.js` should look something like:

``` js
// ...
const { baseline } = require("../conf/default-cli-options");
// ...
if (options.quiet) {
    debug("Quiet mode enabled - filtering out warnings");
    resultsToPrint = ActiveESLint.getErrorResults(resultsToPrint);
}

const baselineFileLocation = getCacheFile(baseline, cwd, 'baseline_');
if (options.baseline || fs.existsSync(baseline)) {
    const baselineManager = new BaselineResultManager(baselineFileLocation);
    let loadedBaselineRecords = [];
    if (options.baseline) {
        loadedBaselineRecords = baselineManager.generateBaseline(resultsToPrint);
    } else {
        loadedBaselineRecords = baselineManager.loadBaseline();
    }

    const baselineResults = await baselineManager.applyBaseline(resultsToPrint, loadedBaselineRecords);

    if (baselineResults.unmatched.length > 0) {
        // exit with errors
    }

    resultsToPrint = baselineResults.results;
}

const resultCounts = countErrors(results);
//...
```

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

We should update [Command Line Interface Reference](https://eslint.org/docs/latest/use/command-line-interface) to document the newly introduced option. A dedicated section should be added in Documentation to explain how baseline works.

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

The baseline can be generated and used only when linting files. It can not be leveraged when using `stdin` since it relies on file paths.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

If the baseline file is not generated, ESLint CLI behavior will not change. This change is therefore backwards compatible to start with.

If the baseline file is generated, ESLint CLI will compare the errors against the errors included in the baseline. Hence it might report less errors than before and someone might argue that this is not backwards compatible since the behavior changes for them. However, as discussed earlier this should facilitate the adoption of the baseline without worrying about adjusting scripts in `package.json` and CI/CD workflow. Plus, the baseline can be easily deleted and cancel the new behavior.

Furthermore, we are adding one more reason to exit with an error code (see "Maintaining a lean baseline"). This might have some negative side-effects to wrapper scripts that assume that error messages are available when that happens. We could introduce a different exit code, to differentiate between exiting due to unresolved errors or ignored errors that do not occur anymore.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

Unfortunately existing approaches do not address the issue at its core and come with their own set of drawbacks. It is worth mentioning that the suggested solution is based on [how baseline works in PHPStan](https://phpstan.org/user-guide/baseline).

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

No, we are only counting errors when generating the baseline. Also only errors are considered when checking against the baseline.

## Related Discussions

* [Change Request: Introduce a system to suppress existing errors](https://github.com/eslint/eslint/issues/16755)
* [PHPStan - The Baseline](https://phpstan.org/user-guide/baseline)

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->