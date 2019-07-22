- Start Date: 2019-07-22
- RFC PR: (leave this empty, to be filled in later)
- Authors: Jordan Eldredge <jordan@jordaneldredge.com>

# Warn When Directives Use the Wrong Type of Comment

## Summary

ESLint supports file level configuration via the use of [directives specified in code comments](https://eslint.org/docs/user-guide/configuring#using-configuration-comments). Some types of directives are only allowed in block level comments (`/* Block Level Comment */`):

* `exported`
* `global(s)`
* `eslint-(en|dis)able`
* `eslint`

Currently if a user adds one of these directives in an inline comment (`// Inline Comment`) it will have no effect and they will need to consult the documentation to figure out what they did wrong.

I propose that if we detect one of these directives in an inline commment that we present the user with a lint error.

## Motivation

As a user of ESLint I occasionally wish to disable a rule, declare a valid global or add some other file level config. In doing so I often forget that some directives are only valid inside block level comments and add them as inline comments by mistake. When the comment fails to have its desired effect I consult the documentation to see what I did wrong. Did I missremember the directive syntax or name? On more than one occasion the problem has been that I used the wrong kind of comment.

It would be nice if I could have been notifed of that mistake the moment I saved my file, rather than having to reread the docs.

I would imagine this type of feedback would be even more helpful for new users who are using directives for the first time.

## Detailed Design

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

## Documentation

I don't beleive that this would requrie any additional documentation.

## Drawbacks

I don't see any drawbacks to adding this feedback.

## Backwards Compatibility Analysis

Users may find that upon upgrading to the version of ESLint that contains this change, that they have new warnings which they might need to fix before upgrading.

## Alternatives

This could also be implemented as a userland ESLint rule. For example, here's a rule I wrote which warns when a user uses the `eslint-disable rule-name` directive in an inline comment. The main downside of having this rule live outside of ESLint is that it would have to exactly match ESLint's directive parsing behavior as well as its notion of which directives are permitted in which types of comments.

```JavaScript
const eslintRuleDirective = /^\s*(eslint-(enable|disable)) .*$/;

function isInvalid(comment) {
  return comment.type === 'Line' && eslintRuleDirective.test(comment.value);
}

module.exports = function rule(context) {
  const sourceCode = context.getSourceCode();
  sourceCode
    .getAllComments()
    .filter(isInvalid)
    .forEach(comment => {
      const directive = comment.value.replace(eslintRuleDirective, '$1');
      context.report(
        comment,
        `ESLint expects \`${directive}\` directives to be used in block level comments.`,
      );
    });
  return {};
};
```

## Open Questions

Should this rule be fixable? I don't currently see any directive problems which offer fixes.

## Help Needed

I expect that I could make the pull request myself without help.

## Related Discussions

* [Warn when ESLint directives which expect to be used in block level comments are used in inline comments #12014](https://github.com/eslint/eslint/issues/12014)