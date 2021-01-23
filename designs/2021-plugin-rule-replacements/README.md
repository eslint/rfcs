- Repo: eslint/eslint, eslint/eslintrc
- Start Date: 2021-01-23
- RFC PR: (to be filled in)
- Authors: mykhalov

# Allow plugins to define rule replacements

## Summary

Provide better error message for removed rules by allowing plugins to define rule replacements.

## Motivation

ESLint provides great developer experience around removed rules:

```
  1:1  error  Rule 'generator-star' was removed and replaced by: generator-star-spacing  generator-star
```

Until now this hasn't been the case with plugins:

```
  1:1  error  Definition for rule '@typescript-eslint/camelcase' was not found  @typescript-eslint/camelcase
```

Although `@typescript-eslint/camelcase` [has been removed in favour of another rule](https://github.com/typescript-eslint/typescript-eslint/blob/master/packages/eslint-plugin/docs/rules/camelcase.md), there is no way for the plugin to explain this.

## Detailed Design

When reading plugin definition, look for `"replacements"` property next to `"rules"`:

```
module.exports = {
    rules,
    replacements: {
        rules: {
            foo: ['bar']
        }
    }
}
```

If exists, propagate this property to error reporting. When constructing error message, look into plugin replacements before looking into to ESLint replacements.

## Documentation

This should be documented in [Developer Guide](https://eslint.org/docs/developer-guide/working-with-plugins).

## Drawbacks

None at the moment.

## Backwards Compatibility Analysis

No effect on backwards compatibility.

## Alternatives

None at the moment.

## Open Questions

None at the moment.

## Help Needed

None at the moment.

## Frequently Asked Questions

None at the moment.

## Related Discussions

Adding deprecation message to ESLint itself ([#1549](https://github.com/eslint/eslint/issues/1549)).
