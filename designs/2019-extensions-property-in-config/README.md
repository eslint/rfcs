- Start Date: 2019-11-12
- RFC PR: https://github.com/eslint/rfcs/pull/48
- Authors: Pig Fang ([@g-plane](https://github.com/g-plane)), Sindre Sorhus ([@sindresorhus](https://github.com/sindresorhus))

# Specifying Extensions in Configuartion File

## Summary

This RFC proposes adding one property called `extensions` at top level of ESLint configuration files.

## Motivation

Currently if we want to specifying linting targets with specified extensions, the only way is to pass `--ext` via CLI, which isn't shareable. If specifying extensions in configuration file (`.eslintrc.*`) is supported, it will be convenient for shareable configurations files.

## Detailed Design

In this RFC, one property will be added to top level of configuration files:

```json
{
    "extensions": ["ts", "vue"]
}
```

With an example above, type of `extensions` property is an array of string. Each string can be a file extension. Extensions strings can be prefixed with a dot (`.`) or not. In [`FileEnumerator`](https://github.com/eslint/eslint/blob/45aa6a3ba3486f1b116c5daab6432d144e5ea574/lib/cli-engine/file-enumerator.js), strings prefixed with dot will be stripped, so this can improve UX.

Currently, `ConfigArrayFactory` will be passed to the constructor of `FileEnumerator`, which allows to read configuration files. However, when linting files, different files may use different configuration files. So, we can't generate `extensionRegExp` in the constructor of `FileEnumerator`; otherwise, we can't know which configuration file will be used and it will crash if configuration file is missing.

Due to this change, `extensionRegExp` can't be a getter. We need to create a new method call `getExtensionRegExp` which accepts two optional parameters. The first parameter is `configArray` whose type is [`ConfigArray`](https://github.com/eslint/eslint/blob/45aa6a3ba3486f1b116c5daab6432d144e5ea574/lib/cli-engine/config-array/config-array.js), and the second parameter is `filePath` whose type is `string`. When calling this method, if these parameters aren't given, base extensions passed from constructor (They may be passed from CLI option.) will be returned. If the first parameter is given, the second parameter must be given. Using these two parameters, we can read configuration file and get to know extensions specified in configuration files.

## Documentation

- [Configuring ESLint](https://eslint.org/docs/user-guide/configuring) page should state the new way to specify extensions in configuration files with an example.

## Drawbacks

- It's a breaking change. See below.

## Backwards Compatibility Analysis

- Adding the new `extensions` property at top level of configuration files will change the schema of configuration, which is a breaking change. Users using old version ESLint can't use this feature because once they add this property, configuration schema check would fail.

## Alternatives

Currently no.

## Open Questions

- The default extension is `.js`. When users specify custom extensions in configuration files, should it be overrided or included? The current behavior of `--ext` CLI option is overriding it. For example, adding `--ext=.ts` to CLI *won't* lint `.js` files. For this RFC, if `.js` is overrided, it can reduce confusion but being less convenient.

- What's the behavior when specifying extensions in configuration files and in CLI at the same time? (Additional notes: [RFC: Configuring Additional Lint Targets with `.eslintrc`](https://github.com/eslint/rfcs/pull/20) suggests that when extensions specified in CLI, configuration files will be ignored.)

## Help Needed

- According to the second question of [Open Question](#open-questions), if we decided to ignore extensions specified in configuration files when specified in CLI, there's a question: How do we detect whether user specifies `--ext=.js` (to disable this feature) or not? As we know, on one hand, if user doesn't specify any extensions in CLI, default extensions will be `.js`; on the other hand, if user specifies `--ext=.js`, extensions are still `.js`. Thus, we need a way to check it.

## Frequently Asked Questions

## Related Discussions

- [RFC: Configuring Additional Lint Targets with `.eslintrc`](https://github.com/eslint/rfcs/pull/20)
- [RFC: Configuring core options in Config Files](https://github.com/eslint/rfcs/pull/22)
- [Issue: Support specifying extensions in the config](https://github.com/eslint/eslint/issues/10828)
