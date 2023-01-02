- Repo: eslint/eslint
- Start Date: 2022-12-30
- RFC PR: (leave this empty, to be filled in later)
- Authors: [Brad Zacher](https://github.com/bradzacher), [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Parsing Session Objects

## Summary

Parsers are not currently told whether ESLint is being run in a CLI-style single pass, a single pass with `--fix` enabled, or a long-running IDE session.
This RFC proposes the creation of a "session" object that provides them this information.

## Motivation

Forking from https://github.com/eslint/eslint/discussions/16557#discussioncomment-4219483: custom parsers sometimes would want to make decisions in stateful logic based on the way ESLint is being run.
There are generally three classifications ESLint sessions for these stateful parsers:

- "Single" runs: which can have a reusable (_immutable_ program) cache, and doesn't need to attach file watchers
- "Single, with fixers": which have a _mostly_ reusable cache, except for file fixes, which we generally assume not to change cross-file types
- "Continuous" runs: the "wild west" (_builder_ program), and should attach file watchers

From knowing session classification, parsers can then make more informed decision on aspects such as file/program caching.

As discussed in https://github.com/eslint/eslint/discussions/16557#discussioncomment-4160219, at least two major ecosystem plugins would benefit from understanding session information at _parse_ time:

- [`eslint-plugin-import`](https://github.com/import-js/eslint-plugin-import)'s out-of-band parsing (imports analysis)
- [`@typescript-eslint/parser`](https://typescript-eslint.io)'s [typed linting mode](https://typescript-eslint.io/linting/typed-linting)

### Benefits to Out-of-Band Parsing

Having an unchanged object reference provided to parsers for the duration of a lint run -and whose identity is guaranteed to change across runs- allows parsers to use objects as a `WeakMap` key for caches.
That way, out-of-band analyses such as imports analysis can be kept persistently for each file within a run.
The caches won't go stale in a long-running eslint process.

### Benefits to Typed Linting

Typed linting on the typescript-eslint repo shows an immediate **10% performance improvement** from typescript-eslint being able to assume a "single" session mode (https://github.com/typescript-eslint/typescript-eslint/pull/3512 -> https://github.com/typescript-eslint/typescript-eslint/issues/3528).
However, bugs occur when `--fix` is enabled (https://github.com/typescript-eslint/typescript-eslint/issues/6176 -> https://github.com/typescript-eslint/typescript-eslint/issues/6184).

Using the session proposed in this RFC to determine whether it's in a "single" session mode _without_ `--fix` would allow typescript-eslint to fix those bugs (https://github.com/typescript-eslint/typescript-eslint/pull/6178).

## Detailed Design

This RFC proposes a `session` object be provided by ESLint to parsers that at first contains just two pieces of information:

```ts
type SessionMode = "persistent" | "single";

interface LintSession {
  mode?: SessionMode;
  options: ESLintOptions;
}
```

The expectation for this session object is that consumers may provide a `mode` _if_ they know it.
Parsers would then receive the session in their [`parseForESLint`](https://eslint.org/docs/latest/developer-guide/working-with-custom-parsers) as a third parameter.

```js
parser.parseForESLint(textToParse, parserOptions, lintSession);
```

If a mode isn't provided, consumers should assume `"persistent"`.
That's what parsers today have to assume, and doesn't remove any functionality the way `"single"` does.

### Code Flow

Consumers of ESLint today use up to four classes from ESLint:

- `Linter`: A lower-level class used when `fs` is not available, such as in browsers.
- `CLIEngine` _(legacy)_: [no longer exported as of ESLint v8.0.0](https://eslint.org/docs/latest/user-guide/migrating-to-8.0.0#remove-cliengine); API predecessor to `ESLint`.
- `ESLint`: The primary class to use in Node.js applications that includes `fs` operations.
- `FlatESLint`: A preview of `ESLint` that uses the new [flat config](https://github.com/eslint/rfcs/pull/9)

For example, the [ESLint extension for VS Code](https://github.com/Microsoft/vscode-eslint) [uses either `CLIEngine` or `ESLint`](https://github.com/Microsoft/vscode-eslint/blob/f095ba58a6bbefd5b7e5ddcff778f3f04551f554/server/src/eslint.ts#L1017-L1025), depending on the ESLint version and its configuration.

#### `Linter`

In `lib/linter.js`, the standalone `parse` function is what calls to `parseForESLint`.
It is called within the `Linter` class.
That `Linter` class can be given two additions to support passing a lint session object to `parse`:

- A third property on its config object, `session`
- The addition of that `session` object to the stored `internalSlotsMap` data for the linter

The `Linter` class will also enforce the invariant assumed by the session object when `mode` is `"single"`.
I.e. if you're running single mode, then each file MUST be linted exactly once (unless fixed).

```ts
const linter = new ESLint({ session: { mode: 'single' },  ... });

// No errors on these lines
linter.lintFiles('foo.ts');
linter.lintFiles('bar.ts');

// Error: Cannot lint the same file twice without fixes in 'single' mode
linter.lintFiles('foo.ts');
```

#### `CLIEngine`

The `CLIEngine` class is still used internally by the `ESLint` class to create a new `Linter`.
That means it cannot assume it is being run by the CLI.
It therefore needs to receive its session information as a constructor parameter.

`CLIEngine` already receives a large `CLIEngineOptions` object as a constructor parameter.
`CLIEngineOptions` can be given an optional new property `session?: SessionOptions`.

```ts
interface SessionOptions {
  mode?: SessionMode;
}
```

`CLIEngine`'s constructor can then provide its `options.session` to its `new Linter`:

```diff
- const linter = new Linter({ cwd: options.cwd });
+ const linter = new Linter({ cwd: options.cwd, session: options.session });
```

#### `ESLint`

The `ESLint` class already receives a large `ESLintOptions` object as a constructor parameter.
`ESLintOptions` can be given an optional new property `session?: SessionOptions`.

For example, the `lib/cli.js` > `translateOptions` would add `session: { mode: "single" }` to the `options` later passed to `ESLint`:

```diff
const options = {
  // ... (existing options) ...
+  session: { mode: "single" },
};
```

`ESLint` can then pass `options.session` to the `CLIEngine`:

```diff
- const cliEngine = new CLIEngine(processedOptions, { preloadedPlugins: options.plugins });
+ const cliEngine = new CLIEngine(processedOptions, { preloadedPlugins: options.plugins, session: options.session });
```

#### `FlatESLint`

This works roughly the same as `ESLint` internally, and so would be changed in a very similar way.

### Object Identity

The `session` object is intentionally kept referentially equal across all uses of a `Linter`.

## Documentation

At least the [Architecture](https://eslint.org/docs/latest/developer-guide/architecture), [Node.js API](https://eslint.org/docs/latest/developer-guide/nodejs-api), and [Working with Custom Parsers](https://eslint.org/docs/latest/developer-guide/working-with-custom-parsers) documentation pages will need to be updated to document this addition.

A formal announcement may not be necessary, as this change only impacts parsers with stateful needs as described under _Motivation_.

## Drawbacks

It may not be desirable to keep adding to the `parseForESLint` function.
Adding a third parameter after a second, optional parameter means parsers that prefer to know session information but do not read from options now have to "skip" that second parameter:

```js
function parseForESLint(code, _options, session) {
  /* ... */
}
```

The complete rewrite of ESLint proposed in https://github.com/eslint/eslint/discussions/16557 would allow for rewriting the parsing API in a way that does not create a function with three parameters.

## Backwards Compatibility Analysis

This is generally an opt-in, non-breaking change.
Existing parsers don't expect a third argument from ESLint and so generally shouldn't be impacted by this addition.

The only possibility of an impact would be an existing parser that adds a third parameter to its exported `parseForESLint` function.
This might be the case for a parser that reuses that function for non-ESLint-parsing contexts, such as test-only behaviors or other npm libraries.
A [search on Sourcegraph](https://sourcegraph.com/search?q=context:global+/%5E%5Cw*%28export.*%29%3F%28%28%28var%7Clet%7Cconst%29%5CW%2B%5Cw%2B%5CW%2B%29%7Cfunction%29+parseForESLint/+-file:.d.ts+-file:.js.flow+-file:%28%5E%7C/%29node_modules/+-file:.pnpm-store.*&patternType=standard&sm=0) for `parserForESLint` function declarations and variables does not find any.
It is the belief of this RFC's authors that that is sufficient evidence to qualify this as a _non-breaking_ change.

## Alternatives

### Add to `parserOptions`

Instead of adding a third parameter to `parseForESLint`, the `session` object could be added to the `parserOptions` object provided to parsers.
This would avoid complicating the `parseForESLint` function signature.

However, doing so would prevent users from writing their own `parserOptions.session`.
This RFC initially prefers adding a third parameter instead of nesting in the second parameter.
But either form would be equivalently useful.

### Breaking Change `parseForESLint`

We could breaking change `parseForESLint`'s second, options parameter to be an object that contains both _options_ and _session_:

```ts
interface ESLintParseContext {
  options?: ParserOptions;
  session?: LintSession;
}
```

Doing so would require _all_ community-authored parsers be updated to detect the newer ESLint API usage.
That would likely cause a high amount of community pain.

### Environment Variables

Instead of adding to `parseForESLint`, ESLint could instead set `process.env` variables to signal its runtime mode.

```js
process.env.ESLINT_LINT_MODE = options.session.mode;
process.env.ESLINT_LINT_OPTIONS_FIX = options.session.options?.fix;
```

However:

- It would only work in Node.js-like environments, not browsers
- More complex options, such as events described in https://github.com/eslint/eslint/discussions/16557#discussioncomment-4219483, would not be feasible

## Open Questions

The term `"single"` for the CLI use case is a bit of a misnomer, as the CLI can repeat up to 10 times when `--fix` is enabled.
Is there a better antonym for `"persistent"`?

Another definition for `"single"` is: any way that ESLint is _triggered once_ to the end-user, such as by the CLI.

## Help Needed

I'm not confident I understand the ESLint Node API well enough to have not missed some use cases.

## Frequently Asked Questions

### Will this overcomplicate parsers?

> Or: does adding this new parameter make parsers more complex?

No - if anything, it makes them simpler!
Stateful parsers have implemented some gnarly systems to infer ESLint's session mode.
See the [`inferSingleRun.ts` file in typescript-eslint](https://github.com/typescript-eslint/typescript-eslint/blob/c50b89e69dcb69b2de721b3786f47d537420fa40/packages/typescript-estree/src/parseSettings/inferSingleRun.ts).

### Why add this now, instead of to ESLint's proposed rewrite?

In short: a significant number of users would immediately benefit from the performance improvements unlocked by this information.

The [`@typescript-eslint/eslint-plugin` npm package](https://www.npmjs.com/package/@typescript-eslint/eslint-plugin) receives about two thirds as many downloads as the [`eslint` package](https://www.npmjs.com/package/eslint).
Only roughly 10-15% of users who configure typescript-eslint appear to be using typed linting:

- [Roughly 23.8k public ESLint configurations appear to extend from `plugin:@typescript-eslint/recommended`](https://sourcegraph.com/search?q=context:global+file:%28.*%29%28eslintrc%7Cpackage%5C.json%29%28.*%29+%22plugin:%40typescript-eslint/recommended%22+count:9000001&patternType=standard&sm=1) _(which does not enable typed linting)_
- [Roughly 3.1k public ESLint configurations appear to enable `@typescript-eslint/no-floating-promises`](https://sourcegraph.com/search?q=context:global+file:©.*eslint%7Cjs%7Cjson%7Cyaml%7Cyml%7Cyaml.*+/no-floating-promises.*%28true%7C1%7C2%7Cerror%7Cwarn%29/+-file:.*node_modules.*+-file:dist%5C/.*+count:200000&patternType=standard&sm=0) _(which requires typed linting)_
- [Roughly 2.3k public ESLint configurations appear to extend from `plugin:@typescript-eslint/recommended-requiring-type-checking`](https://sourcegraph.com/search?q=context:global+file:%28.*%29%28eslintrc%7Cpackage%5C.json%29%28.*%29+%22plugin:%40typescript-eslint/recommended-requiring-type-checking%22+count:9000001&patternType=standard&sm=1) _(which requires typed linting)_

The performance benefits mentioned in this RFC would benefit the repositories using typed linting.
Furthermore, being able to optimize performance based on ESLint session mode is a blocker for typescript-eslint more aggressively recommending typed linting to users.

Given that it may be years before many of ESLint's users are able to benefit from the proposed rewrite -and longer yet before it is polished enough to be recommended to most users- we believe the benefits of this RFC to many thousands of repositories outweigh the technical burden.

## Related Discussions

- https://github.com/eslint/eslint/discussions/16557
- https://github.com/typescript-eslint/typescript-eslint/issues/3528
