- Start Date: 2019-07-22
- RFC PR: (leave this empty, to be filled in later)
- Authors: Jordan Eldredge <jordan@jordaneldredge.com>

# Allow All Directives in Line Comments

## Summary

ESLint currently only allows the following types of directives to be used inside of block comments (`/* Block Comment */`):

* `exported`
* `global(s)`
* `eslint-(en|dis)able`
* `eslint`

Currently if a user adds one of these directives in a line comment (`// Line Comment`) the directive will be silently ignored. Which can be very confusing.

I propose that we allow all directives to work in either type of comment.

## Motivation

As a user of ESLint I occasionally wish to disable a rule, declare a valid global or add some other file level config. In doing so I often forget that these directives are only valid inside block level comments and add them as inline comments by mistake. When the comment fails to have its desired effect I consult the documentation to see what I did wrong. Did I remember the directive syntax or name incorrectly? On more than one occasion the problem has been that I used the wrong kind of comment.

It would be nice if I didn't have to think about the comment type that I'm using.

## Detailed Design

The function `getDirectiveComments` in [`linter.js`](https://github.com/eslint/eslint/blob/1fb362093a65b99456a11029967d9ee0c31fd697/lib/linter/linter.js#L263) currently does an `if (comment.type === "Block")` check before it looks for some types of directives.

I believe we could simply remove that check.

For `/* eslint-env */` directives which, by necessity, are parsed out of the file before an AST is avaliable, the regular expression in [linter.js](https://github.com/eslint/eslint/blob/fb08b7c9d28bc68864eb940e26df274059228b6a/lib/linter/linter.js#L406) would need to be updated to allow it to match line comments.

## Documentation

We would need to update the [configuration](https://eslint.org/docs/user-guide/configuring#using-configuration-comments) documentation to remove the places where it states that some types of directives must use block comments. We would replace each of those comments with a note that "Before version _x.x_ this directive only worked in block comments" so that users using an older version of ESLint could figure out why their line comments were ignored.

## Drawbacks

I don't see any drawbacks to this change.

## Backwards Compatibility Analysis

While this change is backwards compatible in that it does not change any existing API contract, there is a possiblity that user's code bases will contain [accidental directives](#accidental-directives) (see below) which could potentially mean that upgrading to a version that contained this change would result in lint errors when it did not before.

Therefore, we will consider this change a _breaking change_.

### Latent Directives

Users may have added directives to their code using line comments and then never noticed that they had no effect. With this change, they will now begin functioning again. In some cases this may be welcome, since the user's original intention would finally be respected. In other cases the code may have changed since the user added the latent directive. In that case this change may cause an unwanted directive to be enabled.

Anecdotally, I did an audit of a large swath of our code base and discovered only two directives which were being ignored due to their comment type. In both cases, enabling would be the correct behavior.

### Accidental Directives

Users may also have written inline comments which were not intended as a directive, but parse as directives. I found one example in ESLint's own codebase:

> `lineNumTokenBefore = 0; // global return at beginning of script`

-- [newline-before-return.js:173](https://github.com/eslint/eslint/blob/02d7542cfd0c2e95c2222b1e9e38228f4c19df19/lib/rules/newline-before-return.js#L137)

This would result in the following, somewhat confusing, error after upgrading to a version that contained this change:

```
/Users/jeldredge/projects/eslint/lib/rules/newline-before-return.js
  137:51  error  'return' is defined but never used     no-unused-vars
  137:58  error  'at' is defined but never used         no-unused-vars
  137:61  error  'beginning' is defined but never used  no-unused-vars
  137:71  error  'of' is defined but never used         no-unused-vars
  137:74  error  'script' is defined but never used     no-unused-vars

✖ 5 problems (5 errors, 0 warnings)
```

## Alternatives

An earlier revision of this RFC suggested reporting an error/warning when a directive that is only supported in a block comment was found in a line comment. See the following section for a discussion of why this may be a worth doing in addition to the changes detailed above.

## Optional Short Term Improvements

Since code bases may contain line comments which were not intended as directives, but would be parsed as such after this change, (see [Accidental Directives](#accidental-directives) above) this change will be a breaking change and thus will need to wait for a major release. If the next major release is far off we could employ a short term fix of making line comment directives warnings which could be shipped in a minor release.

This would have two benefits:

1. In the time between when the minor release ships and when the next major release ships, user would get explicit feedback when they accidentally write a line comment directive when ESLint is expecting a block comment directive rather than having it silently fail.
2. Accidental directives would be surfaced as errors in the minor version which means that those who adopt the minor release before the next major release, and resolved all new warnings would have a smooth upgrade experience.

### Implementation of Warnings

If we decide to do the additional work of adding warnings in a minor version, here is how it could be implemented:

The function `getDirectiveComments` in [`linter.js`](https://github.com/eslint/eslint/blob/1fb362093a65b99456a11029967d9ee0c31fd697/lib/linter/linter.js#L263) already has some examples of reporting problems with directive comments via the `createLintingProblem` function.

Currently `getDirectiveComments` works by first handling `eslint-disable-(next-)line` directives, which are comment type agnostic, and then only looking for the other types of directives if the comment type is `Block`.

Instead, after handling the `eslint-disable-(next-)line` directives, we could check to see if we are in an inline comment. If we are, we could add a problem with something like:

```JavaScript
problems.push(createLintingProblem({
  ruleId: null,
  message: `The ${match[1]} directive is only allowed in block comments.`,
  loc: comment.loc
}));
```

## Open Questions

As far as I am aware, there are no open questions.

## Help Needed

I expect that I could make the pull request myself without help.

## Related Discussions

* [Warn when ESLint directives which expect to be used in block level comments are used in inline comments #12014](https://github.com/eslint/eslint/issues/12014)