- Start Date: 2019-10-05
- RFC PR: [#43](https://github.com/eslint/rfcs/pull/43)
- Authors: Evan Plaice ([@evanplaice](https://github.com/evanplaice))

# ES Module Compatibility

## Summary

**This Issue**

Node now supports ES modules. ESLint configurations that use the CommonJS format (i.e., `.eslintrc`, `.eslintrc.js`) are not compatible with ES module based packages.

**The Impact**

This applies to ES module based packages (i.e., packages using `"type": "module"`) using a CommonJS configuration (i.e., `.eslintrc`, `.eslintrc.js`).

**The Fix**

Node provides an 'escape hatch' via the `.cjs` extension. The extension explicitly signals to Node that the file should *always* evaluate as CommonJS. Support for the `.cjs` needs to be added to ESLint, and ESLint users building ES module based packages need to be notified.

## Motivation

ES modules are here. ESLint is ubiquitous in the JS ecosystem. With some minor adjustments, ESLint can be made to be compatible with the new format.

## Detailed Design

To understand the design outlined here requires some background info into the mechanics by which Node modules are loaded.

### Jargon

- Package - A NPM package (i.e., contains `package.json`)
- Module - A JS file
- CJS - A CommonJS module
- ESM - A standard ECMAScript module
- Consumer - A package that uses/depends on this package
- Dependent - Package(s) that this package needs to work
- Boundary - The demarcation between package, consumer, dependent

### A Crash Course on Package Boundaries

Historically, NPM packages have come in a variety of different package formats (e.g., IIFE, AMD, UMD, CJS). Prior to an actual spec, NPM settled on CJS as it's de-facto module format. CJS isn't going anywhere, the addition of ESM is additive.

By default, all NPM packages are CJS-based. That means all `.js` files will be read as CJS modules. To include an ES module in a CJS package, it must have the `.mjs` extension.

If, `package.json` contains `"type": "module"` then the package is ESM-based. Meaning, all `.js` files contained within will be treated as ESM. To include a CJS module in a ESM-based package, it must have the `.cjs` extension.

This configuration option does *not* affect consumers or dependents. Whatever the configuration, it applies to all modules within the package boundary. Assuming packages only ever directly import within their package boundary, there should be no issues with interoperability between CJS/ESM.

### The Scenario

A user is creating a new package. They prefer ESM for Browser <-> Node compatibility so they configure their package to be ESM-based.

The user adds `eslint` as a devDependency and a typical NPM script to lint the package's contents.

The user defines `.eslintrc.js` outlining the rule set they prefer to use within their package.

### The Issue

When the user package is ESM-based, all `.js` files within are read as ESM.

However, ESLint is CJS-based so, it loads all files within its package boundary as CJS.

The configuration file is defined as a CJS module (i.e., `module.exports`), but has a `.js` extension syntax so requiring it throws an error. ESLint, by design reaches across the package boundary to load the user-defined configuration but the user has inadvertently signaled to Node to load it with the wrong module loader.

### The Fix

Add support for the `.cjs` file extension

*Note: In Node, `.cjs` will always load as a CommonJS module, irrespective of package configuration*

### Usage

If a user:

- is building a ESM-based package
- is using a JS-based configuration

They should give the configuration a `.cjs` extension.

### Priority

With the addition of `.cjs`, the new priority for configuration files will be

1. .eslintrc.cjs
2. .eslintrc.js
3. .eslintrc.yaml
4. .eslintrc.yml
5. .eslintrc.json
6. .eslintrc
7. package.json

`.cjs` is placed higher than `.js` because -- unlike the latter -- `.cjs` does not allow for ambiguity. A `.cjs` file is always interpreted as CommonJS regardless of context.

## Documentation

A quick mention in the [FAQ](https://github.com/eslint/eslint#frequently-asked-questions) should be suitable to document usage.

## Drawbacks

### Technical

None. The change has no negative functionality or performance impacts.

### People

Some developers within the Node ecosystem are strongly opposed to supporting `"type": "module"` at all.

## Backwards Compatibility Analysis

*tl;dr: Backward compatibility will be unaffected*

### Story 1 - ESLint Compatibility

This change is additive. It associates `.cjs` files with JS configurations.

The existing semantics of loading CommonJS configurations for CommonJS-based packages do not change.

### Story 2 - Node Compatibility

In Node, ES module and `.cjs` support roll out together.

This change *only* impacts users who define their package as an ES module.

Existing packages that use the default (i.e., CommonJS module resolution) will be unaffected.

## Alternatives

### Alternative 1 - Suppress the error

**[PR#12333](https://github.com/eslint/eslint/pull/12333)**

The fencing exists to prevent users from creating dual-mode (i.e., both ESM/CJS) packages as interoperability between the two format can cause issues. In the case of ESLint, loading a simple config file from a user-defined package should not cause any issues or side-effects.

Eating the exception is viable solution.

### Alternative 2 - Support dynamic `import()`

Instead of using 'import-fresh' to require the config, import it in a cross-format-compatible way using dynamic `import()`.

There has been and will continue to be a lot of talk about using this approach as the CJS/ESM interoperability story is fleshed out.

For now it presents too many unknowns.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

## Frequently Asked Questions

> Why doesn't compatibility w/ ESM-based packages 'just work'?

ESLint reaches across the package boundary to retrieve the user-defined configuration. If the consumer package is ESM-based, the `.js` config file will be seen as an ES module. When the ES module loader encounters `module.exports` (i.e., the CommonJS module specifier) it will throw an error.

> What does this interoperability issue affect?

Only ESM-based packages (i.e., packages with `"type": "module"` defined in package.json).

> What about 3rd-party linting tools that provide their own built-in rule set for ESLint (ex [StandardJS](https://standardjs.com/))?

They aren't impacted by this change. 3rd-party modules define both their code and the ESLint rule set used within the same package boundary.

> What about support for ES module configurations

The scope of this RFC is limited to only ES module compatibility concerns. If there is a desire to add support for ESM-based configurations, it will need to be addressed in another RFC.

## Related Discussions

- [Issue #12319](https://github.com/eslint/eslint/issues/12319)
- [PR #12321](https://github.com/eslint/eslint/pull/12321)
- [Transition Path Problems for Tooling - node/modules](https://github.com/nodejs/modules/issues/388)
