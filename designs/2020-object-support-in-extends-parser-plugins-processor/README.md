- Repo: eslint/eslint
- Start Date: 2020-05-12
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima (https://github.com/mysticatea)

# Object support in `extends`, `parser`, `plugins`, and `processor`

## Summary

ESLint accepts objects in all of the `extends`, `parser`, `plugins`, and `processor` fields in configurations.

## Motivation

The longest standing issue [#3458 "Support having plugins as dependencies in shareable config"](https://github.com/eslint/eslint/issues/3458).

Also, the object support lets us configure ESLint more flexible.

## Detailed Design

- [Object support in the `extends` field](#-object-support-in-the-extends-field)
- [Object support in the `parser` field](#-object-support-in-the-parser-field)
- [Object support in the `plugins` field](#-object-support-in-the-plugins-field)
- [Object support in the `processor` field](#-object-support-in-the-processor-field)
- [Deprecations](#-deprecations)

### ■ Object support in the `extends` field

**PoC:** https://github.com/eslint/eslint/commit/63f199218d6b637db680ba99d1ed0e376bd1861e

ESLint accepts objects in the `extends` field. For example:

```js
module.exports = {
  extends: [
    require("eslint-config-foo"),
    require("eslint-plugin-bar").configs.recommended,
  ],
}
```

#### Limitation

If an extendee is an object, ESLint throws a fatal error in the cases that the extendee behaves different between `extends: ["eslint-config-foo"]` and `extends: [require("eslint-config-foo")]`. ESLint should show understandable messages for that error.

Disallowed:

> Those are resolved as relative to the location of the extendee file, but if the extendee is an object then ESLint cannot know the place where the object came from.

- Shareable config names and relative paths in the `extends` field
- Package names and relative paths in the `parser` field

Allowed:

> Those are resolved as relative to the location of the _extender_ file in the both cases.

- Absolute paths and objects in the `extends` field
- Absolute paths and objects in the `parser` field
- Plugin names and objects in the `plugins` field
- Any paths in the `ignorePatterns` field
- Any paths in the `overrides[].files` field
- Any paths in the `overrides[].excludedFiles` field

For example:

```js
module.exports = {
  extends: [
    { parser: "babel-eslint" }, // ERROR: Relative source 'babel-eslint' was found in the 'parser' of the object extendee '.eslintrc.json » extends[0]'.
    { parser: "./my-parser.js" }, // ERROR: Relative source './my-parser.js' was found in the 'parser' of the object extendee '.eslintrc.json » extends[1]'.
    { parser: require.resolve("babel-eslint") }, // OK!
    { parser: require("babel-eslint") }, // OK!

    { extends: ["foo"] }, // ERROR: Relative source 'foo' was found in the 'extends' of the object extendee '.eslintrc.json » extends[4]'.
    { extends: ["./base.js"] }, // ERROR: Relative source './base.js' was found in the 'extends' of the object extendee '.eslintrc.json » extends[5]'.
    { extends: [require.resolve("eslint-config-foo")] }, // OK!
    { extends: [require("eslint-config-foo")] }, // OK!

    { plugins: ["foo"] }, // OK!
    { overrides: [{ files: "*.spec.js" /*...*/ }] }, // OK!
  ],
}
```

Similarly, if `plugin:foo/bar` is present, and the plugin `foo` is an object, and the `plugin:foo/bar` includes package names or relative paths in the `extends`/`parser` field, then ESLint throws a fatal error.

```js
module.exports = {
  extends: ["plugin:foo/bar"], // ERROR: Relative source 'babel-eslint' was found in the 'parser' of the object extendee '.eslintrc.json » plugin:foo/bar'.
  plugins: {
    foo: {
      configs: {
        bar: { parser: "babel-eslint" },
      },
    },
  },
}
```

### ■ Object support in the `parser` field

**PoC:** https://github.com/eslint/eslint/commit/cefdd5c1f44afb8ab2d23f13a688f238d924459b

ESLint accepts objects in the `parser` field. For example:

```js
module.exports = {
  parser: require("babel-eslint"),
}
```

Because we provide [`context.parserPath` API](https://eslint.org/docs/developer-guide/working-with-rules#the-context-object) to third-party rules, ESLint makes a virtual module of the parser object in order to allow `require(context.parserPath)`. For example, it makes `/path/to/node_modules/eslint/lib/virtual-parsers/1.js` module in memory for the first parser object. See [`defineVirtualParserModule()`](https://github.com/eslint/eslint/commit/cefdd5c1f44afb8ab2d23f13a688f238d924459b#diff-91937c0f974645b62296ca160b952f7eR392) of PoC for details.

### ■ Object support in the `plugins` field

**PoC:** https://github.com/eslint/eslint/commit/a86e54ecd4bfc1f20bbe4b27e191b4bd6e1d84c8

ESLint accepts objects in the `plugins` field. For example:

```js
module.exports = {
  plugins: {
    one: "one", // ← equivalent to `plugins:["one"]`.
    two: require("eslint-plugin-two"),
    "other-name": require("eslint-plugin-three"),

    "": require("eslint-plugin-foo"), // ← ERROR: requires plugin ID.
  },
  rules: {
    "one/a-rule": "error",
    "two/a-rule": "error",
    "other-name/a-rule": "error",
  },
}
```

Note: `plugins:["foo","bar"]` is equivalent to `plugins:{foo:"foo",bar:"bar"}`.

#### Reconsidering plugin conflict

This RFC removes all plugin conflict errors in favor of overriding. If the base config and the derived config have the same name plugin, the derived config just overrides the base config. This behavior is similar to other fields.

This means that ESLint will go to wrong if the base config and the derived config (or multiple base configs) have the plugin that has the same name but incompatible. It may be hard to understand what happened because maybe the appeared error is "rule not found" or "invalid options". So, for debugging, ESLint with `--debug` flag prints how ESLint adopted/ignored plugins. See also [the `mergePlugins` function](https://github.com/eslint/eslint/blob/a86e54ecd4bfc1f20bbe4b27e191b4bd6e1d84c8/lib/cli-engine/config-array/config-array.js#L196-L218)

### ■ Object support in the `processor` field

**PoC:** https://github.com/eslint/eslint/commit/bd5de522e9c2dbeb607f4ac5c7d3d42969c03ac8

ESLint accepts objects in the `processor` field. For example:

```js
module.exports = {
  processor: require("./my-processor"),
}
```

There is no notable stuff.

### ■ Deprecations

As the "[Object support in the `extends` field](#-object-support-in-the-extends-field)" section described, ESLint cannot behave samely in the both `extends:["eslint-config-foo"]` and `extends:[require("eslint-config-foo")]` cases if the extendee contains relative paths or package names in their `extends`/`parser` field. And we can change those to absolute paths easily.

As the "[Object support in the `parser` field](#-object-support-in-the-parser-field)" section described, the `context.parserPath` API doesn't fit the object support.

Therefore, this RFC deprecates those:

- Package names and relative paths in the `extends`/`parser` field in the shareable configs/plugin configs
- `context.parserPath` API

The removal plan is:

1. Documentation-only deprecation in a minor version
2. Runtime deprecation warning in the next major version of step 1. (maybe v8)
3. Removal in a major version after over a year at least since step 2. (maybe v10 or later)

Probably we should consider the alternative of `context.parserPath` API before the runtime deprecation warning.

## Documentation

We should update the "[Configuring ESLint](https://eslint.org/docs/user-guide/configuring)" page.

I'm wondering if we should split the page into the recommended manner and conventional manners.

## Drawbacks

- It brings more complexity to our configuration system.
- It may increase incomprehensible errors by incompatible plugin overriding.

## Backwards Compatibility Analysis

No breaking changes.

The maintainers of shareable configs have to drop the old ESLint support to use this new feature or just expose another file for this new feature.

## Alternatives

### ■ [RFC09](https://github.com/eslint/rfcs/pull/9)

It brings a new configuration system besides of existing one. It has some pros and cons:

- Removing `env` and cascading is really good.
- We have to maintain two configuration systems.
- Users have to check how shareable configs support the two configuration systems, then need an extra package for interoperation.
- All ESLint users have to migrate their all config files.

I think that we can do a softer way than the complete replacement because:

- While discussions, the `plugins` field revived and the `extends` field or similar thing is going to revive. It's going to be similar to the existing config system except `env` and cascading.
- That RFC has brought the config array concept and we have done huge refactoring with the concept. Currently, the config loading logic is well-structured and maintainable.
  - (I think that removing `env` and cascading is still good, so we can discuss those in separate of this RFC.)

### ■ Loading plugins from the location of config files completely.

Currently, plugins in shareable configs are loaded from the location of the derived config files.

We can modify that behavior to load plugins from the location of shareable configs, then removing plugin conflict errors in favor of overriding as similar to this RFC.

This approach has big pros:

- Simple.
- Shareable configs need no breaking changes to use `dependencies` for plugins.

I guess that's what users expected.

On the other hand, this approach has cons:

- It's a breaking change of ESLint.
- As similar to the package names and relative paths in the `extends`/`parser` field, it blocks the object support in the `extends` field.<br>
  Seriously, as different from the `extends`/`parser` field, we cannot use absolute paths in the `plugins` field. This means that we cannot define compatible shareable configs for the both `extends:["eslint-config-foo"]` and `extends:[require("eslint-config-foo")]` with keeping backward compatibility.

## Related Discussions

- https://github.com/eslint/eslint/issues/3458 - Support having plugins as dependencies in shareable config
- https://github.com/eslint/rfcs/pull/9 - Config File Simplification
