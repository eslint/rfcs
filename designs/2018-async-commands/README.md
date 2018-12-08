- Start Date: 2018-12-2
- RFC PR: (leave this empty, to be filled in later)
- Authors: David Aghassi

# Async/Parallizing ESLint Processing Of Files

## Summary

ESLint currently is entirely synchronous. The underlying code, `CLIEngine` currently doesn't support any async functions. As such, large codebases pay a toll since it requires users to wait for each file ( time O(n) ) to be processed by ESLint. There's a large gain to be made if ESLint can offer parallelized tasks so that users can more quickly statically analyze their code. This change would also bubble up to the `cli` level for users who don't access the API directly.

## Motivation

On code bases in larger companies (in my case, Intuit) ESLint can take a very long time relative to other build tasks, even with caching enabled. The faster users can iterate and get feedback for their code, the faster they can develop/debug/ship. It also has impacts on CI since the same toll is paid.

## Detailed Design

Based on discussion in the original [PR](https://github.com/eslint/eslint/issues/10606#issue-341171187)there are some main points.

1. The features that are intended to be async/parallel need to be behind a flag on the CLI, and opt in when using the API (maybe through an async command or option when using CLIEngine).
2. There are two options here: `promise` based and `thread` based. Some users suggested looking at [esprint](https://github.com/pinterest/esprint) for inspiration. Some of the concerns were that `promises` would not be faster per say depending on the implementation. As a first iteration, the API should suppor `async` calls backed by `Promise`s. This will allow the main thread to continue operating while ESLint is doing its thing.
3. The experience should not break existing users.

### CLIEngine

I believe the initial implementation needs to stem from the core of ESLint. Many of the core methods rely on `sync` based Node APIs. We can swap these out for `async` based ones that are wrapped in `promisify` to get the promise based async calls we want. However, this will have a bubble up a effect to anything relying on these core APIs since they would return  a `Promise`. None of this should touch the rule API, and should remain relegated to the processing of files under the hood.

#### constructor

https://github.com/eslint/eslint/blob/master/lib/cli-engine.js#L428

Currently the constructor checks if an ignored file exists synchronously. We can swap this logic for:

```javascript
import fs from 'fs';
import util from 'util';

const promiseStat = util.promisify(fs.stat);

...

        if (options.ignore && options.ignorePath) {
            promiseStat(options.ignorePath).then((stats) => {
                if (!stats.isFile()) {
                    throw new Error(`${options.ignorePath} is not a file`);
                }
            })
            .catch((e) => {
                e.message = `Error: Could not load file ${options.ignorePath}\nError: ${e.message}`;
                throw e;
            });
        }
```

#### processFile

The processFile call is used to process a known file by ESLint. The core function this file wraps is [fs.readFileSync](https://github.com/eslint/eslint/blob/master/lib/cli-engine.js#L238). 

Node API allows us to swap 

```javascript
const text = fs.readFileSync(path.resolve(filename), "utf8");
```

for

```javascript
import fs from 'fs';
import util from 'util';

const promiseReadFile = util.promisify(fs.readFile);

...

return promiseReadfile(path.resolve(filename), "utf8").then((text) => {
    ...
});
```

Since we _know_ the file exists, we don't have to have an exception check here (though it is good practice to do so). If we do have an exception, we should keep a tally of those files that caused issues and create a log file like `yarn` does. Since the processes are async, we can keep the exceptions in a map of `Errors` and time stamp each one. This way, we can get the output accordingly. It may be worth noting the file that was not processed and logging it out to a file so as not to stop the entire linting process. Output can then be shown to say "x/total scanned, y unscanned. See log for details".

`executeOnFiles` would have to handle that (assuming nothing else changes). `executeOnFiles` [calls processFile](https://github.com/eslint/eslint/blob/master/lib/cli-engine.js#L612).

#### getCachedFile

This method calls [fs.lstatSync](https://github.com/eslint/eslint/blob/master/lib/cli-engine.js#L362) under the hood to find the cache file. This can be replaced with the normal `lstat` call and `promisifed` like above.

```javascript
let fileStats;

try {
    fileStats = fs.lstatSync(resolvedCacheFile);
} catch (ex) {
    fileStats = null;
}
```

becomes

```javascript
import fs from 'fs';
import util from 'util';

const promisedLstat = util.promisify(fs.lstat);

...

let fileStats =  await promisedLstat(resolvedCacheFile).then((stat) =>  stat). catch ((ex) =>  null);
```

#### executeOnFiles

https://github.com/eslint/eslint/blob/master/lib/cli-engine.js#L577

Calls `processFile`: https://github.com/eslint/eslint/blob/master/lib/cli-engine.js#L612

#### outputFixes

https://github.com/eslint/eslint/blob/master/lib/cli-engine.js#L541

This method calls `fs.writeFileSync`. We can swap that with `promisify` + `fs.writeFile`. That will give us a promise while the file is written. If we fail to write the file, we can keep a running log and exit while out putting a failure log with those files that caused trouble (need more help to understand what mechanisms exist for this currently, if any).

```javascript
import fs from 'fs';
import util from 'util';

const promiseWriteFile = util.promisify(fs.writeFile);

...

    static async outputFixes(report) {
        report.results.filter(result => Object.prototype.hasOwnProperty.call(result, "output")).forEach(result => {
           await  promiseWriteFile(result.filePath, result.output).catch((e) => {
               // Take care of logging failure
           });
        });
    }
```

## Documentation

1. There should be JSDoc code for sure. In code documentation is always the best. In particular, the NodeJS API and the CLI Documentation would need to reflect this change.
2. I do believe this should warrant a blog announcement to say "hey we are moving in this direction, and would like feedback and help with it". Many companies would contribute and help once the initial implementation is there.

## Drawbacks

1. More to maintain for the maintainers
2. Threading and async problems are hard, and can require considerable effort (even with other projects out there doing it).

## Backwards Compatibility Analysis

As long as this functionality is marked as "beta" until ready and hid behind a flag or options, it should not break any existing users.

## Alternatives

https://github.com/pinterest/esprint <--- The main option, but Pinterest seems to have lost interest in it

I tried wrapping ESLint in promises, but that was very messy and didn't work out too well.
I tried making a PR that would change the functions I cared about, but the general discussion (in the PR above) showed there would need to be a more thoughtful attempt.

## Open Questions

1. What is the core architecture of ESLint that requires it to be synchronous?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->
I would like help from the core team with this effort when it comes to implementing (assuming this RFC is accepted) as I do work full time and even this has been a few months of work in my free time outside of work.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

PR - https://github.com/eslint/eslint/issues/10606
Google Group RFC Discussion - https://groups.google.com/forum/m/#!topic/eslint/di7l6__w2jk
Bounty Discussion - https://www.bountysource.com/issues/26284182-lint-multiple-files-in-parallel