- Start Date: 2019-11-18
- RFC PR: (leave this empty, to be filled in later)
- Authors: Gareth Jones ([@g-rath](https://github.com/g-rath))

# First-Class Support For `.eslintrc.ts`

## Summary

Support the ability to consume `.eslintrc.ts` configuration files natively.

## Motivation

Currently configuration settings are validated by JSON Schema, which while
powerful does not have ubiquitous & high quality support across editors &
tooling.

It also tends to generate somewhat cryptic messages, especially when using
complex elements such as combining `oneOf`, `anyOf`, and other such schemas.

Meanwhile, TypeScript is a powerful & popular superset of Javascript that has
high quality support as a first class language in the majority of editors, and
is quickly gaining ubiquitous support in general.

By allowing first-class support for `.eslintrc.ts`, the amount of configuration
problems & general confusion over such experienced by users can be reduced,
since it would enable us to provide tooling to type configuration files and so
allow TypeScript to catch ESLint configuration errors at compile time, rather
than run time.

While we can already hack ways to use `.eslintrc.ts`, none of them lend
themselves to ease-of-use in the same manner that ESLint encourages, and so
first-class support for `.eslintrc.ts` will go long way towards encouraging
community driven typing of ESLint plugins.

Finally, ESLint is being adopted as the linter for TypeScript, meaning
first-class support for `.eslintrc.ts` will help enable a positive feedback loop
to help improve the two tools.

There are some potential UX benefits for users that use & understand TypeScript,
in the form

## Detailed Design

- Adds support for `.eslintrc.ts`
- Requires uses to install `ts-node` manually adjacent to ESLint in
  `node_modules`.
  - Failure to do so with a `.eslintrc.ts` results in ESLint throwing a fatal
    error
  - The error message should include understandable plain text instructions on
    how to fix the error.
- ESLint does not validate the configuration object for now.
- `.eslintrc.ts` is second highest priority, after `.eslintrc.js`
  - This matches the priority of the other supported types, and generally where
    TypeScript files it for other tools i.e webpack

## Documentation

I don't think a large amount of ceremony is required, but @typescript-eslint
might like to do something.

Overall, updating the list of accepted formats for `eslintrc`, and a small note
with a link to @typescript-eslint I think would be all that's needed.

@typescript-eslint probably would like to provide some documentation on how to
nicely type & write definitions for ESLint plugins, but that's not strictly
required by this feature.

## Drawbacks

> Slight increase in maintenance burden for core support.

People may open issues complaining about TypeScript-related errors, or errors
relating to the registering of `ts-node`.

The majority of these should be detectable from their stacktrace or use of
keywords such as TypeScript, `ts`, `ts-node`, and `.eslintrc.ts`. As such, I'd
expect it should be possible to refine `eslint-bot` to triage these issues and
post a comment detailing common problems & their solutions to help reduce this
cost.

Personally I don't think the general amount of maintenance that would result
from this change would be too drastic, as I'm confident that both the core
TypeScript and @typescript-eslint communities would be happy to foot the
maintenance bill for this change.

## Backwards Compatibility Analysis

This should be a completely backwards compatible change, since ESLint currently
ignores `.eslintrc.ts` files.

There appears to be no usage of `.eslintrc.ts` in _any_ public code on GitHub,
meaning no one is compiling `.eslintrc.ts` -> `.eslintrc.js`.

However, this can be made a complete non-issue by ensuring implementation favors
`.js` over `.ts` in the event both are on offer.

## Alternatives

> Use the TypeScript --checkJs` flag

While TypeScript can provide limited typechecking of js files via `--checkJs`,
it's no where near as powerful as native `.ts` type checking.

> Using a flag a la `--require`

While that would be nice, it would also increase the overall implementation
footprint, and (more importantly) defeat one of the great things about ESLint:
It Just Works (everywhere. with everything).

ESLint has near ubiquitous support in editors and tooling, meaning that simply
opening a project that has an ESLint configuration causes editors to traverse
code paths that search for ESLint, and if found, start linting any open files.

Supporting this feature behind a flag would be a poor middle ground that'd make
it far more painful to use, since you'd have to change your settings every time
to call ESLint with the flag, and that's assuming your editor or tool allows you
to pass additional flags to ESLint anytime it gets called!

> Can you just `ts-node/register` in `.eslintrc.js`?

i.e

```js
require('ts-node').register();
module.exports = require('./eslintrc.ts');
```

While the above would work, it means you'd have to have two `eslintrc` files;
this can also cause additional confusion as typically when compiling `.ts` files
their respective outputted `.js` file is placed next to the original file.
Hence, in the TypeScript community it'd be typically assumed `.eslintrc.js` is
the compiled form of `.eslintrc.ts`.

> Why not just compile `.eslintrc.ts`

Because that would require a build step to be run before every lint, and if you
forgot to do that, it would result in either confusing errors from ESLint about
rules you configured differently, or about config files not existing.

### Prior Arts

#### [Webpack](https://github.com/webpack/webpack)

Webpack allows the loading of `.ts` configuration files when `ts-node` is
installed.

#### [webpack-nano](https://github.com/shellscape/webpack-nano/blob/master/lib/config.js#L27-L64)

Uses [`rechoir`](http://github.com/tkellen/node-rechoir) to support arbitrary
module loading, including TypeScript. `rechoir` handles the automatic loading &
registering of transpilers without bundling them; allowing you to simply install
the transpiler required to

```js
const fileTypes = {
  '.esm.js': [esmRegister],
  '.es6': ['@babel/register', esmRegister],
  '.mjs': ['@babel/register', esmRegister],
  '.babel.js': [
    '@babel/register',
    'babel-register',
    'babel-core/register',
    'babel/register'
  ],
  '.babel.ts': ['@babel/register'],
  '.ts': [
    'ts-node/register',
    'typescript-node/register',
    'typescript-register',
    'typescript-require'
  ]
};

/* ... */

const requireLoader = extension => {
  try {
    rechoir.prepare(fileTypes, `config${extension}`, cwd);
  } catch (e) {
    /* ... */
  }
};
```

#### [karma](https://github.com/karma-runner/karma/blob/7617279f68507642fdc7eb0083916f3c6c2a28cb/lib/config.js#L36-L39)

Karma automatically register `ts-node` in their config parser to support
`karma.conf.ts`:

```
try {
  require('ts-node').register()
  TYPE_SCRIPT_AVAILABLE = true
} catch (e) {}
```

#### [DangerJS](https://danger.systems/js/guides/getting_started.html#creating-a-dangerfile)

> Create either a new file, either dangerfile.js or dangerfile.ts will be picked
> up automatically. Otherwise you can use an argument to pass in a custom file.

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

> if ESLint isn't written in TypeScript, will you still get type hinting for the
> config file?

Yes! TypeScript doesn't need the actual runtime code to be written in
Javascript - in fact, TypeScript source code is generally never shipped in npm
packages.

Instead, TypeScript consumes
[type definitions](https://www.typescriptlang.org/docs/handbook/modules.html#working-with-other-javascript-libraries)
which allow you to type pure Javascript libraries. These files have the
extension `.d.ts`, and you've probably seen them around in npm packages.

These definitions can (and are) shipped either with the npm package their
defining, or as a completely separate npm package.
[DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) exists
specifically to support this effort - it's a community driven repo that hosts,
maintains, and publishes type definitions for npm packages.

For example,
[DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/eslint/index.d.ts)
has eslint types based off of the standard estree spec, and
[typescript-eslint](https://github.com/typescript-eslint/typescript-eslint/tree/master/packages/experimental-utils/src/ts-eslint)
has types for eslint based off of the typescript-estree spec.

This means that even if an `eslint` plugin didn't provide types, someone in the
community could add them to `DefinitelyTyped` to enable TypeScript support for
that plugin.

This is an example of how a definition files might look for
`eslint-plugin-jest`:

```ts
// @types/eslint
type ErrorLevel = 0 | 1 | 2 | 'off' | 'warn' | 'error';
declare namespace ESLint {
  interface Rules {
    indent?:
      | ErrorLevel
      | [ErrorLevel, number | 'tabs', { SwitchCase: number /* ... */ }];
  }

  interface Config {
    rules: Rules;
  }
}

// @types/eslint-plugin-jest
declare namespace ESLint {
  interface Rules {
    'consistent-test-it'?:
      | ErrorLevel
      | [ErrorLevel, { fn?: 'it' | 'test' /* ... */ }];
  }
}

// .eslintrc.ts
const config: ESLint.Config = {
  rules: {
    'consistent-test-it': ['error', { fn: 'it' }]
  }
};
```

[Playground Link](https://www.typescriptlang.org/play/?ssl=1&ssc=1&pln=36&pc=1#code/PTAEAEBcE8AcFMDOwkBsCWA7SBYAUDAqAKIBOpA9qQDLwBu8qoAvKAAygA+oAjF6ACZ+AcgoAzMcJEB3AIalMU7sPjkqwgNz4AJvADGqefFCZZAWySxZe48QDK1LJFABvfKA+gnqsdeMAlAFdUJFd3TwisXWwAfgAuEjUaekZ+AG1wiKzEymSGVAAaTOzPTECzACNVEUhZCsRhIrwSrJdQO2l0SD0ACwBhWUR4BLLK1QLQYAAqUAA6edApsABfJpaAXS1mz2X8Yu9SXxtQPopMMXQAczDt7NJgpASgkMQtiN28D-wQCEIkFEQGGwAFpYKhApcsMCAFZIXB4XQGIwmcyWPwkBxOG6RbA+dHPUJuW5ZYR6M6IdCISDwEHUqnArrCeI5Ki0fLpYotMi5NmMNYtTxtMSYZnCRk1OGNSYzeazRYrfnZTbFD5fPA-WZoJykPSzSCIfBkzBU0BGi6XBL2RzYWanc5XFjYzz3F4JIktABERopVJpkGBdP9XQ9CQyxM9qlyHsVAqFmASHuDoFWnM86xjKc+WyAA)

> So does this mean we'll have to write & maintain TypeScript definitions for
> all the rules?

Sort of. Technically it depends on where the definitions end up being hosted as
to who ideally should maintain them, but ultimately it will be a community
effort.

However, it's very easy to automate a large amount of the process, which has
actually already been done via
[this script](https://github.com/bradzacher/eslint-config-brad/blob/master/scripts/convertRuleOptionsToTypescriptTypes.ts),
which generated
[these types](https://github.com/bradzacher/eslint-config-brad/tree/master/src/types/eslint),
and actually found a number of bugs, including one in a base ESLint rule!
([#12051](https://github.com/eslint/eslint/pull/12051))

The majority of the work will be once-off, as once the definitions are written,
the maintenance burden will be roughly the same as that of maintaining the
schemas & docs.

Finally, this is actually technically out of scope of this RFS - whats being
described is something that can happen without involvement from the ESLint core
team.

> Will `.eslintrc.ts` be recompiled every time it's loaded?

Potentially yes, due to the bypassing of the config cache by ESLint.

In theory, the TypeScript compiler instance should be persisted between loads
meaning it should be able to leverage its own cache to not recompile the config
if it hasn't changed.

> Won't this slow down my linting?

In theory this will increase initial lint speeds, but it should only be by a few
seconds, depending on config size, since TypeScript only compiles what it needs
to, and it's rare to import core project files into eslint configuration.

## Related Discussions

https://github.com/eslint/eslint/issues/12078
https://github.com/eslint/eslint/pull/12082
