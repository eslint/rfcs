- Start Date: 2019-10-05
- RFC PR: [#12321](https://github.com/eslint/eslint/pull/12321)
- Authors: Evan Plaice ([@evanplaice](https://github.com/evanplaice))

# ES Module Compatibility

## Summary

ES module support has been rolled out in all evergreen browsers and will soon land (unflagged) in node. Now is a good time to fill in the gaps to ensure ESLint users can effectively lint their ESM-based packages.

## Motivation

The new ES module standard is finally rolling out but, there already exists a massive ecosystem of packages build as CommonJS modules (incl ESLint).

In the majority case, changing a package to be interpreted as an ES module by default is as simple as adding `"type": "module"` to `package.json`.

In excepitonal edge cases it may require extra work to establish interoperability across package boundaries. The way ESLint uses `.js` config files is one of those edge cases.

The goal of this RFC is to explore the different options for addressing this issue.

## Detailed Design

To understand the design outlined here requires some background info into the mechanics by which Node modules are loaded.

### Jargon

- Package - A NPM package (ie contains `package.json`)
- Module - A JS file
- CJS - A CommonJS module
- ESM - A Standard ECMAScript module
- Consumer - A package that uses/depends on this package
- Dependent - Package(s) that this package needs to work
- Boundary - The demarcation between package, consumer, dependent

### A Crash Course on Package Boundaries

Historically, NPM packages have come in a variety of different package formats (ie IIFE, AMD, UMD, CJS). Prior to an actual spec, NPM settled on CJS as it's de-facto module format. CJS isn't going anywhere, the addition of ESM is additive.

By default, all NPM packages are CJS-based. That means all `.js` files will be read as CJS modules. To include an ES module in a CJS package, it must have the `.mjs` extension.

In addition, if `package.json` contains `"type": "module"` then the package is ESM-based. Meaning, all `.js` files contained within will be treated as ESM. To include a CJS module in a ESM-based package, it must have the `.cjs` extension.

This configuration option does *not* affect consumers or dependents. Their functionality depends entirely on their own configuration. Assuming all packages follow this rule, there should be no issues with interoperability between CJS/ESM.

### The Scenario

A user is creating a new package. They want it to be ESM for Browser <-> Node compatibility so they configure their package to be ESM-based.

The user adds `eslint` as a devDependency, w/ a typical NPM script to lint the package contents.

The user defines `eslint.config.js` outlining the ruleset they prefer to use within their package.

### The Issue

The user pacakge is ESM-based, all `.js` files within are read as ESM.

ESLint is CJS-based, it loads all files within it's package boundary as CJS.

The configuration file is defined as a CJS module (ie `module.exports`), but has the `.js` syntax so requiring it throws an error. ESLint, by design reaches across the package boundary to load the configuration but the user has inadvertently signaled to Node that it uses the wrong module type.

### The Fix

Rename `eslint.config.js -> eslint.config.cjs`

There is no documentation outlining how to define a ESM-based configuration. As such, it can be assumed that any JS-based ESLint config will always be a CJS module.

Therefore, the `.cjs` extension needs to be assiciated w/ JS-based configrations.

*/lib/cli-engine/config-array-factory.js*

```diff
function loadConfigFile(filePath) {
    switch (path.extname(filePath)) {
        case ".js":
+        case ".cjs:
            return loadJSConfigFile(filePath);

        case ".json":
            if (path.basename(filePath) === "package.json") {
                return loadPackageJSONConfigFile(filePath);
            }
            return loadJSONConfigFile(filePath);

        case ".yaml":
        case ".yml":
            return loadYAMLConfigFile(filePath);

        default:
            return loadLegacyConfigFile(filePath);
    }
}

```

If a user:

- is building a ESM-based package
- is using a JS-based configuration

They should give the configuration a `.cjs` extension.

## Documentation

A quick mention in the [FAQ](https://github.com/eslint/eslint#frequently-asked-questions) should be suitable to document usage.

## Drawbacks

### Technical

None. The change has no negative functionality or performance impacts.

## People

Some devs within the Node ecosystem are strongly opposed to supporting `"type": "module"` based packages at all.

## Backwards Compatibility Analysis

*tl;dr: Backward compatibility will be unaffected*

### Story 1 - ESLint Compatibility

This change is additive. It associates `.cjs` files with the existing Javascript configuration loader.

The actual semantics of loading the CommonJS config do not change at all. The `.cjs` extension specifically exists as a escape hatch for this use case (ie loading a CommonJS file that exists in a ESM-based package).

### Story 2 - Node Compatiblity

ESM support rolls out (as a non-experimental feature) w/ Node 13. This includes: the ability to define a package as a ESM; and backward interoperability w/ CJS in ES modules by using the `.cjs` file extension.

This change *only* impacts users who define their package as an ES module. Without Node 13+ that isn't possible.

Existing packages that use the default (ie CommonJS) will be unaffected.

## Alternatives

### Alternative 1 - Suppress the error

**[PR#12333](https://github.com/eslint/eslint/pull/12333)**

There is some debate among the node/modules group surrounding whether the error that causes this issue should be removed altogether.

The fencing exists to prevent users from creating dual-mode (ie both ESM/CJS) packages as interoperability between the two formats has issues. In the case of ESLint, loading a simple config file from a user-defined package should not cause any issues or side-effecs.

Eating the exception is viable solution. Although, this may become dead code if the error is removed from Node's core.

### Alternative 2 - Support dynamic `import()`

Instead of using 'import-fresh' to require the config, import it in a cross-format-compatible way using dynamic `import()`.

There has been and will continue to be a lot of talk about using this approach as the CJS/ESM interop story is fleshed out.

For now it presents too many unknowns and -- even if it didn't -- adding `import()` to ESLint's core would force Node 13+ as a hard constraint.

### Alternative 3: Deprecate JS-based configurations altogether

This approach is controversial but this RFC would be incomplete w/o mentioning it.

What is the benefit of supporting JS-based configurations at all?

- JSON (w/ `strip-comments`) and YAML both support comments
- JS will load slower than it's context-free alternatives
- Do the config definitions really require a turing-complete syntax?
- Is the added complexity of supporting JS configs worth it?

Benefits:

- less complexity
- reduces ambiguity (ex CJS vs ESM)
- removes the `import-fresh` dependency

Downsides 

- major breaking change w/ existing patterns
- JSON requires all strings to be quoted
- seriously, major breaking changes are not fun

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

ESLint reaches across the package boundary to retrieve the user-defined configuration. If the consumer package is ESM-based, the `.js` config file will be seen as an ES module, and trying to `require()` it will throw an error.

> What does this interop issue affect?

Only ESM-based packages (ie `"type": "module"`).

> What about 3rd-party linting tools that provide their own built-in ruleset for ESLint (ex [StandardJS](https://standardjs.com/))?

They aren't impacted by this change. 3rd-party modules define both their code and the ESLint ruleset used within the same package boundary.

## Related Discussions

- [Transition Path Problems for Tooling - node/modules](https://github.com/nodejs/modules/issues/388)

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
