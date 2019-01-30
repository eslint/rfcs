- Start Date: 2019-01-20
- RFC PR: https://github.com/eslint/rfcs/pull/9
- Authors: Nicholas C. Zakas (@nzakas)
- Contributors: Teddy Katz (@not-an-ardvaark), Toru Nagashima (@mysticatea)

# Config File Simplification

## Summary

This proposal provides a way to simplify configuration of ESLint through a new configuration file format. The configuration file format is written in JavaScript and removes several existing configuration keys in favor of allowing the user to manually create them.

## Motivation

The ESLint configuration file format (`.eslintrc`) hasn't changed much since ESLint was first released. As we have continued to add new features, the configuration file format has continued to be extended and evolve into a state where it is the most complicated part of ESLint. The `.eslintrc` way of doing things means a variety of different merging strategies (cascading, extending, overriding), loading strategies (for plugins, processors, parsers), and naming conventions (`eslint-config-*`, `eslint-plugin-*`).

Although `.eslintrc` has proven to be a very robust format, it is also limited and unable to easily accommodate feature requests that the community favors, such as:

1. [bundling plugins with configs](https://github.com/eslint/eslint/issues/3458)
1. [specifying ignore patterns in the config](https://github.com/eslint/eslint/issues/10891)
1. [specifying `--ext` information in the config](https://github.com/eslint/eslint/issues/11223)
1. [using `extends` in `overrides`](https://github.com/eslint/eslint/issues/8813) 
1. [Customize merging of config options](https://github.com/eslint/eslint/issues/9192)


The only reason that these requests are difficult to implement in ESLint is because of how complex the `.eslintrc` configuration format is. Any changes made to any part of `.eslintrc` processing end up affecting millions of ESLint installations, so we have ended up stuck. The complicated parts of `.eslintrc` include:

1. Resolution behavior of modules (`plugins`, `parser`, `extends`)
1. Merging of cascading config files in a directory structure
1. Merging of config files via `extends`
1. Overriding of configuration via `overrides`

All of these complexities arise from `.eslintrc` format of describing what should happen rather than how it should happen. We keep trying to anticipate how users want things to happen and guessing wrong has led to increased complexity.

This RFC proposes a fresh start for configuration in ESLint that takes into account all of the requests we've received over the years. By simplifying and stripping configuration we can actually make a configuration system that is flexible enough to accomodate changes in the future and easy enough to implement that we won't be afraid to do so.

## Detailed Design

Design Summary:

1. Introduce a new `eslint.config.js` configuration file format
1. Searching for `eslint.config.js` starts from the current working directory and continues up
1. All `eslint.config.js` files are treated as if they have `root: true`
1. There is no automatic merging of config files
1. The `eslint.config.js` configuration is not serializable and objects such as functions and plugins may be added directly into configuration
1. Remove the concept of environments (`env`)
1. Remove `.eslintignore` and `--ignore-file` (no longer necessary)
1. Remove `--rulesdir`
1. Create a `@eslint/config` package to help users manager configs

### The `eslint.config.js` File

The `eslint.config.js` file is a JavaScript file (there is no JSON or YAML equivalent) that exports a `config` object: 

```js
exports.config = {
    files: ["*.js"],
    ignores: ["*.test.js"],
    extends: [],
    globals: {},
    settings: {},
    processor: object,
    parser: object,
    parserOptions: {},
    ruledefs: {},
    rules: {}
};
```

The following keys are new to the `eslint.config.js` format:

* `files` - **Required.** Determines the glob file patterns that this configuration applies to.
* `ignores` - Determines the files that should not be linted using ESLint. This can be used in place of the `.eslintignore` file. The files specified by this array of glob patterns are subtracted from the files specified in `files`.
* `ruledefs` - Contains definitions for rules grouped by a specific name. This replaces the `plugins` key in `.eslintrc` files and the `--rulesdir` option.

The following keys are specified the same as in `.eslintrc` files:

* `globals`
* `settings`
* `rules`
* `parserOptions`

The following keys are specified differently than in `.eslintrc` files:

* `extends` - an object or array in `eslint.config.js` files (a string or string array in `.eslintrc`)
* `parser` - an object in `eslint.config.js` files (a string in `.eslintrc`)
* `processor` - an object in `eslint.config.js` files (a string in `.eslintrc`)

Each of these keys used to require one or more strings specifying module(s) to load in `.eslintrc`. In `eslint.config.js`, these are all objects, requiring users to manually specify the objects to use.

The following keys are invalid in `eslint.config.js`:

* `env` - responsibility of the user
* `overrides` - responsibility of the user
* `plugins` - replaced by `ruledefs`
* `root` - always considered `true`

Each of these keys represent different ways of augmenting how configuration is calculated and all of that responsibility now falls on the user.

#### Referencing Plugin Rules

The `plugins` key in `.eslintrc` was an array of strings indicating the plugins to load, allowing you to specify processors, rules, etc., by referencing the name of the plugin. It's no longer necessary to indicate the plugins to load because that is done directly in the `eslint.config.js` file. For example, consider this `.eslintrc` file:

```yaml
plugins:
  - react

rules:
  react/jsx-uses-react: error
```

This file tells ESLint to load `eslint-plugin-react` and then configure a rule from that plugin. The `react/` is automatically preprended to the rule by ESLint for easy reference.

In `eslint.config.js`, the same configuration is achieved using a `ruledefs` key:

```js
const reactPlugin = require("eslint-plugin-react");

exports.config = {
    files: ["*.js"],
    ruledefs: {
        react: reactPlugin.rules
    },
    rules: {
        "react/jsx-uses-react": "error"
    }
};
```

Here, it is the `ruledefs` that assigns the name `react` to the rules from `eslint-plugin-react`. The reference to `react/` in a rule will always look up that value in the `ruledefs` key.

#### Referencing Parsers and Processors

In `.eslintrc`, the `parser` and `processor` keys required strings to be specified, such as:

```yaml
plugins: ["markdown"]
parser: "babel-eslint"
processor: "markdown/markdown"
```

In `eslint.config.js`, you would need to pass the references directly, such as:

```js
exports.config = {
    files: ["*.js"],
    parser: require("babel-eslint"),
    processor: require("eslint-plugin-markdown").processors.markdown
};
```

In both cases, users now must pass a direct object reference. This has the benefit of using the builtin Node.js module resolution system or allowing users to specify their own. There is never a question of where the modules will be resolved from.

#### Applying an Environment

Unlike with `.eslintrc` files, there is no `env` key in `eslint.config.js`. Users can mimic the behavior of `env` by assigning directly to the `globals` key:

```js
const globals = require("globals");

exports.config = {
    files: ["*.js"],
    globals: {
        MyGlobal: true,
        ...globals.browser
    }
};
```

This effectively duplicates the use of `env: { browser: true }` in ESLint.

**Note:** This would allow us to stop shipping environments in ESLint. We could just tell people to use `globals` in their config and allow them to specify which version of `globals` they want to use.

#### Extending Another Config

Extending another config works the same as in `.eslintrc` except users pass entire config objects rather than the name of a config. That can easily be done in the following manner:

```js
exports.config = {
    files: ["*.js"],
    extends: require("eslint-config-standard"),
    rules: {
        semi: ["error", "always"]
    }
};
```

This config extends `eslint-config-standard` by assigning it to `extends`. You can also use an array for `extends` to extend from multiple configs:

```js
exports.config = {
    files: ["*.js"],
    extends: [
        require("eslint-config-standard"),
        require("@me/eslint-config")
    ],
    rules: {
        semi: ["error", "always"]
    }
};
```

#### Extending From `eslint:recommended` and `eslint:all`

Both `eslint:recommended` and `eslint:all` can be represented as strings in the `extends` key. For example:

```js
exports.config = {
    files: ["*.js"],
    extends: [
        "eslint:recommended",
        require("eslint-config-standard"),
        require("@me/eslint-config")
    ],
    rules: {
        semi: ["error", "always"]
    }
};
```

This config first extends `eslint:recommended` and then continues on to extend other configs.

#### Overriding Configuration Based on File Patterns

Whereas `.eslintrc` had an `overrides` key that made a hierarchical structure, the `eslint.config.js` file does not have any such hierarchy. Instead, users can return an array of configs that should be used. For example, consider this `.eslintrc` config:

```yaml
plugins: ["react"]
rules:
   react/jsx-uses-react: error
    semi: error

overrides:
    - files: "*.md"
      plugins: ["markdown"],
      processor: "markdown/markdown"
```

This can be written in `eslint.config.js` as an array of two configs:

```js
exports.config = [
    {
        files: "*.js",
        ruledefs: {
            react: require("eslint-plugin-react").rules,
        },
        rules: {
            "react/jsx-uses-react": "error",
            semi: "error"
        }
    },
    {
        files: "*.md",
        processor: require("eslint-plugin-markdown").processors.markdown
    }
];
```

When ESLint uses this config, it will check each `files` pattern to determine which configs apply. Any config with a `files` pattern matching the file to lint will be extracted and used (if multiple configs match, then those configs are merged to determine the final config to use). In this way, returning an array acts exactly the same as the array in `overrides`.

If a config in the config array does not contain `files` or `ignores`, then that config applies to all files. For example:

```js
exports.config = [
    {
        globals: {
            Foo: true
        }
    },
    {
        files: "*.js",
        ruledefs: {
            react: require("eslint-plugin-react").rules,
        },
        rules: {
            "react/jsx-uses-react": "error",
            semi: "error"
        }
    },
    {
        files: "*.md",
        processor: require("eslint-plugin-markdown").processors.markdown
    }
];
```

In this example, the first config in the array defines a global variable of `Foo`. That global variable is merged into the other two configs in the array automatically because there is no `files` or `ignores` specifying when it should be used.

Each item in a config array can be a config array. For example, this is a valid config array:

```js
exports.config = [
    [
        require("eslint-config-vue"),
        require("eslint-config-import")
    ]
    {
        files: "*.js",
        ruledefs: {
            react: require("eslint-plugin-react").rules,
        },
        rules: {
            "react/jsx-uses-react": "error",
            semi: "error"
        }
    },
    {
        files: "*.md",
        processor: require("eslint-plugin-markdown").processors.markdown
    }
];
```

A config array is always flattened before being evaluating, so either though this example is a two-dimensional config array, it will be evaluated as if it were a one-dimensional config array.

#### Replacing `.eslintignore`

Because there is only one `eslint.config.js` file to consider, ESLint doesn't have to first search directories to determine its location. That allows `eslint.config.js` to specify files to ignore directly instead of relying on `.eslintignore`. For backwards compatibility, users could create a config like this:

```js
const fs = require("fs");

exports.config = {
    files: ["*.js"],
    ignores: fs.readFileSync(".eslintignore", "utf8").split("\n")
};
```

### Replacing `--ext`

The `--ext` flag is currently used to pass in one or more file extensions that ESLint should search for when a directory without a glob pattern is passed on the command line, such as:

```bash
eslint src/ --ext .js,.jsx
```

This curently searches all subdirectories of `src/` for files with extensions matching `.js` or `.jsx`.

This proposal removes `--ext` by allowing the same information to be passed in a config. For example, the following config achieves the same result:

```js
const fs = require("fs");

exports.config = {
    files: ["*.js", "*.jsx"],
};
```

ESLint could then be run with this command:

```bash
eslint src/
```

When evaluating the `files` array in the config, ESLint will end up searching for `src/**/*.js` and `src/**/*.jsx`. (More information about file resolution is included later this proposal.)

### The `@eslint/config` Package

Because some of the operations ESLint currently do are quite complicated, this design includes a utility package called `@eslint/config` that users can use for common tasks. This package contains the following utilities:

1. `RuleLoader` - a class that aids in loading rules from a directory.

The `@eslint/config` package is intended to add back functionality that would be removed from ESLint with this design.

**Note:** It is not required to use this package with `eslint.config.js`; this is an optional tool that users may choose to use for convenience.

#### The `RuleLoader` Class

The purpose of the `RuleLoader` class is to aid in loading rules to replace `--rulesdir` functionality and has this form:

```js
class RuleLoader {

    // create a new instance
    constructor() {}

    // mimic `--rulesdir` loading
    loadRulesFromDirectory(directory) {}
}
```

In order to recreate the functionality of `--rulesdir`, a user would need to create a new entry in `ruledefs` and then specify the rules from a directory, such as:

```js
const { RuleLoader } = require("@eslint/config");
const ruleLoader = new RuleLoader();

exports.config = {
    ruledefs: {
        custom: ruleLoader.loadFromDirectory("./custom-rules")
    },
    rules: {
        "custom/my-rule": "error"
    }
};
```

The `RuleLoader#loadFromDirectory()` method returns an object where the keys are the rule IDs (based on the filenames found in the directory) and the values are the rule objects. Unlike today, rules loaded from a local directory must have a namespace just like plugin rules (`custom` in this example).

### Advanced Configs

Some users may need information from ESLint to determine the correct configuration to use. To allow for that, `exports.config` may also be a function that returns an object, such as:

```js
exports.config = (context) => {

    // do something

    return {
        files: ["*.js"],
        rules: {
            "semi": ["error", "always"]
        }
    };

};
```

The `context` object has the following members:

* `core` - information about the ESLint core that is using the config
    * `version` - the version of ESLint being used
    * `hasRule(ruleId)` - determine if the given rule is in the core
* `cwd` - the current working directory for ESLint (might be different than `process.cwd()` but always matches `CLIEngine.options.cwd`, see https://github.com/eslint/eslint/issues/11218)

This information allows users to make logical decisions about how the config should be constructed.

#### Checking for Rule Existence

One of the problems with shareable configs today is when a new rule is added to the ESLint core, shareable configs using that rule are not valid for older versions of ESLint (because ESLint validates that configured rules are present). With advanced configs, a shareable config could detect if a new rule is present before deciding to include it, for example:

```js
exports.config = (context) => {
    const myConfig = {
        files: ["*.js"],
        ruledefs: {
            custom: ruleLoader.loadFromDirectory("./custom-rules")
        },
        rules: {
            "custom/my-rule": "error"
        }
    };

    if (context.hasRule("some-new-rule")) {
        myConfig.rules["some-new-rule"] = ["error"];
    }

    return myConfig;
};
```

### Configuration Location Resolution

When ESLint is executed, the following steps are taken to find the `eslint.config.js` file to use:

1. If the `-c` flag is used then the specified configuration file is used. There is no further search performed.
1. Otherwise:
    a. Look for `eslint.config.js` in the current working directory. If found, stop searching and use that file.
    b. If not found, search up the directory hierarchy looking for `eslint.config.js`.
    c. If a `eslint.config.js` file is found at any point, stop searching and use that file.
    d. If `/` is reached without finding `eslint.config.js`, then stop searching and output a "no configuration found" error.

This approach will allow running ESLint from within a subdirectory of a project and get the same result as when ESLint is run from the project's root directory (the one where `eslint.config.js` is found).

Some of the key differences from the way ESLint's configuration resolution works today are:

1. There is no automatic search for `eslint.config.js` in the user's home directory. Users wanting this functionality can either pass a home directory file using `-c` or manually read in that file from their `eslint.config.js` file.
1. Once a `eslint.config.js` file is found, there is no more searching for any further config files.
1. There is no automatic merging of config files using either `extends` or `overrides`.
1. When `-c` is passed on the command line, there is no search performed.

### File Pattern Resolution

Because there are file patterns included in `eslint.config.js`, this requires a change to how ESLint decides which files to lint. The process for determining which files to lint is:

1. When a filename is passed directly (such as `eslint foo.js`):
    1. ESLint checks to see if there is one or more configs where the `files` pattern matches the file that was passed in and does not match the `ignores` pattern. The pattern is evaluated by prepending the directory in which `eslint.config.js` was found to each pattern in `files`. All configs that match `files` and not `ignores` are merged (with the last matching config taking precedence over others). If no config is found, then the file is ignored with an appropriate warning.
    1. If a matching config is found, then the `ignores` pattern is tested against the filename. If it's a match, then the file is ignored. Otherwise, the file is linted.
1. When a glob pattern is passed directly (such as `eslint src/*.js`):
    1. ESLint expands the glob pattern to get a list of files.
    1. Each file is checked individually as in step 1.
1. When a directory is passed directly (such as `eslint src`):
    1. The directory is converted into a glob pattern by appending the contents of the `files` array (such as `src` becomes `src/**/*.js`).
    1. The glob pattern is checked as in step 2.

### Rename `--no-eslintrc` to `--no-config-file` and `useEslintrc` to `useConfigFile`

Because the config filename has changed, it makes sense to change the command line `--no-eslintrc` flag to a more generic name, `--no-config-file` and change `CLIEngine`'s `useEslintrc` option to `useConfigFile`. In the short term, to avoid a breaking change, these pairs of names can be aliased to each other.

## Documentation

This will require extensive documentation changes and an introductory blog post.

At a minimum, these pages will have to be updated (and rewritten):

* https://eslint.org/docs/user-guide/getting-started#configuration
* https://eslint.org/docs/user-guide/configuring
* https://eslint.org/docs/developer-guide/working-with-plugins#configs-in-plugins
* https://eslint.org/docs/developer-guide/shareable-configs

## Drawbacks

As with any significant change, there are some significant drawbacks:

1. We'd need to do a phased rollout (see Backwards Compatibility Analysis) to minimize impact on users. That means maintaining two configuration systems for a while, increasing maintenance overhead and complexity of the application.
1. Getting everyone to convert to `eslint.config.js` format would be a significant stress on the community that could cause some resentment.
1. We can no longer enforce naming conventions for plugins and shareable configs. This may make it more difficult for users to find compatible npm packages.
1. Creating configuration files will be more complicated.
1. The `--print-config` option becomes less useful because we can't output objects in a meaningful way.
1. People depending on environments may find this change particularly painful.

## Backwards Compatibility Analysis

The intent of this proposal is to replace the current `.eslintrc` format, but can be implemented incrementally so as to cause as little disturbance to ESLint users as possible

In the first phase, I envision this:

1. `eslint.config.js` can be implemented alongside `.eslintrc`.
1. If `eslint.config.js` is found, then:
    1. All `.eslintrc` files are ignored.
    1. `.eslintignore` is ignored.
    1. `--ignore-file` is ignored.
    1. `--rulesdir` is ignored.
    1. `--env` is ignored.
    1. `--ext` is ignored.
    1. `--use-eslintrc` is ignored.
    1. `eslint-env` config comments are ignored.
1. If `eslint.config.js` is not found, then fall back to the current behavior.

This keeps the current behavior for the majority of users while allowing some users to test out the new functionality. Also, `-c` could not be used with `eslint.config.js` in this phase.

In the second phase (and in a major release), ESLint will emit deprecation warnings whenever the original functionality is used but will still honor them so long as `eslint.config.js` is not found.

In the third phase (and in another major release), `eslint.config.js` becomes the official way to configure ESLint. If no `eslint.config.js` file is found, ESLint will still search for a `.eslintrc` file, and if found, print an error message information the user that the configuration file format has changed.

So while this is intended to be a breaking change, it will be introduced over the course of three major releases in order to give users ample time to transition.

## Alternatives

While there are no alternatives that cover all of the functionality in this RFC, there are alternatives designed to address various parts.

* Both https://github.com/eslint/rfcs/pull/7 and https://github.com/eslint/rfcs/pull/5 specify an alternative method for resolving plugin locations in configs. This attempts to solve the problem with bundling plugins with configs (https://github.com/eslint/eslint/issues/3458). These proposals have the benefit of working within the current configuration system and requiring very few changes from users.
* It is possible to come up with a solution to https://github.com/eslint/eslint/issues/8813 that makes use of `extends`, though this would get a bit complicated if an extended config in an `overrides` section also has `overrides`.
* We could switch the current configuration system over so that all config files are considered to implicitly have `root:true`. This could dramatically simplify configuration searching and merging.

## Open Questions

1. Is `ruledefs` a clear enough key name?
1. Do we need a command line flag to opt-in to `eslint.config.js` instead of trying to do it alongside the existing configuration system?
1. Does the file pattern system actually remove the need for `--ext`?
1. How should `files` and `ignores` be merged when a shareable config has them? Should they be overwritten or merged?

## Frequently Asked Questions

### Can a config be an async function?

No. Right now it won't be possible to implement a config with an async function because the rest of ESLint is fully synchronous. Once we look at how to make ESLint more asynchronous, we can revisit and allow configs to be created with async functions.

### Why use `exports.config` instead of `module.exports`?

Using an exported key gives us more flexibility for the future if we decide that config files should be able to output more than one thing. For example, I've been thinking of a `--config-key` option that would allow users to specify which exported key should be used as their config. Users could then export multiple different keys (`config1`, `config2`, etc.) and easily switch between configs on the command line. That option is not part of this proposal because it isn't solving an existing problem and I'd rather focus on existing problems first (this proposal is already big enough).

### How does this affect configuration via `package.json`?

The `eslintConfig` and `eslintIgnore` keys in `package.json` will not be honored when `eslint.config.js` is found. Users could still pull that information into their `eslint.config.js` file manually if they want to.

### Do shareable configs still export an object on `module.exports`?

That is completely up to the shareable config. For simplicity sake, I think we still want to encourage people to do so. However, there is no longer and formal contract between ESLint and shareable configs, so developers could potentially export configs from any npm package using any exported key. They would just need to inform users about how to extend their config properly.

### Is there a way to allow plugins to specify the namespace of their rules?

Maybe, but even if we could do that, the namespaces aren't guaranteed to be unique.

The reason we could enforce naming of both plugin packages and rule namespaces is because ESLint controlled how the plugins were loaded: users passed ESLint a string and then ESLint could both inspect the string to pull out details it needed (the plugin name without the `eslint-plugin-` prefix) and then modify the rule names to be prepended with the plugin name.

In this design, the user is responsible for loading npm packages. Because we are only ever passed an object, we no longer have access to the plugin name.

An earlier iteration of this design had plugins specified like this:

```js
exports.config = {
    plugins: [
        require("eslint-plugin-react"),
        require("eslint-plugin-mardkown")
    ]
};
```

This didn't work because ESLint didn't know how to name the plugin rules (we weren't getting the plugin name, just the object). So the next iteration went to this:

```js
exports.config = {
    plugins: {
        react: require("eslint-plugin-react"),
        markdown: require("eslint-plugin-mardkown")
    }
};
```

This iteration gave the plugin rules names, but then I realized the only things that need names in plugins with this design are rules. Users can get processors, parsers, etc., directly without ESLint being involved in picking them out of plugins. So I renamed `plugins` to `ruledefs` to make this explicit:

```js
exports.config = {
    ruledefs: {
        react: require("eslint-plugin-react").rules,
        markdown: require("eslint-plugin-mardkown").rules
    }
};
```

This is the iteration I first submitted.


## Related Discussions

* https://github.com/eslint/rfcs/pull/7
* https://github.com/eslint/rfcs/pull/5
* https://github.com/eslint/eslint/issues/3458
* https://github.com/eslint/eslint/issues/6732
* https://github.com/eslint/eslint/issues/8813
* https://github.com/eslint/eslint/issues/9192
* https://github.com/eslint/eslint/issues/9897
* https://github.com/eslint/eslint/issues/10125
* https://github.com/eslint/eslint/issues/10643
* https://github.com/eslint/eslint/issues/10891
* https://github.com/eslint/eslint/issues/11223
