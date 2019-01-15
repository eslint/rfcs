- Start Date: 2018-12-27
- RFC PR: https://github.com/eslint/rfcs/pull/7
- Authors: Teddy Katz ([@not-an-aardvark](https://github.com/not-an-aardvark))

# Simplify the resolution of third-party plugins/configs/parsers

## Summary

This change would update ESLint to load plugins relative to the user's project root, and other packages relative to where they're specified in a config. This would simplify ESLint's package-loading, resulting in fewer confusing errors about missing packages. It would also fix an existing design bug where ESLint would sometimes fail to load plugins in valid setups.

## Motivation

Currently, ESLint plugins and shareable configs are usually loaded from the location of the ESLint package itself (depending on a complex mechanism described in the appendix below). This leads to several problems:

* The current behavior assumes that if a user installs a config/plugin and ESLint in the same project, then ESLint will be able to load that config/plugin. This relies on an implementation detail of how npm works rather than a specified behavior, which leads to problems when using other package management strategies, e.g. with `lerna`. (More details about this problem can be found in [eslint/eslint#10125](https://github.com/eslint/eslint/issues/10125).) It also creates problems when using ESLint as a non-top-level dependency with package management strategies that strictly enforce dependency validity with Yarn Plug 'n Play.
* The current behavior leads to a large amount of confusion from users where ESLint behaves differently depending on whether it's installed "globally" with npm. In general, users have the expectation that if they can call `require('foo')` on a package from a REPL in their project, or they specify `foo` as as a `devDependency`, then the package has been installed successfully and will work for their project. With ESLint's unusual package-loading behavior, this is not always the case, breaking user intuition about whether a package has been installed correctly. As a result, many users end up in a frustrating state where they "installed a package but ESLint can't find it for some reason".
* The current behavior results in odd quirks where ESLint's behavior changes depending on what packages ESLint uses as dependencies. For example, it's impossible for a user to install a different version of the `espree` parser and make ESLint use it, because ESLint has its own version of `espree` and will always load it instead of the user's parser.
* The current behavior is very complex, amplifying user confusion and making it difficult to maintain and enhance config-loading behavior.

### Appendix: Current package-loading behavior

This section describes the *current* package-loading behavior (before implementing this RFC) for various types of packages, and notes any oddities for posterity/compatibility analysis.

* Plugins are loaded by calling `require(pluginName)` relative to `lib/config/plugins.js`. Since plugin names always have `eslint-plugin-` prepended, they can't contain relative paths, so they are effectively always loaded from the root of the `eslint` package.
* Parsers and shareable configs are loaded in similar ways:
    * If a shareable config or parser is referenced from a config file which is inside the same `node_modules` folder as the `eslint` package, ESLint attempts to load it from the sibling `node_modules` folder of the config where the parser is referenced. If it's not found there, then it's effectively loaded from the `eslint` package rather than using Node's regular `node_module` cascade (where Node looks for packages `node_modules` folders in parent directories).
    * If referenced from a config file which is *not* inside the same `node_modules` folder as the `eslint` package (including config files in the end-user's project), extended configs and parsers are loaded relative to the `eslint` package.
    * If referenced outside a config file (e.g. on the command line), parsers are loaded relative to the `lib/linter.js` within the `eslint` package.

The current behavior gives the impression that ESLint's config-loading logic evolved somewhat haphazardly in response to new use cases, while still adding patches to maintain compatibility with all existing use cases. Unfortunately, this led to substantial complexity in implementation and behavior, making it difficult for users and maintainers to understand how ESLint was loading packages, and leading to subtle design bugs like [eslint/eslint#10125](https://github.com/eslint/eslint/issues/10125). This RFC should be seen as a substantial simplification to the config-loading model which leaves more behavior to Node's module-resolution algorithm rather than implementing path resolution from scratch, and provides a more consistent mental model for future changes.

## Detailed Design

### Changes to package-loading

With the new design, ESLint will load all plugins and formatters relative to the *project root*. By default, the project root is simply the CWD of the running process. Integrations can configure a specific project root using the already-existing `cwd` option in `CLIEngine`. ESLint will load all other third-party packages (e.g. shareable configs and parsers) relative to the config file where they're referenced.

In other words, if a config specifies `plugins: ['foo']`, ESLint will find the same package as the user would find by calling `require('eslint-plugin-foo')` from a file in the CWD. If a config specifies `extends: ['bar']`, ESLint will find the same package as the config author would find by calling `require('eslint-config-bar')` from within that config file. This would be implemented either by using Node's built-in APIs or by using some of the functionality from a package like [`resolve-from`](https://www.npmjs.com/package/resolve-from).

There is one exception: If a config specifies the string `"espree"` as a parser, but the `espree` package cannot be loaded from the config, then ESLint's `espree` dependency will be used as a fallback instead of throwing a fatal error. This exception exists to provide compatibility for shareable configs that explicitly specify `parser: "espree"` with the intention of using the package bundled with ESLint.

### Changes to the `Linter` API

Additionally, the `Linter` API will change with regard to how parsers are loaded. Previously, an API consumer could use `Linter#defineParser` to load a parser with a particular name, and any unknown parsers would be resolved by calling `require` on the parser name from the `lib/linter.js` file.

The `require` fallback for parsers was the only place where `Linter` dynamically accessed the filesystem, and it was present only for backwards compatibility. Since parser-loading behavior would need to change with this RFC anyway, this seems like a good time to get rid of the exception in `Linter`.

With this change, the `require` fallback will be removed, and API consumers must use `defineParser` to load custom parsers with `Linter`. The parser map will contain a mapping from the string `"espree"` to ESLint's `espree` dependency by default. (In other words, it would not be necessary for integrations to use `defineParser` when using the default parser.)

## Documentation

This change would effectively remove the distinction between "local" and "global" ESLint installations, because the location of the ESLint package would no longer be relevant. As a result, we would need to update the documentation that describes the distinction between local and global installations, and more generally modify the parts of the documentation that describe config loading to ensure that they were up to date.

Our existing advice for config/plugin authors regarding dependencies (namely, that parsers and configs can be dependencies, while plugins must be peerDependencies) would remain the same.

We would also need to update the `generator-eslint` boilerplate for plugins to avoid generating a readme file that describes a distinction between local and global installations.

## Drawbacks

* A lot of existing documentation says that global ESLint installations require global plugin installations. (This includes README files in third-party plugins which we're unable to modify.) As a result, this change could temporarily cause confusion for new users who choose to use global installations.
* This change is backwards-incompatible for end users who use global plugin installations or home-directory configs, which could cause some migration pain. (However, the change is compatible with existing shareable configs and plugins, so end users would be able to migrate without waiting for a third party to migrate.)
* This change does not allow shareable configs to manage their own plugins, as proposed in [RFC #5](https://github.com/eslint/rfcs/pull/5). However, it is intended to be compatible with future solutions to that problem.

## Backwards Compatibility Analysis

* This change has no compatibility impact on shareable configs and plugins. Shareable configs already specify plugins as `peerDependencies`, implying that the plugins must be installed and loadable by the end user. This change simply updates ESLint to use that guarantee.
* This is a breaking change for end users who use global ESLint installations and global plugin installations; the plugins must now be installed locally.
* This is a breaking change for end users who use home-directory configs along with shareable configs or custom parsers; the shareable configs/parsers must be resolvable from the location of the home-directory config.
* This is a breaking change for integration authors who depend on `Linter` loading custom parsers from the filesystem. These integration authors could explicitly use `Linter#defineParser` instead. (Most integration authors who need access to the filesystem are using `CLIEngine` anyway.)

## Alternatives

* Along with configs/parsers, ESLint could load plugins relative to the configs that depend on them. That would introduce the possibility of having two plugins with the same name, requiring a disambiguation mechanism for configuring that plugin's resources; see [RFC #5](https://github.com/eslint/rfcs/pull/5) for an example. This RFC is intended to be a simpler step to solve some problems, while allowing additional work to proceed on that problem if desirable.
* To mitigate some compatibility impact, ESLint could fall back to the existing behavior when it fails to load a package using the new behavior, instead of throwing a fatal error. However, this would likely cause more confusion because users wouldn't understand why ESLint was finding their packages in certain cases, which would make it more difficult to debug problems. This fallback is only provided for the `espree` package (which falls back to the default parser if no user-installed version of the pacakge is found).

## Open Questions

* This proposal uses the CWD as the "package root" from which plugins are loaded. Are there other folders that would serve this purpose better (e.g. in cases where the `--config` flag is used to load a config from a separate project)?

## Related Discussions

* [eslint/eslint#10125](https://github.com/eslint/eslint/issues/10125)
* [eslint/rfcs#5](https://github.com/eslint/rfcs/pull/5)
* [eslint/eslint#3458](https://github.com/eslint/eslint/issues/3458)
* [eslint/eslint#6732](https://github.com/eslint/eslint/issues/6732)
* [eslint/eslint#9897](https://github.com/eslint/eslint/issues/9897)
* [eslint/eslint#10643](https://github.com/eslint/eslint/issues/10643)
