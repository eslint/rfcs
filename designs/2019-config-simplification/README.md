- Start Date: 2019-01-20
- RFC PR: https://github.com/eslint/rfcs/pull/9
- Authors: Nicholas C. Zakas (@nzakas)
- repo: eslint/eslint
- Contributors: Teddy Katz (@not-an-aardvark), Toru Nagashima (@mysticatea), Kai Cataldo (@kaicataldo), Shahar Dawn (@mightyiam)

# Config File Simplification

## Summary

This proposal provides a way to simplify configuration of ESLint through a new configuration file format. The configuration file format is written in JavaScript and removes several existing configuration keys in favor of allowing the user to manually create them.

## Motivation

The ESLint configuration file format (`.eslintrc`) hasn't changed much since ESLint was first released. As we have continued to add new features, the configuration file format has continued to be extended and evolve into a state where it is the most complicated part of ESLint. The `.eslintrc` way of doing things means a variety of different merging strategies (cascading, extending, overriding), loading strategies (for plugins, processors, parsers), and naming conventions (`eslint-config-*`, `eslint-plugin-*`).

Although `.eslintrc` has proven to be a very robust format, it is also limited and unable to easily accommodate feature requests that the community favors, such as:

1. [bundling plugins with configs](https://github.com/eslint/eslint/issues/3458)
1. [Customize merging of config options](https://github.com/eslint/eslint/issues/9192)
1. [specifying ignore patterns in the config](https://github.com/eslint/eslint/issues/10891) (implemented in `.eslintrc` after this RFC was written)
1. [specifying `--ext` information in the config](https://github.com/eslint/eslint/issues/11223) (implemented in `.eslintrc` after this RFC was written)
1. [using `extends` in `overrides`](https://github.com/eslint/eslint/issues/8813) (implemented in `.eslintrc` after this RFC was written)

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
1. An `@eslint/eslintrc` utility is provided to aid with backwards compatibility
1. Remove the concept of environments (`env`) (including `--env`, `/*eslint-env*/`, and `env` in config files)
1. Remove `--rulesdir`
1. Remove `--ext`
1. Remove `--resolve-plugins-relative-to`
1. Replace `context.parserPath` with `context.parser` (the path to the parser is no longer valid)
1. Allow plugins to specify a `parsers` key so parsers can be referenced the same as rules and processors in plugins.

### The `eslint.config.js` File

The `eslint.config.js` file is a JavaScript file (there is no JSON or YAML equivalent) that exports an object: 

```js
module.exports = {
    name: "name",
    files: ["*.js"],
    ignores: ["*.test.js"],
    settings: {},
    languageOptions: {
        ecmaVersion: 2020,
        sourceType: "module",
        globals: {},
        parser: object || "string",
        parserOptions: {},
    }
    linterOptions: {
        reportUnusedDisableDirectives: "string"
    },
    processor: object || "string",
    plugins: {}
    rules: {}
};
```

The following keys are new to the `eslint.config.js` format:

* `name` - Specifies the name of the config object. This is helpful for printing out debugging information and, while not required, is recommended for that reason.
* `files` - Determines the glob file patterns that this configuration applies to. These patterns can be negated by prefixing them with `!`, which effectively mimics the behavior of `excludedFiles` in `.eslintrc`.
* `ignores` - Determines the files that should not be linted using ESLint. The files specified by this array of glob patterns are subtracted from the files specified in `files`. If there is no `files` key, then `ignores` acts the same as `ignorePatterns` in `.eslintrc` files; if there is a `files` key then `ignores` acts like `excludedFiles` in `.eslintrc`.

The following keys are specified the same as in `.eslintrc` files:

* `settings`
* `rules`

The following keys are specified differently than in `.eslintrc` files:

* `plugins` - an object mapping plugin names to implementations. This replaces the `plugins` key in `.eslintrc` files and the `--rulesdir` option.
* `processor` - an object or string in `eslint.config.js` files (a string in `.eslintrc`)
* `languageOptions` - top-level grouping for all options that affect how JavaScript is interpreted by ESLint.
    * `ecmaVersion` - sets the JavaScript version ESLint should use for these files. This value is copied into `parserOptions` if not already present.
    * `sourceType` - sets the source type of the JavaScript code to parse. One of `module`, `script`, or `commonjs`.
    * `parser` - an object or string in `eslint.config.js` files (a string in `.eslintrc`)
    * `parserOptions` - an object specifying any additional parameters to be passed to the parser.
    * `globals` - any additional global variables to add.
* `linterOptions` - an object for linter-specific settings
    * `reportUnusedDisableDirectives` - new location for the same option name.

Each of these keys used to require one or more strings specifying module(s) to load in `.eslintrc`. In `eslint.config.js`, these are all objects or strings (referencing an object in a plugin), requiring users to manually specify the objects to use.

The following keys are invalid in `eslint.config.js`:

* `extends` - replaced by config arrays
* `env` - responsibility of the user
* `overrides` - responsibility of the user
* `root` - always considered `true`

Each of these keys represent different ways of augmenting how configuration is calculated and all of that responsibility now falls on the user.

#### Extending Another Config

Extending another config is accomplished by returning an array as the value of `module.exports`. Configs that come later in the array are merged with configs that come earlier in the array. For example:

```js
module.exports = [
    require("eslint-config-standard"),
    {
        files: ["*.js"],
        rules: {
            semi: ["error", "always"]
        }
    }
];
```

This config extends `eslint-config-standard` because that package is included first in the array. You can add multiple configs into the array to extend from multiple configs, such as:

```js
module.exports = [
    require("eslint-config-standard"),
    require("@me/eslint-config"),
    {
        files: ["*.js"],
        rules: {
            semi: ["error", "always"]
        }
    }
];
```

Each item in a config array can be a config array. For example, this is a valid config array and equivalent to the previous example:

```js
module.exports = [
    [
        require("eslint-config-standard"),
        require("@me/eslint-config")
    ],
    {
        files: ["*.js"],
        rules: {
            semi: ["error", "always"]
        }
    }
];
```

A config array is always flattened before being evaluated, so even though this example is a two-dimensional config array, it will be evaluated as if it were a one-dimensional config array.

When using a config array, only one config object must have a `files` key (config arrays where no objects contain `files` will result in an error). If a config in the config array does not contain `files` or `ignores`, then that config is merged into every config with a `files` pattern. For example:

```js
module.exports = [
    {
        languageOptions: {
            globals: {
                Foo: true
            }
        }
    },
    {
        files: ["*.js"],
        rules: {
            semi: ["error", "always"]
        }
    },
    {
        files: ["*.mjs"],
        rules: {
            semi: ["error", "never"]
        }
    }
];
```

In this example, the first config in the array defines a global variable of `Foo`. That global variable is merged into the other two configs in the array automatically because there is no `files` or `ignores` specifying when it should be used. The first config matches zero files on its own and would be invalid if it was the only config in the config array.

#### Extending From `eslint:recommended` and `eslint:all`

Both `eslint:recommended` and `eslint:all` can be represented as strings in a config array. For example:

```js
module.exports = [
    "eslint:recommended",
    require("eslint-config-standard"),
    require("@me/eslint-config"),
    {
        files: ["*.js"],
        rules: {
            semi: ["error", "always"]
        }
    }
];
```

This config first extends `eslint:recommended` and then continues on to extend other configs.

#### Setting the Name of Shareable Configs

For shareable configs, specifying a `name` property for each config they export helps ESLint to output more useful error messages if there is a problem. The `name` property is a string that will be used to identify configs to help users resolve problems. For example, if you are creating `eslint-config-example`, then you can specify a `name` property to reflect that:

```js
module.exports = {
    name: "eslint-config-example",

    // other info here
};
```

It's recommended that the shareable config provide a unique name for each config that is exported.

#### Disambiguating Shareable Configs With Common Dependencies

Today, shareable configs that depend on plugin rules must specify the plugin as a peer dependency and then either provide a script to install those dependencies or ask the user to install them manually.

With this design, shareable configs can specify plugins as direct dependencies that will be automatically installed with the shareable config, improving the user experience of complex shareable configs. This means it's possible for multiple shareable configs to depend on the same plugin and, in theory, depend on different versions of the same plugin. In general, npm will handle this directly, installing the correct plugin version at the correct level for each shareable config to `require()`. For example, suppose there are two shareable configs, `eslint-config-a` and `eslint-config-b` that both rely on the `eslint-plugin-example` plugin, but the former relies on 1.0.0 and the latter relies on 2.0.0. npm will install those plugins like this:

```
your-project
├── eslint.config.js
└── node_modules
  ├── eslint
  ├── eslint-config-a
  | └── node_modules
  |   └── eslint-plugin-example@1.0.0
  └── eslint-config-b
    └── node_modules
      └── eslint-plugin-example@2.0.0
```

The problem comes when the shareable configs try to use the default namespace of `eslint-plugin-example` for its rules, such as:

```js
const example = require("eslint-plugin-example");

module.exports = {
    plugins: {
        example
    }
};
```

If both shareable configs do this, and the user tries to use both shareable configs, an error will be thrown when the configs are normalized because the plugin namespace `example` can only be assigned once.

To work around this problem, the end user can create a separate namespace for the same plugin so that it doesn't conflict with an existing plugin namespace from a shareable config. For example, suppose you want to use `eslint-config-first`, and that has an `example` plugin namespace defined. You'd also like to use `eslint-config-second`, which also has an `example` plugin namespace defined. Trying to use both shareable configs will throw an error because a plugin namespace cannot be defined twice. You can still use both shareable configs by creating a new config from `eslint-config-second` that uses a different namespace. For example:

```js
// get the config you want to extend
const configToExtend = require("eslint-config-second");

// create a new copy (NOTE: probably best to do this with @eslint/config somehow)
const compatConfig = Object.create(configToExtend);
compatConfig.plugins["compat::example"] = require("eslint-plugin-example");
delete compatConfig.plugins.example;

// include in config
module.exports = [

    require("eslint-config-first");
    compatConfig,
    {
        // overrides here
    }
];
```

#### Referencing Plugin Rules

The `plugins` key in `.eslintrc` was an array of strings indicating the plugins to load, allowing you to specify processors, rules, etc., by referencing the name of the plugin. It's no longer necessary to indicate the plugins to load because that is done directly in the `eslint.config.js` file. For example, consider this `.eslintrc` file:

```yaml
plugins:
  - react

rules:
  react/jsx-uses-react: error
```

This file tells ESLint to load `eslint-plugin-react` and then configure a rule from that plugin. The `react/` is automatically preprended to the rule by ESLint for easy reference.

In `eslint.config.js`, the same configuration is achieved using a `plugins` key:

```js
const react = require("eslint-plugin-react");

module.exports = {
    files: ["*.js"],
    plugins: {
        react
    },
    rules: {
        "react/jsx-uses-react": "error"
    }
};
```

Here, it is the `plugins` that assigns the name `react` to the rules from `eslint-plugin-react`. The reference to `react/` in a rule will always look up that value in the `plugins` key.

**Note:** If a config is merged with another config that already has the same `plugins` namespace defined and the namespace doesn't refer to the same rules object, then an error is thrown. In this case, if a config already has a `react` namespace, then attempting to combine with another config that has a `react` namespace that contains different rules will throw an error. This is to ensure the meaning of `namespace/rule` remains consistent.

#### Plugins Specifying Their Own Namespaces

Rules imported from a plugin must be assigned a namespace using `plugins`, which puts the responsibility for that namespace on the config file user. Plugins can define their own namespace for rules in two ways. (Note that plugins will not be required to define their own namespaces.)

First, a plugin can export a recommended configuration to place in a config array. For example, a plugin called `eslint-plugin-example`, might define a config like this:

```js
module.exports = {
    configs: {
        recommended: {
            plugins: {
                example: {
                    rules: {
                        rule1: require("./rules/rule1")
                    }
                }
            }
        }
    }
};
```

Then, inside of a user config, the plugin's recommended config can be loaded:

```js
module.exports = [
    require("eslint-plugin-example").configs.recommended,
    {
        rules: {
            "example/rule1": "error"
        }
    }
];
```

The user config in this example now inherits the `plugins` from the plugin's recommended config, automatically adding in the rules with their preferred namespace. (Note that the user config can't have another `plugins` namespace called `example` without an error being thrown.)

The second way for plugins to specify their preferred namespace is to export a `plugin` key directly that users can include their own config. This is what it would look like in the plugin:

```js
exports.plugin = {
    example: {
        rules: {
            rule1: require("./rules/rule1")
        }
    }
};
```

Then, inside of a user config, the plugin's `plugin` can be included directly`:

```js
module.exports = {
    plugins: {
        ...require("eslint-plugin-example").plugin
    },
    rules: {
        "example/rule1": "error"
    }
};
```

This example imports the `plugin` from a plugin directly into the same section in the user config.

#### Referencing Parsers and Processors

In `.eslintrc`, the `parser` and `processor` keys required strings to be specified, such as:

```yaml
plugins: ["markdown"]
parser: "babel-eslint"
processor: "markdown/markdown"
```

In `eslint.config.js`, there are two options. First, you can pass references directly into these keys:

```js
module.exports = {
    files: ["*.js"],
    languageOptions: {
        parser: require("babel-eslint"),
    },
    processor: require("eslint-plugin-markdown").processors.markdown
};
```

Second, you can use a string to specify an object to load from a plugin, such as:

```js
module.exports = {
    plugins: {
        markdown: require("eslint-plugin-markdown"),
        babel: require("eslint-plugin-babel")
    },
    files: ["*.js"],
    languageOptions: {
        parser: "babel/eslint-parser",
    },
    processor: "markdown/markdown"
};
```

In this example, `"babel/eslint-parser"` loads the parser defined in the `eslint-plugin-babel` plugin and `"markdown/markdown"` loads the processor from the `eslint-plugin-markdown` plugin. Note that the behavior for `parser` is different than with `.eslintrc` in that the string **must** represent a parser defined in a plugin.

The benefit to this approach of specifying parsers and processors is that it uses the builtin Node.js module resolution system or allows users to specify their own. There is never a question of where the modules will be resolved from.

**Note:** This example requires that `eslint-plugin-babel` publishes a `parsers` property, such as:

```js
module.exports = {
    parsers: {
        "eslint-parser": require("./some-file.js")
    }
}
```

This is a new feature of plugins introduced with this RFC.

#### Applying an Environment

Unlike with `.eslintrc` files, there is no `env` key in `eslint.config.js`. For different ECMA versions, ESLint will automatically add in the required globals. For example:

```js
const globals = require("globals");

module.exports = {
    files: ["*.js"],
    languageOptions: {
        ecmaVersion: 2020
    }
};
```

Because the `languageOptions.ecmaVersion` property is now a linter-level option instead of a parser-level option, ESLint will automatically add in all of the globals for ES2020 without requiring the user to do anything else.

Similarly, when `sourceType` is `"commonjs"`, ESLint will automatically add the `require`, `exports`, and `module` global variables (as well as set `parserOptions.ecmaFeatures.globalReturn` to `true`). In this case, ESLint will pass a `sourceType` of `"script"` as part of `parserOptions` because parsers don't support `"commonjs"` for `sourceType`.

For other globals, ssers can mimic the behavior of `env` by assigning directly to the `globals` key:

```js
const globals = require("globals");

module.exports = {
    files: ["*.js"],
    languageOptions: {
        globals: {
            MyGlobal: true,
            ...globals.browser
        }
    }
};
```

This effectively duplicates the use of `env: { browser: true }` in ESLint.

**Note:** This would allow us to stop shipping environments in ESLint. We could just tell people to use `globals` in their config and allow them to specify which version of `globals` they want to use.

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
module.exports = [
    {
        files: "*.js",
        plugins: {
            react: require("eslint-plugin-react"),
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

#### Using AND patterns for files

If any entry in the `files` key is an array, then all of the patterns must match in order for a filename to be considered a match. For example:

```js
module.exports = [
    {
        files: [ ["*.test.*", "*.js"] ],
        rules: {
            semi: ["error", "always"]
        }
    }
];
```

Here, the `files` key specifies two glob patterns. The filename `foo.test.js` would match because it matches both patterns whereas the filename `foo.js` would not match because it only matches one of the glob patterns.

**Note:** This feature is primarily intended for backwards compatibility with eslintrc's ability to specify `extends` in an `overrides` block.

#### Ignoring files

With `eslint.config.js`, there are three ways that files and directories can be ignored:

1. **Defaults** - by default, ESLint will ignore `node_modules` and `.git` directories only. This is different from the current behavior where ESLint ignores `node_modules` and all files/directories beginning with a dot (`.`).
2. **.eslintignore** - the regular ESLint ignore file.
3. **eslint.config.js** - patterns specified in `ignores` keys when `files` is not specified. (See details below.) 

Anytime `ignores` appears in a config object without `files`, then the `ignores` patterns acts like the `ignorePatterns` key in `.eslintrc` in that the patterns are excluded from all searches before any other matching is done. For example:

```js
module.exports = [{
    ignores: "web_modules"
}];
```

Here, the directory `web_modules` will be ignored as if it were defined in an `.eslintignore` file. The `web_modules` directory will be excluded from the glob pattern used to determine which files ESLint will run against.

The `--no-ignore` flag will disable `eslint.config.js` and `.eslintignore` ignore patterns while leaving the default ignore patterns in place.

### Replacing `--ext`

The `--ext` flag is currently used to pass in one or more file extensions that ESLint should search for when a directory without a glob pattern is passed on the command line, such as:

```bash
eslint src/ --ext .js,.jsx
```

This curently searches all subdirectories of `src/` for files with extensions matching `.js` or `.jsx`.

This proposal removes `--ext` by allowing the same information to be passed in a config. For example, the following config achieves the same result:

```js
const fs = require("fs");

module.exports = {
    files: ["*.js", "*.jsx"],
};
```

ESLint could then be run with this command:

```bash
eslint src/
```

When evaluating the `files` array in the config, ESLint will end up searching for `src/**/*.js` and `src/**/*.jsx`. (More information about file resolution is included later this proposal.)

Additionally, ESLint can be run without specifying anything on the command line, relying just on what's in `eslint.config.js` to determine what to lint:

```bash
eslint
```

This will go into the `eslint.config.js` and use all of the `files` glob patterns to determine which files to lint.

### Replacing `--rulesdir`

In order to recreate the functionality of `--rulesdir`, a user would need to create a new entry in `plugins` and then specify the rules from a directory. This can be accomplished using the [`requireindex`](https://npmjs.com/package/requireindex) npm package:

```js
const requireIndex = require("requireindex");

module.exports = {
    plugins: {
        custom: {
            rules: requireIndex("./custom-rules")
        }
    },
    rules: {
        "custom/my-rule": "error"
    }
};
```

The `requireIndex()` method returns an object where the keys are the rule IDs (based on the filenames found in the directory) and the values are the rule objects. Unlike today, rules loaded from a local directory must have a namespace just like plugin rules (`custom` in this example).

### Function Configs

Some users may need information from ESLint to determine the correct configuration to use. To allow for that, `module.exports` may also be a function that returns an object, such as:

```js
module.exports = (context) => {

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

* `name` - the name of the application being used
* `version` - the version of ESLint being used
* `cwd` - the current working directory for ESLint (might be different than `process.cwd()` but always matches `CLIEngine.options.cwd`, see https://github.com/eslint/eslint/issues/11218)

This information allows users to make logical decisions about how the config should be constructed.

A configuration function may return an object or an array of objects. An error is thrown if any other type of value is returned.

#### Including Function Configs in an Array

A function config can be used anywhere a config object or a config array is valid. That means you can insert a function config as a config array member:

```js
module.exports = [
    (context) => someObject,
    require("eslint-config-myconfig")
];
```

Each function config in an array will be executed with a `context` object when ESLint evaluates the configuration file. This also means that shareable configs can export a function instead of an object or array.

**Note:** If a function config inside of a config array happens to return an array, then those config array items are flattened as with any array-in-array situation.

### The `@eslint/eslintrc` Utility

To allow for backwards compatibility with existing configs and plugins, an `@eslint/eslintrc` utility is provided. The package exports the following classes:

* `ESLintRCCompat` - a class to help convert `.eslintrc`-style configs into the correct format.


```js
class ESLintRCCompat {
    
    constructor(baseDir) {}
    
    config(configObjct) {}
    plugins(...pluginNames) {}
    extends(...sharedConfigNames) {}
    env(envObject) {}

}
```

#### Importing existing configs

The `ESLintRCCompat#extends()` function allows users to specify an existing `.eslintrc` config location in the same format that used in the `.eslintrc` `extends` key. Users can pass in a filename, a shareable config name, or a plugin config name and have it converted automatically into the correct format. For example:

```js
const { ESLintRCCompat } = require("@eslint/eslintrc");

const eslintrc = new ESLintRCCompat(__dirname);

module.exports = [
    "eslint:recommended",
  
    // load a file
    eslintrc.extends("./.eslintrc.yml"),

    // load eslint-config-standard
    eslintrc.extends("standard"),

    // load eslint-plugin-vue/recommended
    eslintrc.extends("plugin:vue/recommended"),

    // or multiple at once
    eslintrc.extends("./.eslintrc.yml", "standard", "plugin:vue/recommended")

];
```

#### Translating config objects

The `ESLintRCCompat#config()` methods allows users to pass in a `.eslintrc`-style config and get back a config object that works with `eslint.config.js`. For example:

```js
const { ESLintRCCompat } = require("@eslint/eslintrc");

const eslintrc = new ESLintRCCompat(__dirname);

module.exports = [
    "eslint:recommended",
  
    eslintrc.config({
        env: {
            node: true
        },
        root: true
    });
];
```

#### Including plugins

The `ESLintRCCompat#plugins()` method allows users to automatically load a plugin's rules and processors without separately assigning a namespace. For example:

```js
const { ESLintRCCompat } = require("@eslint/eslintrc");

const eslintrc = new ESLintRCCompat(__dirname);

module.exports = [
    "eslint:recommended",

    // add in eslint-plugin-vue and eslint-plugin-example
    eslintrc.plugins("vue", "example")
];
```

This example includes both `eslint-plugin-vue` and `eslint-plugin-example` so that all of the rules are available with the correct namespace and processors are automatically hooked up to the correct `files` pattern.

#### Applying environments

The `ESLintRCCompat#env()` method allows users to specify an `env` settings as they would in an `.eslintrc`-style config and have globals automatically added. For example:

```js
const { ESLintRCCompat } = require("@eslint/eslintrc");

const eslintrc = new ESLintRCCompat(__dirname);

module.exports = [
    "eslint:recommended",
  
    // load node environment
    eslintrc.env({
        node: true
    })
];
```

### Configuration Location Resolution

When ESLint is executed, the following steps are taken to find the `eslint.config.js` file to use:

1. If the `-c` flag is used then the specified configuration file is used. There is no further search performed.
1. Otherwise:
    1. Look for `eslint.config.js` in the current working directory. If found, stop searching and use that file.
    1. If not found, search up the directory hierarchy looking for `eslint.config.js`.
    1. If a `eslint.config.js` file is found at any point, stop searching and use that file.
    1. If `/` is reached without finding `eslint.config.js`, then stop searching and output a "no configuration found" error.

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
    1. The directory is converted into a glob pattern by appending `**/*` to the directory (such as `src` becomes `src/**/*`).
    1. The glob pattern is checked as in step 2.
1. When a relative directory is passed directly (such as `eslint .`):
    1. The relative directory pattern is resolved to a full directory name.
    1. The glob pattern is checked as in step 3.

**Note:** ESLint will continue to ignore `node_modules` by default.

### Rename `--no-eslintrc` to `--no-config-file` and `useEslintrc` to `useConfigFile`

Because the config filename has changed, it makes sense to change the command line `--no-eslintrc` flag to a more generic name, `--no-config-file` and change `CLIEngine`'s `useEslintrc` option to `useConfigFile`. In the short term, to avoid a breaking change, these pairs of names can be aliased to each other.

### Implementation Details

The implementation of this feature requires the following changes:

1. Create a new `ESLintConfigArray` class to manage configs.
1. Create a `--no-config-file` CLI option and alias it to `--no-eslintrc` for backwards compatibility.
1. Create a `useConfigFile` option for `CLIEngine`. Alias `useEslintrc` to this option for backwards compatibility.
1. In `CLIEngine#executeOnFiles()`:
    1. Check for existence of `eslint.config.js`, and if found, opt-in to new behavior.
    1. Create a `ESLintConfigArray` to hold the configuration information and to determine which files to lint (in conjunction with already-existing `globUtils`)
    1. Rename the private functions `processText()` and `processFiles()` to `legacyProcessText()` and `legacyProcessFiles()`; create new versions with the new functionality named `processText()` and `processFiles()`. Use the appropriate functions based on whether or not the user has opted-in.
    1. Update `Linter#verify()` to check for objects on keys that now support objects instead of strings (like `parser`) and add a `disableEnv` property to the options to indicate that environments should not be honored.
1. At a later point, we will be able to remove a lot of the existing configuration utilities.

#### The `ESLintConfigArray` Class

The `ESLintConfigArray` class is the primary new class for handling the configuration change defined in this proposal.

```js
class ESLintConfigArray extends Array {

    // normalize the current ConfigArray
    async normalize(context) {}

    // get a single config for the given filename
    getConfig(filename) {}

    // get the file patterns to search for
    get files() {}

    // get the "ignore file" values
    get ignores() {}
}
```

In this class, "normalize" means that all functions are called and replaced with their results, the array has been flattened, and configs without `files` keys have been merged into configs that do have `files` keys for easier calculation.

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

The intent of this proposal is to replace the current `.eslintrc` format, but can be implemented incrementally so as to cause as little disturbance to ESLint users as possible.

In the first phase, I envision this:

1. Extract all current `.eslintrc` functionality into `@eslint/eslintrc` package. Change ESLint to depend on `@eslint/eslintrc` package.
1. `eslint.config.js` can be implemented alongside `.eslintrc`.
1. If `eslint.config.js` is found, then:
    1. All `.eslintrc` files are ignored.
    1. `--rulesdir` is ignored.
    1. `--env` is ignored.
    1. `--ext` is ignored.
    1. `eslint-env` config comments are ignored.
1. If `eslint.config.js` is not found, then fall back to the current behavior.
1. Switch ESLint itself to use `eslint.config.js` as a way to test and ensure compatibility with existing shareable configs in `.eslintrc` format.

This keeps the current behavior for the majority of users while allowing some users to test out the new functionality. Also, `-c` could not be used with `eslint.config.js` in this phase.

In the second phase (and in a major release), ESLint will emit deprecation warnings whenever the original functionality is used but will still honor them so long as `eslint.config.js` is not found. In this phase, we will work with several high-profile plugins and shareable configs to convert their packages into the new format. We will use this to find the remaining compatibility issues.

In the third phase (and in another major release), `eslint.config.js` becomes the official way to configure ESLint. If no `eslint.config.js` file is found, ESLint will still search for a `.eslintrc` file, and if found, print an error message information the user that the configuration file format has changed.

So while this is intended to be a breaking change, it will be introduced over the course of three major releases in order to give users ample time to transition.

## Alternatives

While there are no alternatives that cover all of the functionality in this RFC, there are alternatives designed to address various parts.

* Both https://github.com/eslint/rfcs/pull/7 and https://github.com/eslint/rfcs/pull/5 specify an alternative method for resolving plugin locations in configs. This attempts to solve the problem with bundling plugins with configs (https://github.com/eslint/eslint/issues/3458). These proposals have the benefit of working within the current configuration system and requiring very few changes from users.
* It is possible to come up with a solution to https://github.com/eslint/eslint/issues/8813 that makes use of `extends`, though this would get a bit complicated if an extended config in an `overrides` section also has `overrides`.
* We could switch the current configuration system over so that all config files are considered to implicitly have `root:true`. This could dramatically simplify configuration searching and merging.

## Open Questions

1. Do we need a command line flag to opt-in to `eslint.config.js` instead of trying to do it alongside the existing configuration system?

## Frequently Asked Questions

### Will the inability to serialize these configs affect any features?

There are three features that non-serializable configs affect directly:

1. **Parallel linting** - we can no longer pass the configs between the main thread and a worker thread. However, we can work around this by passing the config filename to each worker thread to load independently. This may have some overhead but given the speed gains of parallel linting they should be negligible.
1. **Caching** - we can no longer use the serialized version of the configs as part of the cache key. We can instead read the config file in as a string and use that as part of the cache key.
1. **--print-config** - printing the config to the console will be a bit more confusing because there will be non-serializable objects included. We can look into ways of addressing this issue as we receive feedback on use cases.

### Can a config be an async function?

Yes. We will be able to implement async config functions once the new async ESLint API has been implemented.

### How does this affect configuration via `package.json`?

The `eslintConfig` and `eslintIgnore` keys in `package.json` will not be honored when `eslint.config.js` is found. Users could still pull that information into their `eslint.config.js` file manually if they want to.

### Do shareable configs still export an object on `module.exports`?

That is completely up to the shareable config. For simplicity sake, I think we still want to encourage people to do so. However, there is no longer a formal contract between ESLint and shareable configs, so developers could potentially export configs from any npm package using any exported key. They would just need to inform users about how to extend their config properly.

### Why are "eslint:recommended" and "eslint:all" strings instead of objects?

In order to use objects for these two configs, we'd need to somehow pass those objects to the config file. That would mean either exposing something on the ESLint package itself (i.e., `require("eslint").configs.recommended`) or publishing a separate package to do the same (i.e., `require("@eslint/configs).recommended`). 

In the first case, you'd end up with a situation where the config needs to specify a particular version of ESLint as its dependency, and that could mean a config could force an ESLint upgrade unnecessarily (especially when shareable configs depend on `"eslint:recommended"`).

In the second case, we'd be stuck trying to keep the core ESLint configs in sync with another package, which is maintenance overhead.

By using strings as placeholders, we allow the core to fill in the values for those configs without adding more restrictions onto the config files.

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
* https://github.com/eslint/rfcs/pull/55
