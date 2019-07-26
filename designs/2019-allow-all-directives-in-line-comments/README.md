- Start Date: 2019-07-22
- RFC PR: (leave this empty, to be filled in later)
- Authors: Jordan Eldredge <jordan@jordaneldredge.com>

# Allow All Directives in Line Comments

## Summary

ESLint currently only allows the following types in directives to be used inside of block comments (`/* Block Comment */`):

* `exported`
* `global(s)`
* `eslint-(en|dis)able`
* `eslint`

Currently if a user adds one of these directives in a line comment (`// Line Comment`) the directive will be silently ignored. Which can be very confusing.

I propose that we allow all directives to work in either type of comment.

## Motivation

As a user of ESLint I occasionally wish to disable a rule, declare a valid global or add some other file level config. In doing so I often forget that these directives are only valid inside block level comments and add them as inline comments by mistake. When the comment fails to have its desired effect I consult the documentation to see what I did wrong. Did I missremember the directive syntax or name? On more than one occasion the problem has been that I used the wrong kind of comment.

It would be nice if I didn't have to think about the comment type that I'm using.

## Detailed Design

The function `getDirectiveComments` in [`linter.js`](https://github.com/eslint/eslint/blob/1fb362093a65b99456a11029967d9ee0c31fd697/lib/linter/linter.js#L263) currently does a `if (comment.type === "Block")` check before it looks for some types of directives.

I believe we could simply remove that check.

## Documentation

We would need to update the [configuration](https://eslint.org/docs/user-guide/configuring#using-configuration-comments) documentation to remove the places where it states that some types of directives must use block comments. We would replace each of those comments with a note that "Before version _x.x_ this directive only worked in block comments" so that users using an older version of ESLint could figure out why their line comments were ignored.

## Drawbacks

I don't see any drawbacks to this change.

## Backwards Compatibility Analysis


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

An earlier revision of this RFC suggested reporting an error/warning when a directive that is only supported in a bock comment was found in a line comment.

## Open Questions

As far as I am aware, there are no open questions.

## Help Needed

I expect that I could make the pull request myself without help.

## Related Discussions

* [Warn when ESLint directives which expect to be used in block level comments are used in inline comments #12014](https://github.com/eslint/eslint/issues/12014)