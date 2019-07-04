- Start Date: 2018-12-2
- RFC PR: https://github.com/eslint/rfcs/pull/4
- Authors: David Aghassi

# Async/Parallizing ESLint Processing Of Files

## Summary

ESLint currently is entirely synchronous. The underlying code, `CLIEngine` currently doesn't support any async functions. As such, large codebases pay a toll since it requires users to wait for each file ( time O(n) ) to be processed by ESLint. There's a large gain to be made if ESLint can offer parallelized tasks so that users can more quickly statically analyze their code. This change would also bubble up to the `cli` level for users who don't access the API directly.

## Motivation

On code bases in larger companies (in my case, Intuit) ESLint can take a very long time relative to other build tasks, even with caching enabled. The faster users can iterate and get feedback for their code, the faster they can develop/debug/ship. It also has impacts on CI since the same toll is paid.

## Detailed Design

Based on discussion in the original [PR](https://github.com/eslint/eslint/issues/10606#issue-341171187)there are some main points.

1. The features that are intended to be async/parallel need to be behind a flag on the CLI, and opt in when using the API (maybe through an async command or option when using CLIEngine).
2. There are two options here: `promise` based and `thread` based. Some users suggested looking at [esprint](https://github.com/pinterest/esprint) for inspiration. Some of the concerns were that `promises` would not be faster per say depending on the implementation. As a first iteration, the API should support `async` calls backed by `Promise`s. This will allow the main thread to continue operating while ESLint is doing its thing.
3. The experience should not break existing users.

Based on looking at other libraries, like JSCodeShift, the general flow should look like.

1. Get all the files to be processed (via a [promise](https://github.com/facebook/jscodeshift/blob/master/src/Runner.js#L216)). These can be stored in a FIFO (first in first out) queue for later parsing since order should not matter. NOTE: The loading of the files can remain synchronous for first iteration, but should be asynchronous in a second round change once the initial change is released.
2. [Determine the threshold of processes](https://github.com/facebook/jscodeshift/blob/master/src/Runner.js#L224) we can have running at once. This is done by looking at the CPU count on the given system, and taking `Math.min` of the files relative to the CPU (which means in most cases this will come out to be 4 assuming a quad core processor). If someone passes `runInBand`, it is single threaded by default.
3. [Determine how many files can go to each process](https://github.com/facebook/jscodeshift/blob/master/src/Runner.js#L225). Take some arbitrary `CHUNK_SIZE` (50 in this case) and apply it relative to how efficiently you can split the number of files across the processes provided. If we have 200 files for example and 4 processors, the math works out to ((200/4)/50) = 1. NOTE: `CHUNK_SIZE` will have to be determined through testing for ESLint to see what is most performant.
4. Send the batched files off to a worker that has a predefined set of functions (in the case of ESLint, this would be the bit you specified). Based on the [messages the workers send](https://github.com/facebook/jscodeshift/blob/master/src/Runner.js#L270-L285) act accordingly. In the case of ESLint, this gets to the point we mentioned earlier about keeping track of failures. `jscodeshift` [reports issues to the console](https://github.com/facebook/jscodeshift/blob/master/src/Runner.js#L66). One thing we need to determine is whether or not a failure to lint, or even write/fix, is grounds for ending the process. I'd prefer if ESLint failed gracefully and just said "unable to lint/fix files:" and provided output.
5. Once the workers are done, [provide output to the user](https://github.com/facebook/jscodeshift/blob/master/src/Runner.js#L292).

The flow in turn would look like:

- Determine list of files to lint
- _Determine processors available_
- _Determine batch size_
- _Send each batch to a worker_
- For each _batch sent to a worker_:
  - Check the cache to see if the file has changed
  - Determine the correct configuration for the file
  - Lint the file
  - Optionally apply fixes to the file
  - Optionally write fixes to the disk
  - _Send a message back to the parent process_
- Gather messages
- Output lint results to console

For this change, we will not need to modify the logic for `.eslintcache`, nor `cli.execute` as they are not exposed. We do need to keep the functionality of `CLIEngine.executeOnFiles` unchanged.

Since Node 6 is intended to be deprecated in ESLint 6, it is ok for us to rely on and use `async/await` syntax to support this change.

## Documentation

1. There should be JSDoc code for sure. In code documentation is always the best. In particular, the NodeJS API and the CLI Documentation would need to reflect this change.
2. I do believe this should warrant a blog announcement to say "hey we are moving in this direction, and would like feedback and help with it". Many companies would contribute and help once the initial implementation is there.

## Drawbacks

1. More to maintain for the maintainers
2. Threading and async problems are hard, and can require considerable effort (even with other projects out there doing it).

## Backwards Compatibility Analysis

As long as this functionality is marked as "beta" until ready and hid behind a flag or options, it should not break any existing users.

As noted by [discussions during the RFC](https://github.com/eslint/rfcs/pull/4#issuecomment-469583046), Node 6 or earlier will not need supporting.

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
