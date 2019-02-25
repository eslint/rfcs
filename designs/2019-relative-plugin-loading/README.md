- Start Date: 2019-02-25
- RFC PR: https://github.com/eslint/rfcs/pull/14
- Authors: Teddy Katz

# Load plugins relative to the configs that reference them

## Summary

This change would update ESLint's config-processing logic to load plugins relative to the configs that reference those plugins. ESLint would throw an error when it encounters multiple plugins with the same name. Additionally, this change would also add support for loading plugins by filepath.

## Motivation

Note: The term "ESLint invoker" is used below to refer to the party that installs and directly invokes ESLint. The term "end user" is used to refer to the party that writes the code which gets linted. In most cases, the ESLint invoker is also the end user. The two parties are different when a user uses a third-party tool that invokes ESLint for them, such as `standard` or `react-scripts`.

ESLint requires that plugin names are globally unique during any given invocation, to ensure that users can unambiguously refer to rules from plugins using strings like `pluginName/ruleName`. To satisfy this requirement, we recommend that shareable configs declare plugins as `peerDependencies`, and we require the ESLint invoker to install those plugins themselves. Then, ESLint loads the plugins relative to the current working directory (assuming the [2018 simplifed package loading RFC](https://github.com/eslint/rfcs/tree/3a2cac4078b6d35c7cca817531019043b3f59aa6/designs/2018-simplified-package-loading) is implemented).

The current system has a few problems:

* It's inconvenient that that the ESLint invoker needs to manually install any plugins required by a shareable config. As a result of this constraint, a shareable config can't add a plugin without requiring new manual installation steps from ESLint invokers. This greatly decreases the ergonomics of using custom rules from plugins in shareable configs, and results in increased pressure to add new rules and options to ESLint core.
* ESLint loads plugins from the CWD of the process, which is typically somewhere in the end user's project. However, with the current setup the correct place to load plugins from is the ESLint invoker's project. As a result, if the ESLint invoker is not the end user (e.g. in `create-react-app` projects), the default plugin-loading location is incorrect. The ESLint invoker can change the CWD, but changing the CWD has some inconvenient side-effects:
    * Only the `node_modules` folder in the CWD is ignored by default, so changing the CWD would cause some files to be unignored.
    * Editor integrations usually won't know to change the CWD (since they might not be able to figure out who the "intended" ESLint invoker is for a given project), so they might fail to load the correct plugins.
* Users often want to write custom project-specific rules and put them in the project's repository. However, plugins currently need to be in `node_modules`, because allowing other paths for plugins would introduce the possibility of name conflicts.

## Detailed Design

The design has three components:

* Plugins are resolved relative to the config files that depend on them, rather than being loaded relative to the ESLint invoker's project.
* Plugin name conflicts result in a fatal error.
* Plugins in configs can be loaded by filepath.

### Plugins are resolved relative to the config files that depend on them

A config is said to "load" a given plugin name if either of the following are true:

* The config includes the plugin name in a `plugins` list, e.g. `plugins: ["foo"]`
* The config extends a shareable config from the plugin, e.g. `extends: ["plugin:foo/bar"]`

To load a plugin from a config, the following steps are taken:

* If the config loading the name `eslint-plugin-X` is itself a bundled config in plugin called `eslint-plugin-X`, then the load resolves to that plugin itself. (This is called a *self-load*.)
* Otherwise, the plugin is loaded as a Node module relative to the location of the config file that loads the plugin's name.

**Open question:** Should `extends: ["plugin:foo/bar"]` be considered a "load" of a plugin? (The "Open Questions" section at near the bottom of this RFC contains more details on how this would affect behavior. To understand the tradeoffs, it may be easier to read through this RFC first before considering the open question.)

### Plugin name conflicts result in a fatal error

If at least two different config files load the same plugin name, excluding self-loads, then a fatal config error is thrown.

For example, if a user uses a shareable config that loads `eslint-plugin-foo`, then the user's config file cannot include `plugins: ["foo"]` or extend any of `eslint-plugin-foo`'s bundled configs. The user also cannot extend any other shareable configs that depend on a plugin called `eslint-plugin-foo`. However, the user can still configure rules from the plugin like `foo/rule-name` (configuring rules is not considered to "load" to a plugin, using the definition above).

Since the check for duplicates excludes self-loads, a user's config is allowed to contain something like `extends: ["plugin:bar/baz"]` even if the `baz` config in `eslint-plugin-bar` also contains `plugins: ["baz"]`. Additionally, since the duplicate check appears on the granularity of a single config file, a single config file is allowed to contain both `extends: ["plugin:bar/baz"]` and also `extends: ["plugin:bar/quux"]`.

Note that loads of the same plugin name in two different configs are forbidden even if the two loads would end up resolving to the same plugin. (Allowing these loads could cause robustness problems, because the loads might later point to two different plugins as a result of a patch release, causing the user's ESLint build to break. See the "Alternatives" section for more details.)

### Plugin names in configs can be relative or absolute paths

Currently, ESLint always prepends `eslint-plugin-` to plugin names before resolving them, unless the names already start with `eslint-plugin-`. As a side-effect, relative paths are effectively not allowed in the `plugins` array in config files. (If a config file contains something like `plugins: ["./foo"]`, ESLint will currently attempt to load the Node module `eslint-plugin-./foo`, almost always resulting in a plugin-not-found error.) This constraint has been in place because allowing users to load relative plugins would introduce the possibility of a plugin name conflict.

With this change, ESLint now has a mechanism for handling plugin name conflicts, so it can now support this case. A minor wrinkle is that plugins still must have names, so ESLint imposes the constraint that paths in `plugins` arrays must end in `eslint-plugin-<something>`. To describe the changes in more detail:

If a value in a `plugins` array starts with any of the following:

* `.` (i.e. a relative path)
* `/` or `\` (i.e. a Unix or Windows absolute path)
* A capital or lowercase ASCII letter followed by `:` (i.e. a Windows absolute path with a drive letter)

...then the value is interpreted as a path to a plugin. When this happens:

* The last part of the path must be `eslint-plugin-SOME_PLUGIN_NAME` with an optional file extension. `SOME_PLUGIN_NAME` must consist of only lowercase/uppercase ASCII letters, digits, and underscores. If these constraints are not satisfied, the config is invalid and a fatal error is raised.
* ESLint loads the plugin as a Node module, resolving relative paths from the config file that contains the `plugins` directive. The name of the plugin (e.g. to be used when configuring its rules) is `SOME_PLUGIN_NAME`.
* If the same config file also extends a config like `plugin:SOME_PLUGIN_NAME/foo` or `plugin:eslint-plugin-SOME_PLUGIN_NAME/foo`, then the name resolves to the same plugin as the corresponding name from the `plugins` array. Similarly, the plugin's configs can self-load the plugin by referring to it as `SOME_PLUGIN_NAME`.
* If a different config file loads a plugin called `SOME_PLUGIN_NAME`, then a duplicate-plugins error is raised as usual.

Note that this will require ESLint to process `plugins` entries before `extends` entries within a single config file, since the meaning of `extends: ["plugin:foo/bar"]` is different depending on whether the config file also contains something like `plugins: ["./eslint-plugin-foo"]`. However, ESLint can still process the `extends` entries of a parent config file before the `plugins` entries of a child config file, because if the parent uses `extends: ["plugin:foo/bar"]` and the child uses `plugins: ["./eslint-plugin-foo"]`, then the configs would be invalid regardless of procesing order since they load the same plugin name twice.

Since this functionality would allow users to easily create project-specific rules, this RFC also deprecates the `--rulesdir` CLI flag.

## Documentation

With this change, we would update the documentation for shareable configs to encourage shareable config authors to include plugins as `dependencies` rather than `peerDependencies`. However, we would need to note that the set of plugin names used by a shareable config is still part of the config's public API. Adding a new plugin to a shareable config would still be a breaking change, because it could conflict with another one of the end user's plugins. (In most cases, adding a plugin is a breaking change anyway because it usually entails enabling new rules.)

Additionally, it might be a good idea to add a section to the documentation with a tutorial on how to use project-specific rules (as a replacement for `--rulesdir`).

As discussed in the compatibility analysis section, this would be a breaking change, so it would also be discussed in the migration guide.

## Drawbacks

### Users can't use plugin-provided configs for transitive plugin dependencies

In other words, if an end user depends on `eslint-config-foo`, and `eslint-config-foo` depends on `eslint-plugin-bar`, then the end user is not allowed to use `extends: [plugin:bar/baz]`, because this would result in duplicate loads of the `bar` plugin. The end user could still manually configure `bar`'s rules to achieve the same effect, but this is less convenient and users might not understand how to do that.

It's possible that this drawback can be eliminated (see the "Open Questions" section).

### Users have no recourse when they encounter a conflict

Instead of providing a disambiguation mechanism for duplicate plugins, this proposal raises a fatal error for them. A potential disadvantage is that users might encounter plugin conflicts, and have no way to resolve those conflicts. (For example, a users's shareable config uses a plugin and the user wants to upgrade that plugin, or if the user has two shareable configs that depend on the same plugin.) This makes some config setups impossible. These config setups are believed to be relatively rare. (Note that these some of these config setups are already impossible with the current ESLint behavior, if two configs depend on incompatible versions of the same plugin.)

### The `plugins` property in a config has a slightly different meaning, potentially causing confusion

Previously, if a shareable config had `plugins: ["bar"]`, then adding `plugins: ["bar"]` to the end user's config would be a no-op. With this change, adding `plugins: ["bar"]` to the end user's config would be invalid because it would introduce a naming conflict. The confusion could potentially be mitigated by having useful error messages (e.g. recommending that `bar` be removed from the plugins list for a parent-child config conflict, and stating that two shareable configs can't be used together for a sibling-sibling conflict).

## Backwards Compatibility Analysis

This change is backwards-incompatible in a few cases where new plugin name conflicts are introduced.

Note that the term "extends" is used in this section to apply to both (a) the `extends` config file directive (where config `A` has `extends: B`), and (b) the analogous cases where `.eslintrc` files in parent directories implicitly extend `.eslintrc` files in subdirectories through config cascading.

### A config extends more than one other config, each of which loads the same plugin name

Previously, this was allowed, with the caveat that the ESLint invoker had to install the plugins manually. With this change, it's not possible to use both shareable configs simultaneously.

### A config contains `plugins: ["foo"]`, and extends a config that also uses the `foo` plugin

Previously, `plugins: ["foo"]` in the parent config would be a no-op. With this change, it would be an error. A user could fix their config by simply removing the `plugins: ["foo"]` directive.

### A config contains `extends: ["plugin:foo/bar"]` and extends a config that also uses the `foo` plugin.

Previously, this was allowed, with the caveat that the ESLint invoker had to install the plugin manually. With this change, this would be disallowed because it would create a duplicate load of the `foo` plugin. As a workaround, the user could manually change their config rather than extending from the plugin config. (See the "Open Questions" section for a potential way this could be avoided.)

## Alternatives

### Add a `CLIEngine` flag to avoid side-effects of changing CWD

We could add a `CLIEngine` flag that determines the path where plugins should be loaded from, to separate it from the CWD. This would make it more convenient for ESLint invokers to change where plugins are loaded from. It would not solve the ergonomics issues of using shareable configs with plugins, or the potential errors from editor integrations (which might not know how they should set the flag).

### Allow disambiguation of plugin names

Instead of raising an error for duplicate plugin names, we could provide a way to disambiguate rule references, as proposed in https://github.com/eslint/rfcs/pull/5. However, this would introduce a lot of complexity, and it's not clear how often users actually want to load two configs that depend on the same plugin. Raising an error for this case should allow us to compatibly introduce a disambiguation mechanism in the future, if needed.

### Use a different strategy for determining whether two plugins conflict with each other

A few possibilities include:

* Instead of raising an error when two plugins with the same name are loaded, ESLint could simply pick one of the plugins to use, or try to merge the plugins. However, this could create confusing issues, because ESLint would be applying rule configurations for one version of a plugin to another version of a plugin, which could result in invalid configurations.
* Instead of *always* raising an error when two plugins with the same name are loaded, another possibility would be to only raise an error *if the two plugins are in different locations on disk*. (For example, if the two plugins were the same package and were deduplicated by a package manager, they would be allowed.) However, this would create robustness issues for end users, because conflicts could suddenly arise as a result of a patch release in a plugin. (Also see the overview [here](https://gist.github.com/not-an-aardvark/169bede8072c31a500e018ed7d6a8915).)
* We could allow multiple loads to the same plugin name in two different config files, as long as the config files were considered to be part of the same "package" (under the assumption that two files in the same "package" would always resolve the the same plugin). However, this could be confusing since it's not clear that there is a good working definition of a "package" for this case, and the problem it solves can already be worked around by shareable config authors.

### Relax the filename requirements for path-loaded plugins

This RFC requires path-loaded plugins to have names ending in `eslint-plugin-`. Alternatively, we could just use the last segment of the path to determine a plugin name without requiring it to end with `eslint-plugin-`. However, this could cause some confusion, e.g. if creates a config with `plugins: ["./foo/bar/index.js"]`, then the plugin would end up being called `index`. Given that users can control filenames in their own projects, it seems better to require users to make the name explicit, since this also reduces the liklihood of plugin name conflicts.

## Open Questions

### Should `extends: ["plugin:foo/bar"]` be considered a "load" of a plugin?

(Also see: the section titled "Users can't use plugin-provided configs for transitive plugin dependencies" under "Drawbacks".)

Background: Currently, the `plugins:` config file directive loads a plugin from the CWD, and also introduces its rules and environments into a global namespace. The `extends: ["plugin:foo/bar"]` pattern loads a plugin from the CWD, and extends one of its configs (without necessarily making its rules and environments globally available). As a result, it's possible for a config to use `extends: ["plugin:foo/bar"]` without also using `plugins: ["foo"]`, as long as *someone* uses `plugins: ["foo"]` to load the plugin's rules. In fact, our documentation encourages plugin authors to put `plugins: ["thePluginName"]` in plugin-provided configs (i.e. using self-loading) so that end users don't have to include the `plugins` directive.

In other words, `extends: ["plugin:foo/bar"]` has two effects: it loads a plugin, and it also extends that plugin's config. With this RFC, "loading a plugin" works differently depends on which config file loads the plugin, since plugins are resolved relative to config files. As a result, if a plugin has already been loaded by a shareable config, a user sometimes just wants to extend the plugin's config without separately loading that plugin from their own config. Unfortunately, `extends: ["plugin:foo/bar"]` does both, so using `extends: ["plugin:foo/bar"]` raises an error with this RFC. (To illustrate why the error happens: If the end user decided to separately install a different version of `eslint-plugin-foo` from the shareable config, then `plugin:foo/bar` in the end user's config would resolve to the end user's version of `eslint-plugin-foo`, and `plugins: ["foo"]` in the shareable config would resolve to the shareable config's version of `eslint-plugin-foo`.)

This is inconvenient, because end users might want to extend plugin configs for plugins introduced by their styleguide. Setting compatibility aside, it seems like it would be better if `extends: ["plugin:foo/bar"]` could only refer to configs from already-loaded plugins, similar to how `rules: { "foo/baz": "error" }` can only refer to rules from already-loaded plugins.

However, this might be bad for compatibility (end users would have their configs broken if they were extending a plugin config without also using `plugins:`). We could have a fallback where `extends: ["plugin:foo/bar"]` loads `eslint-plugin-foo` only if no plugins called `foo` have been loaded yet. If we do this, I think we should print a deprecation warning and remove the fallback in the following major release. It could cause some confusion where `extends: ["eslint-config-baz", plugin:foo/bar"]` works correctly while `extends: ["plugin:foo/bar", "eslint-config-baz"]` fails in a confusing way (since two copies of `eslint-plugin-foo` would be loaded, one of them the fallback and one of them from within `eslint-config-baz`).

## Frequently Asked Questions

None yet

## Related Discussions

* [eslint/eslint#3458](https://github.com/eslint/eslint/issues/3458): Feature request to allow shareable configs to bundle their plugins
* [eslint/eslint#6237](https://github.com/eslint/eslint/issues/6237): Feature request to allow loading plugins by path
* [eslint/eslint#8769](https://github.com/eslint/eslint/issues/8769): Discussion about the difficulty of using project-specific linting rules
* [eslint/rfcs#5](https://github.com/eslint/rfcs/pull/5): Alternate proposal that allows users to disambiguate conflicting plugin names, rather than throwing an error for them
* [eslint/rfcs#9](https://github.com/eslint/eslint/issues/9): Alternate proposal that creates a new config format where shareable configs can explicitly call `require` on their plugins
* [eslint/rfcs#13](https://github.com/eslint/rfcs/pull/13): Alternate proposal that makes similar changes as this RFC (with the exception of path-based plugin loading), and also makes some other changes to config files
