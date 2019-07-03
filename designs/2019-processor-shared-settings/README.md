- Start Date: 2019-07-01
- RFC PR: https://github.com/eslint/rfcs/pull/29
- Authors: @Conduitry

# Access to options in processors

## Summary

This would add the ability to send options to processors, via a new `processorOptions` object in the `.eslintrc` configuration.

## Motivation

There is currently no good way to provide any sort of configuration to processor plugins. There are instances where this is useful (`eslint-plugin-html` and `eslint-plugin-svelte3` are two examples I am familiar with). The concept of "shared settings" exists, but these are only provided to rule plugins.

## Detailed Design

A new top-level key called `processorOptions` would be added. (Typically, this would be configured alongside `processor` inside an override). This options object would then be sent as a third argument to `preprocess` and `postprocess`.

## Documentation

We'd document how to send options to processors that support them in <https://eslint.org/docs/user-guide/configuring#specifying-processor>.

We'd document how to receive them as a third argument in <https://eslint.org/docs/developer-guide/working-with-plugins#processors-in-plugins>.

## Drawbacks

Slight increased maintenance burden is all I can think of. The entire configuration object is readily available in `Linter.prototype._verifyWithProcessor` where we would be calling `preprocess` and `postprocess`.

## Backwards Compatibility Analysis

Existing users and plugins will be unaffacted, as this will simply be a third argument passed to `preprocess` and `postprocess`.

## Alternatives

There are currently no processor-specific options. The only real way to access the linting configuration from within a processor is to monkey-patch `Linter.prototype.verify` so that we can intercept the config values. This is [what's happening in `eslint-plugin-svelte3` (near the bottom of the file)](https://github.com/sveltejs/eslint-plugin-svelte3/blob/master/index.js). `eslint-plugin-html` is also doing something similar. It involves searching through `require.cache` for the appropriate file, and is brittle and ugly.

## Open Questions

~~Do we want to provide the entire configuration object to the pre- and postprocessors, or just the shared settings?~~ Now that we're going to have configuration that's explicitly intended for this processor, it probably makes sense to only send that object.

## Help Needed

I can certainly give implementing this a shot, but I've only dug into the ESLint codebase to the extent necessary to work around that lack of this feature. I'd appreciate some help, particularly with writing tests.

If there are other places in the code besides `Linter.prototype._verifyWithProcessor` where processors are run, I'd also like to hear about them.

## Frequently Asked Questions

## Related Discussions

https://gitter.im/eslint/eslint?at=5d0e804ad4535e477a829360
