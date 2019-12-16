- Start Date: 2019-12-16
- RFC PR:
- Authors: LJ Talbot ([@LJNeon](https://github.com/LJNeon/))

# Files with .mjs or .cjs extensions should override sourceType

## Summary

Files with a `.mjs` extension are always modules and files with a `.cjs` extension are always scripts. The `sourceType` configuration option currently causes eslint to handle those files incorrectly.

## Motivation

Node.js support for esm is now unflagged in version 13, and the ability to support both cjs and esm modules was also added. As node.js esm support stabilities the amount of packages that use this will grow significantly. This will introduce code that does not utilize only one `sourceType`, and node.js handles this via `.mjs` and `.cjs` file extensions. This is guaranteed behavior and should be supported.

## Detailed Design

This would be implemented by changing `sourceType` to only affect files with a `.js` file extension. Files with a `.mjs` extension are always treated as modules, and files with a `.cjs` extension are always treated as scripts.

## Documentation

I am unsure if this would need a formal announcement, but I would presume not.

## Drawbacks

I can't think of any but I am sure that they exist.

## Backwards Compatibility Analysis

This should have full backwards compatibility, as files with a `.js` extension will be handled the same.

## Alternatives

Eslint currently allows [glob-based configuration overrides](https://eslint.org/docs/user-guide/configuring#configuration-based-on-glob-patterns) to be used in order to handle this behavior. However, as this is guaranteed behavior it would be preferable to support it by default.

## Open Questions

 - Would it be simple to implement this with full backwards compatibility like I think it would be?
 - What potential drawbacks exist here?
 - Should this change only occur when the `node` environment is set?

## Help Needed

I wouldn't be able to implement this RFC, it would need to be implemented by others.

## Related Discussions

https://github.com/eslint/eslint/issues/12675
