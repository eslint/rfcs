- Start Date: 2019-09-29
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Lint files in parallel if many files exist

## Summary

This RFC adds the supports for parallel linting by `worker_threads`/`child_process` modules.

## Motivation

Linting logic uses CPU heavily, so parallel linting by threads may reduce spent time.

## Detailed Design

**PoC**: [eslint/eslint#very-rough-worker-threads-poc](https://github.com/eslint/eslint/tree/very-rough-worker-threads-poc/lib/eslint)

This RFC does the following items:

- Adds a new CLI option.
- Adds a new constructor option to `ESLint` class.
- Changes the behavior of `ESLint#executeOnFiles(patterns)` method.

### § New CLI Option

`eslint` command runs linting in parallel automatically if the target files are many. So this RFC adds two CLI options to let users control the behavior.

- `--concurrency <n>` ... The number should be a positive integer. If this is `1`, ESLint does linting in the main thread. Otherwise, ESLint creates this number of worker threads then the worker threads do linting.<br>
  Defaults to a dependent variable on the number of target files.

If `--concurrency` option is not present, ESLint estimates the proper value of the option. It's the number of target files divided by the constant `128`, but `os.cpus().length` at most.

```js
concurrency = Math.min(os.cpus().length, Math.ceil(targetFiles.length / 128))
```

This means that ESLint does linting in the main thread if the number of target files is less than `128` in order to avoid the overhead of multithreading. But ESLint does linting with using worker threads automatically if target files are many.

If people don't like this behavior, they can enforce arbitrary concurrency with `--concurrency` option.

If `--concurrency` option is present along with the following options, ESLint throws a fatal error.

- `--stdin`
- `--stdin-filename`
- `--init`
- `-h`, `--help`
- `-v`, `--version`
- `--print-config`
- `--env-info`

### § New constructor option of `ESLint` class

`ESLint` class is the new API that is discussed in [RFC 40](https://github.com/eslint/rfcs/pull/40).

This RFC adds a constructor option to `ESLint` class.

- `concurrency` (`number`) ... Corresponds to `--concurrency` CLI option. The number should be a positive integer. Defaults to a dependent variable on the number of target files.

This RFC doesn't change `CLIEngine` class because this requires asynchronous API to expose.

### § New behavior of `ESLint#executeOnFiles(patterns)` method

The overview is:

![Overview](diagrams/overview.svg)

1. Master enumerates the lint target files.
1. Master checks if the files exist in the lint result cache and it extracts only the files which are not cached.
1. Master estimates the proper `concurrency` from the number of the extracted files if not present.
1. Lint the extracted files:
   - If `concurrency` is `1`, master [verifies files][verify files] directly. It doesn't create any worker.
   - Otherwise, master creates workers then the workers [verify files]. The number of the workers is `concurrency`.
     1. Master sends the initial data to each worker. The initial data contains the constructor options of `CLIEngine` and several files.
     1. Workers [verify files]. It reports the lint result to master, and master may send the next file to the worker. The worker verifies the received file.
     1. Workers exit with `0` after it reported the result of all files.
1. Master collects the lint result from workers then write it to the lint result cache.
1. Master returns the merged result from cache and workers.

#### Worker's implementation

Worker can be `require("worker_threads").Worker` or `require("child_process").ChildProcess`.

If `v12.11.0` or newer version of Node.js runs ESLint, it's `require("worker_threads").Worker`. Otherwise it's `require("child_process").ChildProcess`.

#### About `--cache` option

We can use parallel linting along with `--cache` option because worker threads don't touch the cache. Master gives worker threads only files which are not cached (or cache was obsolete) and updates the cache with the collected result.

#### About `--fix` option

We can use parallel linting along with `--fix` option. ESLint modifies files after all linting completed.

#### About `--debug` option

We can use parallel linting along with `--debug` option, but ESLint cannot guarantee the order of the debug log. The order of the logs that are received from worker threads is messed.

#### Locally Concurrency

Now ESLint does linting in parallel by thread workers if target files are many.

Apart from that, each worker does linting concurrently by non-blocking IO to use CPU resource efficiently. This is a simple read-lint-report loop with `Promise.all()`:

```js
// Wait for all loops finish.
await Promise.all(
  // Run the loop concurrently.
  initialFiles.map(async initialFilePath => {
    let filePath = initialFilePath

    do {
      // Read file content by non-blocking IO.
      const text = await readFile(filePath, "utf8")
      // Lint the file.
      const result = verifyText(engine, filePath, text)
      // Send the lint result and get the next file path. This is also non-blocking async op.
      filePath = await pushResultAndGetNextFile(result)
    } while (filePath)
  }),
)
```

This logic is used in two places:

1. Each worker use it.
1. Master uses it in the main thread if target files are not many.

Therefore, if `concurrency` is `1` then:

```
Main thread
├ 🔁 Read-lint-report loop
├ 🔁 Read-lint-report loop
├ ...
└ 🔁 Read-lint-report loop
```

And if `concurrency` is greater than `1` then:

```
Main thread
├ ⏩ Worker thread
│    ├ 🔁 Read-lint-report loop
│    ├ ...
│    └ 🔁 Read-lint-report loop
├ ⏩ Worker thread
│    ├ 🔁 Read-lint-report loop
│    ├ ...
│    └ 🔁 Read-lint-report loop
├ ...
└ ⏩ Worker thread
      ├ 🔁 Read-lint-report loop
      ├ ...
      └ 🔁 Read-lint-report loop
```

#### Loading Config

On the other hand, this RFC doesn't change the logic that loads config files. It's still synchronous. Because the logic is large and the configs are cached in most cases.

We can improve config loading by async in the future, but it's out of the scope of this RFC.

### § Constants

This RFC contains two constants.

- `128` ... the denominator to estimate the proper `concurrency` option value.
- `16` ... concurrency for non-blocking IO.

Those values are changeable by measuring performance in real.

### § Performance

Please try the PoC.

```
npm install github:eslint/eslint#very-rough-worker-threads-poc
```

This section is an instance of performance measurement. The worker threads make ESLint over 2x faster if there are many files.

- Processor: Intel Core i7-8700 CPU (12 processors)
- RAM: 16.0 GB
- OS: Windows 10 Pro 64bit
- Node.js: 12.11.0

```
$ eslint lib tests/lib --concurrency 1
Number of files: 697
Lint files in the main thread.
Linting complete in: 19975ms

$ eslint lib tests/lib
Number of files: 697
Lint files in 7 worker threads.
Linting complete in: 7723ms

$ eslint lib --concurrency 1
Number of files: 368
Lint files in the main thread.
Linting complete in: 8237ms

$ eslint lib
Number of files: 368
Lint files in 4 worker threads.
Linting complete in: 4274ms
```

If forced to use workers with a few files, it's slower.

```
$ eslint lib/cli-engine
Number of files: 28
Lint files in the main thread.
Linting complete in: 1521ms

$ eslint lib/cli-engine --concurrency 2
Number of files: 28
Lint files in 2 worker threads.
Linting complete in: 2054ms
```

## Documentation

- The "[Command Line Interface](https://eslint.org/docs/user-guide/command-line-interface)" page should describe new `--concurrency` option.
- The "[Node.js API](https://eslint.org/docs/developer-guide/nodejs-api)" page should describe new `concurrency` option.

## Drawbacks

This feature introduces complexity for parallelism. It will be an additional cost to maintain our codebase.

This feature may not fit the TypeScript ecosystem. TS compiler parses whole codebase to check types but workers cannot share the parsed information. Every worker has to parse whole codebase individually.

## Backwards Compatibility Analysis

This feature doesn't change existing features.

We have the [parser services](https://eslint.org/docs/developer-guide/working-with-custom-parsers) API. The parser services can be implemented to use the shared state for target files. Also plugin rules can be implemented to use the shared state for target files.
As far as I know, such a parser/plugin doesn't exist. But if it existed, this feature breaks it because it cannot share state between worker threads.

I think the impact of this feature is very small.

## Alternatives

- [RFC 4](https://github.com/eslint/rfcs/pull/4) and [RFC 11](https://github.com/eslint/rfcs/pull/11) implement this feature behind a flag. On the other hand, this RFC does lint files in parallel by default if files are many, and people can enforce to not use workers by `--concurrency 1` option.

## Related Discussions

- https://github.com/eslint/eslint/issues/3565 - Lint multiple files in parallel [$500]
- https://github.com/eslint/eslint/issues/10606 - Make CLIEngine and supporting functions async via Promises
- https://github.com/eslint/eslint/pull/12191 - New: Allow parallel linting in child processes
- https://github.com/eslint/rfcs/pull/4 - New: added draft for async/parallel proposal
- https://github.com/eslint/rfcs/pull/11 - New: Lint files in parallel

[verify files]: #locally-concurrency
