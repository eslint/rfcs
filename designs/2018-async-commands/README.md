- Start Date: 2018-12-2
- RFC PR: (leave this empty, to be filled in later)
- Authors: David Aghassi

# Async/Parallizing CLIEngine

## Summary

`CLIEngine` currently doesn't support any async functions. As such, large codebases pay a toll since it requires users to wait for each file ( time O(n) ) to be processed by ESLint. There's a large gain to be made if ESLint can offer parallelized tasks so that users can more quickly statically analyze their code.

## Motivation

On code bases in larger companies (in my case, Intuit) ESLint can take a very long time relative to other build tasks, even with caching enabled. The faster users can iterate and get feedback for their code, the faster they can develop/debug/ship. It also has impacts on CI since the same toll is paid.

## Detailed Design

Based on discussion in the original [PR](https://github.com/eslint/eslint/issues/10606#issue-341171187)there are some main points.

1. The features that are intended to be async/parallel need to be behind a flag on the CLI, and opt in when using the API (maybe through an async command or optoin when using CLIEngine).
2. There are two options here: `promise` based and `thread` based. Some users suggested looking at [esprint](https://github.com/pinterest/esprint) for inspiration. Some of the concerns were that `promises` would not be faster per say depending on the implementation. On the flipside, it has been noted that `threads`/workers can be expensive on platforms like Windows. Ideally, the implementation would use a balance of both. For example, `jest` has async methods that use threads under the hood.
3. The experience should be opt in so as not to break existing users.
4. There should be some sort of mechanism to handle eslint rules that expect `sync` reading. Rules may need to identify themselves as a "type". This way, if a rule is being loaded that requires `sync`, the code can handle it accordingly. Any rule that doesn't identify as a `sync` or `async` would be defaulted to `sync`.

## Documentation

1. There should be JSDoc code for sure. In code documentation is always the best.
2. I do believe this should warrant a blog announcement to say "hey we are moving in this direction, and would like feedback and help with it". Many companies would contribute and help once the initial implementation is there.

## Drawbacks

1. More to maintain for the maintainers
2. Threading and async problems are hard, and can require considerable effort (even with other projects out there doing it).
3. This may expose unintended edge cases for eslint rules that expect synchronous scanning. There will need to be a shim mechanism for sure.

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
I would like help from the core team with this effort as I do work full time and even this has been a few months of work in my free time outside of work.

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