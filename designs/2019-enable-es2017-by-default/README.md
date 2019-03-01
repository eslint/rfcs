- Start Date: 2019-02-28
- RFC PR: https://github.com/eslint/rfcs/pull/16
- Authors: Teddy Katz

# Enable ES2017 parsing and globals by default

## Summary

This RFC proposes to:

* Add an `es2017` environment which enables all ES2015 and ES2017 globals.
* Enable the `es2017` environment by default.
* Change the `es6` environment to no longer enable `parserOptions: { ecmaVersion: 6 }`.
* Change the default value of `parserOptions.ecmaVersion` in `espree` from `5` to `2017`.

## Motivation

### Users expect to be able to use new JS features by default

By default, ESLint enables only the globals and syntax features from the ES5 spec, typically raising a linting error of some kind if more recent features are used. However, a large fraction of ESLint users want to use more recent features in their codebases, for a variety of reasons:

* Some users only need their code to run in environments that support ES2015 (e.g. Node, or modern browsers other than IE11)
* Some users use build tools such as `babel` or `typescript` to compile their ES2015+ code down to ES5.

Due to the increased prevalence of self-updating browsers, new JS features are available at a much faster rate than in the past, reducing the need for users to be aware of which browsers/environments support which features. As a result, new users are often confused by the fact that ESLint raises parsing errors by default for ES2015 code. This adds friction to the process of creating a config for the first time, and might delay or prevent users from using ESLint at all.

The current behavior is "noisy by default" -- if ESLint isn't sure what behavior the user wants and its default gets things wrong, an ES5 default makes the error manifest as a false positive rather than a false negative. This has some significant advantages -- a user is much more likely to notice a false positive than a false negative, allowing them to correct the config error.

However, at this point it appears that most of our users would expect ES6 parsing by default. As a result, the advantage of a false positive over a false negative is outweighed by the fact that making this change would be replace a very large number of false positives with a small number of false negatives. (Using similar reasoning, [eslint/eslint#11105](https://github.com/eslint/eslint/issues/11105) updated `eslint --init` to set `parserOptions.ecmaVersion` to `2018` by default.)

### The `es6` environment composes poorly with other environments because it sets `parserOptions.ecmaVersion`

Currently, the `es6` environment has two effects: it enables the globals introduced in ES2015, and it also sets `parserOptions.ecmaVersion` to `2015`. While this originally made it easier for users to enable everything introduced in ES2015, the use of `parserOptions: { ecmaVersion: 2015 }` in the `es6` env has caused substantial confusion and constrained the design of new envs. Specifically:

* As new parser options like `ecmaVersion: 2016` have emerged, users have needed to use `env: es6` and `parserOptions: { ecmaVersion: 2016 }` separately, negating the benefit of having the `es6` env handle both globals and parser options.
* Users are often confused about the distinction between enabling a set of globals and enabling parser options, since `env: es6` does both and there isn't a way to only enable ES6 globals with a single directive.
* Setting `parserOptions.ecmaVersion` to `2015` in an env has the potential to inadvertently *downgrade* a user's parser options. (For example, if we introduced an `es2017` env that set `parserOptions: { ecmaVersion: 2017 }`, then using `env: { es2017: true, es6: true }` would have the effect of enabling all ES2017 globals while only enabling ES2015 parsing. This is because `ecmaVersion: 2015` in the `es6` env would override `ecmaVersion: 2017` in the `es2017` env.)
    * Note: An explicit `parserOptions` value in a config always has precedence over anything configured in an `env`. This behavior was originally the result of a bug, but it has been preserved because many users are now depending on it to avoid the `ecmaVersion`-downgrade problem described above (as described in [eslint/eslint#8291](https://github.com/eslint/eslint/pull/8291)).

Also see [eslint/eslint#9109](https://github.com/eslint/eslint/issues/9109) for further discussion about how `env: es6` causes confusion.

Many users use `env: es6` without also configuring `parserOptions` themselves. As a result, as long as the default `ecmaVersion` is `5`, changing the behavior of `env: es6` to only configure globals would be an unacceptably large breaking change, since it would have the effect of causing new syntax errors on ES2015 features for those users. However, if we're changing the default value of `ecmaVersion` to support newer syntax by default, we should take this opportunity to fix the problem without any additional breaking changes (aside from the breaking changes that would occur anyway from changing the default `ecmaVersion`).

## Detailed Design

This proposal would:

* Add a built-in `es2017` environment that enables all of the globals defined by the ES2017 spec. (This includes all of the globals in the existing `es6` environement, and also the `Atomics` and `SharedArrayBuffer` globals, which were introduced in ES2017.)
    * Implementation note: The `es2017` environment would not need to explicitly enable ES5 globals such as `Math`, because those globals are already always enabled anyway.
* Enable the `es2017` environment by default. This is roughly equivalent to always passing the object `{ env: { es2017: true } }` as the `baseConfig` option in `CLIEngine`, in that all user-supplied configs would extend the base config, and could override it. (A user could set `{ env: { es2017: false } }` in their own config to override the default.)
* Update the existing `es6` environment to only enable ES2015 globals, and not change `parserOptions`.
* Change the default value of `parserOptions.ecmaVersion` in `espree` from `5` to `2017`.

As a result of these changes, the user-facing effects would be:

* By default, ESLint would parse ES2017 features, and rules like `no-undef` would allow the usage of ES2015 and ES2017 globals.
* Users could revert to the previous default behavior by adding `env: { es2017: false }, parserOptions: { ecmaVersion: 5 }` to their config.
* The `es6` environment would only configure globals, and not parser options. As a result, enabling the `es6` environment would have no effect by default, since all of the ES2015 globals would already be enabled by the `es2017` environment. However, the `es6` environment would be useful when the user explicitly disables the `es2017` environment, for the purposes of enabling only ES2015 globals without also enabling ES2017 globals.

## Documentation

Since this is a breaking change, it would appear in the migration guide. As described in the "Drawbacks" section, we would want to prominently announce this change to minimize the number of users that expect ES5-only behavior and upgrade without changing their configs.

We would also update our documentation to communicate that `es6` and `es2017` environments only affect globals, not parser options. We would want to add examples indicating how to configure ESLint various common use cases, including the previous default behavior of ES5 parsing and globals.

## Drawbacks

The main downside of this proposal is that it would be a silent breaking change for existing users who only want ES5 parsing/globals. This kind of breaking change is always unappealing, since some users who upgrade might not notice the change until they realize that certain code is allowed when it should have been blocked. However, as fewer and fewer users run their code directly on ES5 environments, the default behavior becomes less and less useful, and I think we've reached a point where it's worthwhile to change the default.

It seems like a deprecation warning would not be feasible in this case, since it involves changing the default behavior.  However, we could prominently note the change in the migration guide to reduce the risk of unnoticed behavior changes.

As discussed in the "Motivation" section, allowing more things by default would create some false negatives from ESLint, if the user expects ESLint to to parse a stricter subset of ES than it actually does. However, this drawback is believed to be outweighed by a much larger decrease in false positives.

## Backwards Compatibility Analysis

This would be a breaking change for users whose code directly runs in ES5 environments (and configs designed to enforce ES5 code). ESLint would start allowing ES2015-ES2017 code by default, potentially against the users' wishes. These users could restore the old behavior by adding `env: { es2017: false }, parserOptions: { ecmaVersion: 5 }` to their config.

Similarly, changing the default `ecmaVersion` would also be a breaking change in `espree`. Users could restore the old behavior by providing `ecmaVersion` explicitly.

The presence of new globals would also cause rules like `no-redeclare` to start reporting more errors.

With a minor exception, removing `parserOptions: { ecmaVersion: 2015 }` from the `es6` env would *not* be an additional breaking change, because:

* Users who enable the `es6` env without also configuring `parserOptions` will end up with the new default parsing behavior of `parserOptions: { ecmaVersion: 2017 }`.
* Users who enable the `es6` env and also configure `parserOptions` will still have their own parser options used, because `parserOptions` have precedence over `env`.

(Exception: If a user users a third-party `env` that enables something like `parserOptions: { ecmaVersion: 2017 }`, and they also use the `es6` env, they might have been relying on the fact that the final `ecmaVersion` would be `2015`. There are no known examples of users doing this, although it might exist somewhere. This only applies to third-party environments, not shareable configs, because `parserOptions` in a config has precedence over environments.)

## Alternatives

### Set the default ES version to something other than ES2017

Starting in April 2019, the oldest maintained version of Node will be Node 8, which supports all of the syntactic features of ES2017. Additionally, most commonly-targeted browsers also support all ES2017 syntactic features (aside from IE11, which only supports ES5).

Support for ES2017 shared memory and atomics (including the `Atomics` and `SharedArrayBuffer` globals) is a bit less consistent; it's only supported in Node 9+, and it's currently disabled in non-Chrome browsers as a temporary mitigation for the [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability) vulnerabilities). However, given that there are no built-in alternatives to shared memory/atomics, it seems unlikely that someone would *accidentally* start using shared memory/atomics as a result of the lack of linting errors about the feature.

On the other hand, async functions (also introduced in ES2017) are extremely popular, and it seems valuable to parse them by default.

To address the lack of support for shared memory, we could choose to parse ES2017 by default while only allowing ES2015 globals by default, but it seems like that inconsistency might not be worthwhile. A user could manually disable the `Atomics`/`SharedArrayBuffer` globals either by configuring them to `off`, or by disabling the `es2017` environment and enabling the `es6` environment.

### Keep setting `parserOptions` in the `es6` env, and create an `es2017` env that also sets parser options

This could create confusion if the `es6` and `es2017` environments are enabled at the same time (possibly by different configs). Depending on which environment is enabled first, this could result in enabling ES2017 globals while only enabling ES2015 parsing, which is unlikely to be what the user expected.

If we additionally decided to enable the `es2017` environment by default, the unexpected case would appear in all configs that enable the `es6` environment.

### Keep setting `parserOptions` in the `es6` env, and create an `es2017` env that does not set parser options

This would avoid the environment conflict errors described in the previous section, but the inconsistency of this solution is unappealing and seems like it would cause confusion.

### Enable ES2017 parsing and globals by default, without creating an `es2017` env

This is feasible (since there are only two ES2017 globals, it's not too difficult for a user to manually disable them), but it seems like we will need to solve the `env` inconsistency problem eventually if there are future ES versions that add more globals (assuming we don't eliminate environments entirely).

If we don't remove `parserOptions` from the `es6` environment, this problem would end up happening regardless of whether we enable an environment by default.

### Set the default `ecmaVersion` in `Linter`, rather than changing the default in espree

This alternative isn't feasible because custom parsers have no way to distinguish between parser options provided by the user, and "default" options provided by ESLint. (For example, some parsers respond to the `ecmaVersion` flag but have a different default behavior than espree. If ESLint always passed a default `ecmaVersion` value to parsers, the parsers would never be able to use their own default behavior.)

As a result, ESLint doesn't pass any "default" options to parsers for options that can be provided by users.

Also see: [eslint/eslint#8744](https://github.com/eslint/eslint/issues/8744)

### Always support the latest stage 4 features by default, and use rules to disallow newer syntax/globals as needed

This was proposed in [eslint/eslint#11419 (comment)](https://github.com/eslint/eslint/issues/11419#issuecomment-466317403). It would still be possible to make that change separately from this RFC, but this RFC is intended as a less extreme solution for now.

## Frequently Asked Questions

None yet

## Related Discussions

* [eslint/eslint#11419](https://github.com/eslint/eslint/issues/11419)
* [eslint/eslint#8290](https://github.com/eslint/eslint/issues/8290)
* [eslint/eslint#8291](https://github.com/eslint/eslint/pull/8291)
* [eslint/eslint#9109](https://github.com/eslint/eslint/issues/9109)
