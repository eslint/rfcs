- Repo: (the repo it will be implemented in, e.g. eslint/espree)
- Start Date: 2020-10-10
- RFC PR: (leave this empty, to be filled in later)
- Authors: fa93hws

# Deprecate caching

## Summary
Current [eslint caching](https://eslint.org/docs/user-guide/command-line-interface#options) assumes all rules are independent, which is not correct.

## Motivation
Deprecate `caching` in [options](https://eslint.org/docs/user-guide/command-line-interface#options).

The reason for that is the assumption that all rules are independent is not valid.
For example, [no-extraneous-dependencies](https://github.com/benmosher/eslint-plugin-import/blob/master/docs/rules/no-extraneous-dependencies.md) has a dependency on `package.json`.
In other words, when `package.json` has been touched, all cache regarding this rule should not be used anymore, which is not the case at the moment.

In order to implement the cache properly, what need to be done is either:
1. Collect dependencies from the rule
2. Reduce the flexibility of the rule implementation

From my point of view, one of the biggest selling point of eslint is its flexibility, e.g. custom rules, plugins etc, which brings the community to a higher level.
In order to keep the extendibility of eslint, it's nearly impossible to collect the dependencies properly from a rule because we never know what kind of genius idea would come into a developer's mind.

As a consequence, I think the following is ultra important to this community:
1. Keep the extendibility of eslint
2. Documented features should be working as expected

Since caching isn't working as expected when rules have dependencies on others are introduced and it's nearly impossible to collect the deps without reducing flexibility significantly, I think it may better to be deprecated.


## Detailed Deasign
State in the docs that `caching` is deprecated.

## Documentation
Yes, I think a formal announcement is recommended.

## Drawbacks
Those who uses rules that have no dependencies only will lose the caching functionality.

However, from my point of view, as a tool to improve the maintainability, number of eslint rules are usually positive related to the size of the code base.
A larger code base is more likely to have customer rules and rules that have extra dependencies.
And the motivation of the caching is to reduce the linting time for those huge repo.
As a consequence, I won't evaluate this drawback as significant.


## Backwards Compatibility Analysis
Docs are changed only and code is still there, so there should be no backward compatibility issue.

## Alternatives
1. Reduce the flexibility of the eslint, so that deps can be collected easily or rules can not have any dependencies.

I don't think we should do this, the flexibility is one of the most charming character of the tool.

2. Implement a magic functionality that can collect the deps automatically.

It's nearly impossible. Wrapping the `fs` library may be one of the possibility but I think that's an way overkill for a linting library..

3. Let user to provide dependency tree

It's super hard to determine a schema that fits all rules. It's the most feasible alternatives I can think of. But considering the difficulty, I think it's better to deprecate it first.

## Frequently Asked Questions
Q: I really need the cache, what should I do?

A: There is [nodejs api](https://eslint.org/docs/developer-guide/nodejs-api). It would be better to write your own caching implementation, because you have much more idea on the dependencies.
For example, you can have a branch that pass the tests.
Linting then can be performed on affected files compare to the commit in that branch.
The algorithm to find affected files should be implemented by you because you are the only one who has knowledge on that.


## Related Discussions
https://github.com/eslint/eslint/issues/12828 Dependency: version of the plugins

https://github.com/eslint/eslint/issues/13505 Dependency: cli options

https://github.com/eslint/eslint/issues/12578 Dependency: eslintrc

https://github.com/benmosher/eslint-plugin-import/blob/master/docs/rules/no-extraneous-dependencies.md Dependency: package.json

https://github.com/benmosher/eslint-plugin-import/blob/HEAD/docs/rules/no-cycle.md Dependency: All imports (including transitive imports) of a file