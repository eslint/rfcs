- Repo: eslint/eslint
- Start Date: 2022-05-06
- RFC PR: 
- Authors: SherrybabyOne

# add fix support of eslint.lintFiles, eslint.lintText

## Summary

[eslint.lintFiles](https://eslint.org/docs/developer-guide/nodejs-api#-eslintlintfilespatterns) and [eslint.lintText](https://eslint.org/docs/developer-guide/nodejs-api#-eslintlinttextcode-options) add parameter fix to control rule checking.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

I want  `eslint.lintText()` and `eslint.lintFiles()` provide the **fix** parameter under options to control whether to fix the rules or not.

This way you can use the same eslint instance for both fix and no-fix rule checking instead of having to create two ESLint instances:

```javascript
const eslint = new ESLint();

// no fix
const resultsForDiagnose = await eslint.lintText(text, { filePath, fix: false });
// fix
const resultsForFormat = await eslint.lintText(text, { filePath, fix: true });

// no fix
const lintFilesForDiagnose = await eslint.lintFiles(files, { fix: false });
// fix
const lintFilesForFormat = await eslint.lintFiles(files, { fix: true });
```

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

### eslint.lintText(code, options)

The options of `lintText(code, options)` support the fix parameter and pass it to `cliEngine.executeOnText()`:

```javascript
async lintText(code, options = {}) {
    // ...

    const {
        filePath,
        warnIgnored = false,
        fix,
        ...unknownOptions
    } = options || {};

    // ...

    const { cliEngine } = privateMembersMap.get(this);

    return processCLIEngineLintReport(
        cliEngine,
        cliEngine.executeOnText(code, filePath, warnIgnored, fix)
    );
}
```

`cliEngine.executeOnText` merges the fix passed in `lintText()` and the `options.fix` in `ESLint.constructor()` (I omit irrelevant code):

```javascript
executeOnText(text, filename, warnIgnored, fixFromLintText) {
    const {
        options: {
            fix: fixFromESLint,
        }
    } = internalSlotsMap.get(this);

        // Do lint.
        results.push(verifyText({
            // text,
            // filePath: resolvedFilename,
            // config,
            // cwd,
            fix: fixFromLintText !== undefined ? fixFromLintText : fixFromESLint,
            // allowInlineConfig,
            // reportUnusedDisableDirectives,
            // fileEnumerator,
            // linter
        }));
}
```

### eslint.lintFiles(patterns)

`eslint.lintFiles(patterns)` adds options parameter and provides fix parameter:

```javascript
async lintFiles(patterns, options = {}) {
    // ...

    const {
        fix,
        ...unknownOptions
    } = options || {};

    const unknownOptionKeys = Object.keys(unknownOptions);

    if (unknownOptionKeys.length > 0) {
        throw new Error(`'options' must not include the unknown option(s): ${unknownOptionKeys.join(", ")}`);
    }

    const { cliEngine } = privateMembersMap.get(this);

    return processCLIEngineLintReport(
        cliEngine,
        cliEngine.executeOnFiles(patterns, fix)
    );
}
```

`cliEngine.executeOnFiles` has basically the same changes as `cliEngine.executeOnText` above:

```javascript
executeOnFiles(text, fixFromLintFiles) {
    const {
        options: {
            fix: fixFromESLint,
        }
    } = internalSlotsMap.get(this);

        // Do lint.
        results.push(verifyText({
            // text,
            // filePath: resolvedFilename,
            // config,
            // cwd,
            fix: fixFromLintFiles !== undefined ? fixFromLintFiles : fixFromESLint,
            // allowInlineConfig,
            // reportUnusedDisableDirectives,
            // fileEnumerator,
            // linter
        }));
}
```

## Documentation

No.

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

I need both fix and no-fix in one process, I currently have to initialize two eslint instances:

```javascript
// no-fix rules
this.eslintForDiagnose = new ESLint({.fix: false });
this.eslintForDiagnose.lintText();

// fix rules
this.eslintForFormat = new ESLint({ fix: true });
this.eslintForFormat.lintText();
```

This is because fix/no-fix depends on the **options.fix** parameter in `ESLint.constructor()` and I can't control whether to fix the rules or not when calling `lintText()` or `lintFiles`, so I had to create two ESLint instances to handle fix and no-fix respectively , this is twice the performance penalty for memory.

Also, I believe there is a use-case where the `fix` can not be provided when the ESLint instance is initialized, but only when `lintText(`) or `lintFiles()` are actually called, similar to the above.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

No.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

No.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

Currently no.

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I am willing to submit a pull request for this change if the rfc passed.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

No.

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

[eslint/eslint#15797](https://github.com/eslint/eslint/issues/15797) Change Request: lintText、lintFiles can pass the parameter fix
