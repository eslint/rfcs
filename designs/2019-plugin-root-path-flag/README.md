- Start Date: 2019-03-13
- RFC PR: https://github.com/eslint/rfcs/pull/18
- Authors: Teddy Katz

# Add a flag to specify the folder where plugins are resolved from

## Summary

With the [2018-simplified-package-loading RFC](https://github.com/eslint/rfcs/blob/8bc0b80e0b3e54d10991a4774c41f7375dfcbbfe/designs/2018-simplified-package-loading/README.md) implemented, ESLint always resolves plugins relative to the current working directory. The CWD works well for the most common use case, but is inconvenient for certain integrations. This RFC proposes adding a CLI flag to specify an alternative place where plugins should be resolved from.

## Motivation

We require whoever invokes ESLint to also install any necessary plugins. Generally speaking, in order to ensure that plugins are resolved correctly, ESLint should resolve plugins relative to whichever project installed the plugins. As a result, ESLint would ideally resolve plugins relative to the package that invokes ESLint.

However, we can't reliably tell who is invoking ESLint, so the [2018-simplified-package-loading RFC](https://github.com/eslint/rfcs/blob/8bc0b80e0b3e54d10991a4774c41f7375dfcbbfe/designs/2018-simplified-package-loading/README.md) always loads plugins relative to the CWD. This works well when the end user invokes ESLint, because the CWD is usually somewhere in the end user's project. Unfortunately, it causes problems for third-party integrations that invoke ESLint on behalf of the end user, such as `standard` and `create-react-app`. The plugins used in these integrations are transitive dependencies for the end user, so they might not be reachable from the CWD.

It is possible for these integrations to work around the issue by changing the CWD (with the `cwd` option in `CLIEngine`), but changing the CWD has a number of inconvenient side-effects. For example, if the `cwd` is set to the directory of a third-party package, the end user's `node_modules` folder is not ignored by default, and relative paths might be resolved in an unexpected way.

## Detailed Design

This RFC adds a `--resolve-plugins-relative-to` CLI flag, and a corresponding `resolvePluginsRelativeTo` option in `CLIEngine`. The option value, if provided, must be an absolute path to a directory.

When provided, ESLint loads all plugins relative to the `resolvePluginsRelativeTo` directory, rather than the CWD. The expectation is that integrations like `standard` and `create-react-app`, which already use `CLIEngine`, would pass an option like `{ resolvePluginsRelativeTo: __dirname }` to indicate that the package's own plugins should be used regardless of the CWD.

The implementation is expected to be fairly simple when building on top of [eslint/eslint#11388](https://github.com/eslint/eslint/pull/11388). That implementation already has an equivalent directory path that gets passed around to specify plugin loading, but that path currently is always the same as the CWD.

(Note: In the [2018-simplified-package-loading RFC](https://github.com/eslint/rfcs/blob/8bc0b80e0b3e54d10991a4774c41f7375dfcbbfe/designs/2018-simplified-package-loading/README.md), the `resolvePluginsRelativeTo` was referred to as the "project root".)

## Documentation

This feature might be useful to add as a footnote for the larger announcement of package-loading changes. It would be worth mentioning in the parts of the documentation that describe how plugins are loaded, and it would also be added to the Node.js docs. However, most end users would not need to use this option or be aware of its existence.

## Drawbacks

Like any new feature, this flag will slightly increase the complexity and maintenance costs of ESLint core.

If we change how plugins are loaded in the future, this flag may become obsolete or turn into a no-op.

## Backwards Compatibility Analysis

This change is backwards-compatible. It adds a new CLI flag while keeping the behavior the same if the flag is not specified.

## Alternatives

As described in [#14](https://github.com/eslint/rfcs/pull/14), We could change how plugin-loading works to always resolve plugins from the config that imports them, eliminating the need to load things from the CWD. That change would eliminate the need for this command-line flag. However, that change presents complex compatibility and design challenges which are still under discussion. If we change how the `plugins` directive works, it seems like it won't happen before v7.0.0, and this flag would be important to have in the meantime. Given that a user can already somehwat customize this behavior by changing the CWD, the existence of this flag is unlikely to impose a substantial compatibility burden even if we do change how plugins are loaded later.

## Open Questions

None


## Frequently Asked Questions

None yet

## Related Discussions

* [#7](https://github.com/eslint/rfcs/pull/7)
* [#14](https://github.com/eslint/rfcs/pull/14)
