- Repo: eslint/eslint
- Start Date: 2020-06-05
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima <https://github.com/mysticatea>

# Accessing `cwd` from formatters

## Summary

This RFC lets formatters can access `cwd` in order to make relative paths or do something like.

## Motivation

Our Node.js API has the `cwd` option, and ESLint uses it while linting. But formatters cannot access the `cwd` option. It prevents formatters from making relative paths for readability.

## Detailed Design

Pass `cwd` to the `cwd` property of the second argument, then formatters can use it.

```js
module.exports = (results, context) => {
  console.log(context.cwd) // → the absolute path to the current working directory.
  console.log(context.rulesMeta) // → the metadata of the used rules (it exists currently).
}
```

### Implementation

The adapter object that the `ESLint.prototype.loadFormatter` method makes gives the `cwd`.

> ```diff
> @@ -575,7 +575,7 @@ class ESLint {
>             throw new Error("'name' must be a string");
>         }
>
> -       const { cliEngine } = privateMembersMap.get(this);
> +       const { cliEngine, options } = privateMembersMap.get(this);
>         const formatter = cliEngine.getFormatter(name);
>
>         if (typeof formatter !== "function") {
> @@ -595,6 +595,9 @@ class ESLint {
>                 results.sort(compareResultsByFilePath);
>
>                 return formatter(results, {
> +                   get cwd() {
> +                       return options.cwd;
> +                   },
>                     get rulesMeta() {
>                         if (!rulesMeta) {
>                             rulesMeta = createRulesMeta(cliEngine.getRules());
> ```
>
> https://github.com/eslint/eslint/commit/43a7e207ca8c0b816d4fba2b4b290f761c33adb4

## Documentation

We should update the "[Working with Custom Formatters](https://eslint.org/docs/developer-guide/working-with-custom-formatters)" page to mention the `cwd`.

## Drawbacks

Nothing in particular.

## Backwards Compatibility Analysis

This addition is backward compatible.

If custom formatters want to use the `cwd` property and continue to support old versions of ESLint, they must check if the `cwd` property exists or not.

## Alternatives

## Related Discussions

- https://github.com/eslint/eslint/issues/13376 - Output relative paths rather than absolute paths
