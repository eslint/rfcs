- Start Date: 2019-07-01
- RFC PR: https://github.com/eslint/rfcs/pull/29
- Authors: @Conduitry

# Access to shared settings in processors

## Summary

This would expose the shared settings in `.eslintrc` to any processors running on the linted files.

## Motivation

There is currently no good way to provide any sort of configuration to processor plugins. There are instances where this is useful (`eslint-plugin-html` and `eslint-plugin-svelte3` are two examples I am familiar with). The concept of "shared settings" exists, but these are only provided to rule plugins.

## Detailed Design

All calls to a processor's `preprocess` and `postprocess` functions would now include a third argument `settings`, which contains the same shared settings object (`config.settings`) that is sent to rules.

## Documentation

https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins could simply mention the new third argument.

## Drawbacks

Slight increased maintenance burden is all I can think of. The shared settings appear to already be readily available in `Linter.prototype._verifyWithProcessor` and `Linter.prototype._verifyWithConfigArray` where we would be calling `preprocess` and `postprocess`.

## Backwards Compatibility Analysis

Existing users and plugins will be unaffacted, as this will simply be a third argument passed to `preprocess` and `postprocess`.

## Alternatives

The only real way to access the shared settings from within a processor is to monkey-patch `Linter.prototype.verify` so that we can intercept the config values. This is [what's happening in `eslint-plugin-svelte3` (near the bottom of the file)](https://github.com/sveltejs/eslint-plugin-svelte3/blob/master/index.js). `eslint-plugin-html` is also doing something similar. It involves searching through `require.cache` for the appropriate file, and is brittle and ugly.

## Open Questions

Do we want to provide the entire configuration object to the pre- and postprocessors, or just the shared settings?

## Help Needed

I can try to give this a shot, but I've only dug into the ESLint codebase to the extent necessary to work around this limitation. I'd appreciate some help, probably particularly with writing tests.

If there are other places in the code besides `Linter.prototype._verifyWithProcessor` and `Linter.prototype._verifyWithConfigArray` where processors are run, I'd also like to hear about them.

## Frequently Asked Questions

## Related Discussions

https://gitter.im/eslint/eslint?at=5d0e804ad4535e477a829360
