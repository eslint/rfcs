- Start Date: 2018-11-20
- RFC PR: https://github.com/eslint/rfcs/pull/6
- Authors: Nicholas C. Zakas (@nzakas)

# Programmatic Style Editors

## Summary

A way to extend the capabilities of ESLint's `--fix` command to enable applying  programmatic style edits instead of (or in addition to) using stylistic rules. Developers can create automatically applied style guides by writing JavaScript code rather than configuring individual rules.

## Motivation

The way ESLint has traditionally helped developers enforce style guides is through the implementation of individual rules. Each rule then needs to be manually configured by someone and the ESLint team ends up maintaining the rule. Because people enjoy using shareable configs, developers frequently ask for new style rules and new options to existing style rules be implemented in the core. This has lead to an explosion of stylistic rules that are getting more difficult to maintain and get to work seamlessly with one another (i.e., ensuring rules don't conflict with one another or overlap).

Tools like [Prettier](https://prettier.io/) have taken a different approach, where there is a small set of configuration options available. Instead of warning on each small style disparity, it simply formats everything according to the configuration options. This opinionated approach is simpler to maintain but has the downside of being too strict for some developers who prefer more fine-grained control.

Prettier has also shown that there is a desire to not fill up end-user screens with multiple error messages advising developers to add a space here or remove a space there; they'd much rather just have one command to fix the style of their source code.

Additionally, a growing number of developers use ESLint together with Prettier to get both the error-checking capability of ESLint with the source code formatting of Prettier. While there are some approaches to making these two tools work together, any quick search of Twitter will find new developers tend to struggle to get these two tools to work together.

All of this is to say that the current way that ESLint deals with style guides, including its interaction with Prettier, leaves a lot of room for improvement. To that end, this programmatic style editor feature is intended to:

1. Allow arbitrary modification of source code within the context of autofixing.
1. Enable the use of third-party tools to do this modification easily.
1. Reduce developers' dependence on ESLint core rules for style guide enforcement and therefore reduce the maintenance burden on the ESLint team.

## Detailed Design

At a high-level, this design entails:

1. Enable plugins to export style editors
1. Allow third-party tools to be incorporated into style editors
1. Automatically apply style editors with using `--fix`
1. Allow turning on/off style editors using `--fix-type`
1. Ensure style editors are composable

Definitions:

* **Style Edit** - an individual code modification written in JavaScript.
* **Style Editor** - a collection of style edits to apply in order. Style editors are applied through configuration.

### Style Editors in Plugins

A plugin can contain one or more style editors, exported on a `styles` object in a manner similar to `rules` and `processors`, for example:

```js
// eslint-plugin-mystyles
const es5StyleEditor = require("./styles/es5");

module.exports = {
    styles: {
        es5: es5StyleEditor
    }
};
```

In this code, the plugin `eslint-plugin-mystyles` exports a style editor called `es5`. In order to apply the style editor to code, the end-user would add a `style` property to the configuration object:

```js
module.exports = {
    plugins: ["mystyles"],
    parserOptions: {
        ecmaVersion: 5
    },
    style: "mystyles/es5"
}
```

The `style` property can be used at the top level of the configuration to apply to all files or in an `overrides` object to apply to a subset of files.

Options for style editors can be specified using a `styleOptions` key, as in this example:

```js
module.exports = {
    plugins: ["mystyles"],
    parserOptions: {
        ecmaVersion: 5
    },
    style: "mystyles/es5",
    styleOptions: {
        indent: "tabs"
    }
}
```

The `styleOptions` property is just an opaque object to ESLint, without any predefined properties. Style editors can define their options and ESLint will pass the `styleOptions` object directly into the style editor through `context.options`.

### Style Editor Format

A style editor is an object literal with a `meta` property to provide information about the style editor and an `edits` property that is an array of the style edits to apply, for example:

```js
// style editor format
module.exports = {
    meta: {
        name: "ES5 Style",
        description: "A style editor just for ES5"
    },
    edits: [
        // individual style edits
    ]
}
```

The `edits` array contains individual objects in this form:

```js
{
    type: "edit type",
    edit(context) {
        // modify the source code
    }
}
```

This format of style edit is intentionally generic so that we can add new style edit types in the future. For the purposes of this proposal, I am starting with just one type: `text`. An example text edit looks like this:

```js
{
    type: "text",
    edit(context) {
        return context.sourceText.replace(/\r\n?/g, "\n");
    }
}
```

This fixer simply normalizes line endings from Windows-style to Unix-style. It accesses the source code text through `context.sourceText` and returns a string. Each `text` fixer must return a string with the modifications to the source code.

The `context` object has the following properties:

* `sourceText` - the source code that will eventually be linted
* `filename` - the name of the file being fixed
* `options`- the `styleOptions` object from configuration
* `parserPath` - the path to the parser ESLint will use to parse this file
* `parserOptions` - the `parserOptions` object specified in configuration

By making the `text` style edit work with strings, this allows the use of both simple string methods and third-party tools directly.

An example using Prettier:

```js
{
    type: "text",
    edit(context) {
        const prettier = require("prettier");
        return prettier.format(context.sourceText, { semi: false, parser: "babylon" });
    }
}
```

An example using [Recast](https://github.com/benjamn/recast) (example is from Recast README):

```js
{
    type: "text",
    edit(context) {
        const recast = require("recast");
        const ast = recast.parse(context.sourceText);

        // Grab a reference to the function declaration we just parsed.
        var add = ast.program.body[0];

        // Make sure it's a FunctionDeclaration (optional).
        var n = recast.types.namedTypes;
        n.FunctionDeclaration.assert(add);

        // If you choose to use recast.builders to construct new AST nodes, all builder
        // arguments will be dynamically type-checked against the Mozilla Parser API.
        var b = recast.types.builders;

        // This kind of manipulation should seem familiar if you've used Esprima or the
        // Mozilla Parser API before.
        ast.program.body[0] = b.variableDeclaration("var", [
            b.variableDeclarator(add.id, b.functionExpression(
                null, // Anonymize the function expression.
                add.params,
                add.body
            ))
        ]);

        // Just for fun, because addition is commutative:
        add.params.push(add.params.shift());

        // return the modified code
        return recast.print(ast).code;
    }
}
```

Any JavaScript utility that can work on a string and return a string can be used inside a `text` style edit.

Because the style edits have a defined order inside of a style editor, end-users can prescribe any number of steps to edit styles in source code. For example, you might want to run a Recast edit and then a Prettier edit:

```js
// style editor
module.exports = {
    meta: {
        name: "ES5 Style",
        description: "A style editor just for ES5"
    },
    edits: [
        require("./edits/recast"),
        require("./edits/prettier")
    ]
}
```

This ability to order individual edits means that end-users could potentially run Prettier to get a baseline format for their source code and then run additional fixes to tweak the style in whichever way they like. Or they could perform a bunch of AST transforms using Recast and then apply Prettier to format everything.

### Style Editor Composition

Because style editors are just objects, end-users can compose new style editors from existing ones. For example, if someone wanted to use an Airbnb style editor but tweak the indentation scheme a bit, they could create a new style editor like this:

```js
const airbnbStyle = require("eslint-plugin-airbnb").styles.default;

module.exports = {
    meta: {
        name: "Modified Airbnb Style",
        description: "A style editor based on Airbnb"
    },
    edits: [
        ...airbnbStyle.edits,
        require("./edits/indentation")
    ]
}
```

This style editor first applies the edits from the Airbnb style editor and then applies a custom indentation scheme. The possibilities for mixing and matching style editors are limitless, and all of it can be done without involving the ESLint team.

### Order of Operations

When ESLint is called with the `--fix` or `--fix-dry-run` flag, the style editor is applied to the source code after lint fixes are applied and before one final lint without fixes. The complete order of operations is as follows:

1. Preprocess source code using configured processors (if any)
1. Lint source code
1. Apply lint fixes to source code
1. Apply style editor to source code
1. Lint source code
1. Postprocess source code using configured processors (if any)

Because autofixing might change the AST, and therefore introduce style inconsistencies, multipass autofixing must happen before the style editor is applied. And because style editors could potentially introduce linting errors, one additional lint operation must take place before the source code results are reported. This last lint operation does not do any autofixing and only ensures that the final source code is given the correct linting messages.

**Important:** Style editors are not applied when the `--fix` and `--fix-dry-run` flags are not used.

### Error Reporting

If a style editor attempts to use its own parser to parse text, and the parser throws an error, ESLint will report the error in the same format as a parser error thrown during linting.

If a style editor only modifies text, and the modified text throws an error while the linter is parsing it, the same error format is displayed as it would with any source code that hasn't been style fixed.

### Using with `--fix-type`

In order to allow control over when style editors will be applied, a fourth option will be added to `--fix-type` called `style`. When `style` is specified using `--fix-type`, any specified style editors will be applied.

To apply only the style editor, an end-user would run:

```
$ eslint --fix --fix-type style *.js
```

To apply just layout rules and the style editor, an end-user would run:

```
$ eslint --fix --fix-type layout,style *.js
```

This would *not* run a style editor and only fix problems:

```
$ eslint --fix --fix-type problem *.js
```

### Implementation Details

I envision the implementation as follows:

1. Update `CLIEngine` and `cli` to accept `style` as a valid option for `--fix-type`.
1. Create a `Style` class to represent individual style editors.
1. When plugins are loaded, look for style editors in addition to rules and processors.
1. Style editors would be loaded and applied from `CLIEngine`.
1. Add a `Linter#defineStyle()` method to store style editors inside of `Linter`.

## Documentation

This feature would need a new documentation section to explain how to create style editors. Additionally, the following pages would need to be updated:

* [Configuring ESLint](https://eslint.org/docs/user-guide/configuring#configuring-eslint)
* [Working with Plugins](https://eslint.org/docs/developer-guide/working-with-plugins#working-with-plugins)
* [Command Line Interface](https://eslint.org/docs/user-guide/command-line-interface#command-line-interface)

## Drawbacks

* This proposal adds new functionality that will have to be taught to existing users.
* It will take some time for the community to create style editors and therefore adoption might be slow.
* Applying style guides does add some complexity to the linting process that doesn't already exist.
* Adding two additional steps to the autofixing process will slow down the process as a whole by some unknown amount.
* Style editors do not allow for programmatic detection of whether or not the style has been applied.

## Backwards Compatibility Analysis

Because style editors are a new feature built on top of existing features, backwards compatibility is not a concern.

If people don't specify a style editor, then their experience doesn't change at all.

## Alternatives

An alternative approach is to somehow merge this functionality into the existing autofix functionality. Changing the `fixable` value on rules to be `text` (or other options) and then make the `fix()` method receive different parameters based on that type. I started down this path initially but decided that would keep us stuck in the same rules-based scenario that we are in currently.

Another alternative approach is to implement style editors as simple functions rather than a collection of edits. This would allow anything to be done to the source code that an end-user would like, but ESLint would lose visibility into what was going on. Ultimately, the current proposal allows the same use case if an end-user would prefer to create just one large style edit instead of a bunch of smaller ones.

Prior art includes:

* [Prettier](https://prettier.io)
* [Recast](https://github.com/benjamn/recast)
* [JSCodeShift](https://github.com/facebook/jscodeshift)

## Open Questions

### Does it make sense to bundle style editors with the fix commands?

My initial thought was yes, that end-users would expect the `--fix` and `--fix-dry-run` command to make all possible changes to the source code that ESLint knows about. As such, it would make sense to include it as an option with `--fix-type` as well. 

The alternative would be to add something like a `--style` flag that would have to be used to apply style editors. I'm not completely against this but it seems like this would be less expected than using the existing fix options.

### Should style edits have IDs?

For the purpose of composition, it seems like style edits might benefit from having an `id` property so that other style editors can programmatically search for just the edits they want. I'm not sure if that's necessary, however, and perhaps that's too much of an API contract.

## Frequently Asked Questions

### Can I specify more than one style editor?

No. The intent is that `style` is always specified as a single string value, similar to how `parser` works. Style editor composition allows developers to mix and match parts of style editors to come up with the result that they want, so specifying multiple style editors should not be necessary.

### What other type of style edits could possibly exist?

At the moment, I have two other style edits in mind:

1. `layout` - provides an API for tweaking spacing, semicolons, commas, and parentheses. While I have an idea about how this would work, it's a much more involved edit than `text`, so I will submit a separate RFC for this in the future.
1. `ast` - provides an API for doing AST transforms within the context of the AST ESLint already generates. I think this could potentially be a more efficient way of doing AST transforms than using tools like Recast, but once again, this is a much more involved edit and I will submit a separate RFC for this in the future.

I wouldn't want to spend the time designing these features until after style editors ships with just `text` edits first.

### How will this affect the use of stylistic rules in ESLint?

My hope is that once style editors are implemented, it will allow us to deprecate stylistic rules and get us out of the constant need to add options to cater to everyone's preferences about how their code should look. Instead, developers will use style editors and make the tweaks themselves, leaving us completely out of the loop.

Eventually, I'd like to see all stylistic changes done inside of style editors such that no one uses stylistic rules anymore. (Admittedly, this is a long-term vision and would not happen immediately upon implementation of this proposal.)

### Who will write the style edits?

Once again, my hope is that this will end up being done by the ESLint community, allowing them to make their own modifications when existing style editors don't suit their purpose.

My thought was that I'd spend some time writing a few basic style edits that could be included in style editors as a starting point, but I'd maintain those myself. I don't want the ESLint team to have to maintain a bunch of style edits because that situation would be too similar to what we have now with stylistic rules.

### Why do style editors use an array instead of a method?

An alternative design could have style editors implement an `edit()` method instead of an `edits` array. That design gives us less flexibility in a few ways:

1. With individual style edits, ESLint can determine the best way to apply one style after another. For instance, with a `text` style edit, ESLint knows that it's not necessary to parse the code before passing it into the style edit. Similarly, if there are two `text` style edits in a row, ESLint knows that the result of the first style edit doesn't need to be parsed yet and can be passed directly into the second `text` style edit. This flexibility of knowing what types of style edits need to be applied is important for performance optimiziation.
2. With a method, other style editors would have no choice but to apply all of another style editor before apply their own changes. With an array, it's possible for one style editor to pick and choose which edits from another style editor to include. 
