- Repo: eslint/eslint
- Start Date: 2021/1/12
- RFC PR: https://github.com/eslint/rfcs/pull/75
- Authors: [A-Katopodis](https://github.com/A-Katopodis)

# (Break on parsing errors)

## Summary

The suggested change is for ESLint tp support an argument that will make the CLI exit with code 2 if any fatal errors are reported. The option will be an opt-in boolean argument called `--fatal-parse-error`.

## Motivation

We met with a couple of cases where we assumed a succesfull run of ESLint in CI enviroments. When ESLint wouldn't be able to read the tsconfig.json, or had a wrongly configured source type the error would be a `fatal` one but the exit code would be `1`. There was no distiction between a run with no fatal errors and with fatal errors.

More specifically the tsconfig.json invalid read is a error from `@typescript-eslint parser`, the parser correctly indicates a fatal parsing failure like it should so it falls under the same category as the incorrectly configured `sourceType` as in both cases we can't actually run any rules in the file.

According to the eslint docs about exit codes:

- 0: Linting was successful and there are no linting errors. If the --max-warnings flag is set to n, the number of linting warnings is at most n.
- 1: Linting was successful and there is at least one linting error, or there are more linting warnings than allowed by the --max-warnings option.
- 2: Linting was unsuccessful due to a configuration problem or an internal error.

Being able to exit with code `2` instead of `1` it will allow for some CI pipelines to better understand the results and react differently if ESLint reports any `fatal` errors. For example, an Azure DevOps build task that is responsible for running ESLint and output a SARIF would fail on exit code `2` but not 1 indicating there is a configuration and not all of the files where scanned properly for rules.  

## Detailed Design

Design Summary:

1. Add a `fatal-parse-error` option.
2. Gather and report the fatal messages from `cli-engine`.
3. Return exit code `2` in the case of any fatal errors on `cli`

A command example:

`eslint **.js --fatal-parse-error`

With this argument if there is at least one fatal error in the results ESLint produces it will exit with code 2.

## Add a `fatal-parse-error` option.
We would require the option in regard to other ones as well.
```
// inside of eslint.js
 ..
 ..
 * @property {number} fatalErrorCount Number of fatal errors for the result.
 ..
 ..
```
```
// inside of lib/option.js
..
..
{
    option: "fatal-parse-error",
    type: "Boolean",
    default: "false",
    description: "Trigger exit code 2 on any fatal errors."
}
..
..
```
## Gather and report the fatal messages.

In order for ESLint to be able to make a choice based on the fact that a `fatal` error has been found or not we must first retrieve this information. Altough we will not be needing the count of the fatal errors using count instead of a boolean and offer more flexibility than a Boolean parameter, keeps the codebase consistent.

The changes for that are on `cli-engine.js`:
```
function calculateStatsPerFile(messages) {
    return messages.reduce((stat, message) => {
        if (message.fatal || message.severity === 2) {
            stat.errorCount++;
            if (message.fatal) {
                stat.fatalErrorCount++;
            }
            if (message.fix) {
                stat.fixableErrorCount++;
            }
        } else {
            stat.warningCount++;
            if (message.fix) {
                stat.fixableWarningCount++;
            }
        }
        return stat;
    }, {
        errorCount: 0,
        fatalErrorCount: 0,
        warningCount: 0,
        fixableErrorCount: 0,
        fixableWarningCount: 0
    });
}
```
```
function calculateStatsPerRun(results) {
    return results.reduce((stat, result) => {
        stat.errorCount += result.errorCount;
        stat.fatalErrorCount += result.fatalErrorCount;
        stat.warningCount += result.warningCount;
        stat.fixableErrorCount += result.fixableErrorCount;
        stat.fixableWarningCount += result.fixableWarningCount;
        return stat;
    }, {
        errorCount: 0,
        fatalErrorCount: 0,
        warningCount: 0,
        fixableErrorCount: 0,
        fixableWarningCount: 0
    });
}
```

## Return exit code `2` in the case of any fatal errors on `cli`

Now we can retreve those fatal errors from `cli.js` and act accordingly:

First change the function that retrieves those errors:
```
function countErrors(results) {
    let errorCount = 0;
    let fatalErrorCount = 0;
    let warningCount = 0;

    for (const result of results) {
        errorCount += result.errorCount;
        fatalErrorCount += result.fatalErrorCount;
        warningCount += result.warningCount;
    }

    return { errorCount, warningCount };
    return { errorCount, fatalErrorCount, warningCount };
}
```

Now with the new information this method gives us we can use it:
```
// on the execute method
if (await printResults(engine, results, options.format, options.outputFile)) {
    const { errorCount, warningCount } = countErrors(results);
    const { errorCount, fatalErrorCount, warningCount } = countErrors(results);
    const tooManyWarnings =
        options.maxWarnings >= 0 && warningCount > options.maxWarnings;
    const fatalErrorExists =
        options.fatalParseError && fatalErrorCount > 0;

    if (!errorCount && tooManyWarnings) {
        log.error(
            "ESLint found too many warnings (maximum: %s).",
            options.maxWarnings
        );
    }

    if(fatalErrorExists){
        return 2;
    }

    return (errorCount || tooManyWarnings) ? 1 : 0;
}
return 2;
```

## Documentation
The section of the documentation requiring the change is the Command Line Interface:

[Command Line Interface](https://eslint.org/docs/user-guide/command-line-interface)


## Drawbacks
Some users who may want to enable this new feature may find themselves needing to reconfigure their process. It can change some CI pipelines if used. But since its optional the impact is minimal and it may allow them to discover issues.


## Backwards Compatibility Analysis
There are no expected backward compatibility issues. The parameter will be disabled by default and only users who will use this new parameter will experience any kind of change.


## Open Questions

### Question:
Do we want to still report all errors or only failed linting errors when `break-on-lint-error` is passed?

### Answer:
After discussions we will be reporting all errors.

----
### Question:
Assume that for some files ESLint successfully lints them and reports a rule but for some others it doesnt.
If ESLint finds a single file that has a parsing error should it report just that file or every rule as 
well?
### Answer:
The choice is that we will be returing all of the found linting errors as discussed in these [comments](https://github.com/eslint/rfcs/pull/76#discussion_r572924056)

## Alternatives
There 2 alternatives which we may want to consider, they were already discussed on the related issue:

## Use a new exit code 3
The proposal remains the same but with exit code `3` instead of `2`

## Max Fatal Errors
Another tweak to the proposal would be for the argument to be an integer similar to `--max-warnings`. Instead of reporting a different exit code than `1` we would 

## Related Discussions
https://github.com/eslint/eslint/issues/13711
