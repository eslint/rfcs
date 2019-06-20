- Start Date: 2019-05-12
- RFC PR: https://github.com/eslint/rfcs/pull/20
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Configuring Additional Lint Targets with `.eslintrc`

## Summary

This proposal adds the ability to specify additional target files into configuration files. This enhancement will solve the pain that people have to use the `--ext` option with wanted file extensions even if they use plugins which support additional file types.

## Motivation

People have to use the `--ext` option or glob patterns to check wanted files even if they use plugins which support additional file types.

```yml
plugins:
  - markdown
  - html
  - "@typescript-eslint"
  - react
  - vue
```

```bash
# ESLint checks only `*.js`.
eslint src docs

# Needs `--ext` option
eslint src docs --ext .js,.md,.html,.ts,.jsx,.tsx,.vue
```

However, using plugins which support additional file types is the intention that wants to check those files. The requirement of redundant CLI options is not reasonable.

Per "[Related Discussions](#related-discussions)" section, this is very regularly requested; I want to configure additional target files with `.eslintrc`.

## Detailed Design

This proposal enhances `overrides` property of the config file.

- If a config file in a directory has `overrides` property, ESLint checks the files which are matched by any of override entries (i.e., `files`/`excludedFiles` criteria) additionally in the directory.
    - If any of `files` value of an override entry ended with `*`, this enhancement doesn't use the entry in order to avoid checking too many kinds of files.
- This enhancement affects only the case where a directory path is provided on the CLI. If people provide glob patterns on the CLI, ESLint behaves the same as currently.
- The `--ext` option precedences this enhancement. If the `--ext` option was given, this enhancement is disabled. So people can use the current behavior by `--ext .js`.
- The ignoring configuration (`.eslintignore`) precedences this enhancement. If `.eslintignore` contains the additional target files, ESLint just ignores those as same as currently.

The `overrides` property means that people intend to check those files. So this behavior is intuitive.

<table><td>
💡 <b>Example</b>:
<pre lang="yml">
overrides:
  - files: "*.ts"
    parser: "@typescript-eslint/parser"
  - files: "tests/**/*"
    env: { mocha: true }
</pre>

With the above config, `eslint .` command will check `*.ts` files additionally. But the command doesn't check all files inside `tests` directory.
</td></table>

### Code blocks

If a code block that processors extracted has the virtual filename, ESLint filters the file extension of the virtual filename with `--ext` option. This enhancement affects to that check. ESLint lints the code block if the `overrides` matches the virtual filename.

### Legacy file extension processors

This enhancement **doesn't** affect legacy file extension processors. This means that `plugins` property in config files never changes the kinds of target files.

On the other hand, `extends` property in config files can change the kinds of target files by the `overrides` property in the shareable configs.

### Implementation

- For files, we can add the check to [lib/cli-engine/file-enumerator.js#L410](https://github.com/eslint/eslint/blob/553795712892c8350b1780e947f65d3c019293a7/lib/cli-engine/file-enumerator.js#L410).
- For code blocks, we can add the check to [lib/cli-engine/cli-engine.js#L248](https://github.com/eslint/eslint/blob/21f3131aa1636afa8e5c01053e0e870f968425b1/lib/cli-engine/cli-engine.js#L248).

## Documentation

This enhancement needs migration guide because of a breaking change.

- If your config contains `overrides` property, `eslint` command with directory paths lints the files which are matched automatically. This may increase errors of your command.<br>
  If you don't want to add file types to check, please use glob patterns instead of directory paths or `--ext .js` option.

This enhancement, so it needs to update some documents.

- In the description of `--ext` CLI option, it should say that your config file may add file types automatically.
- In the description of `overrides` property, it should say that the `overrides[].files` property adds target files automatically.

## Drawbacks

- This is a breaking change.
- Implicit behavior may be confusing people.

## Backwards Compatibility Analysis

In the following situation, `eslint` command increases errors.

- Their configuration has `overrides` property (with `files` property which doesn't end with `*`).
- Their `eslint` command is using directory paths without `--ext` option.

I guess that the impact is limited because people use `--ext` option or glob patterns if they use configuration which affects files other than `*.js`.

## Alternatives

- To add `extensions` property that behaves as same as `--ext` option. Explicit reduces confusion. However, plugins and shareable configs cannot add the `extensions` property without the breaking change that drops old ESLint support.

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [eslint/eslint#801](https://github.com/eslint/eslint/issues/801) - Configurable Extension Filter
- [eslint/eslint#1674](https://github.com/eslint/eslint/issues/1674) - Allow setting file extensions in .eslintrc
- [eslint/eslint#2274](https://github.com/eslint/eslint/issues/2274) - Configure file extensions in .eslintrc
- [eslint/eslint#2419](https://github.com/eslint/eslint/issues/2419) - Allow configuration of default JS file types
- [eslint/eslint#7324](https://github.com/eslint/eslint/issues/7324) - File extensions in .eslintrc
- [eslint/eslint#8399](https://github.com/eslint/eslint/issues/8399) - Add .jsx to the default --ext extensions list
- [eslint/eslint#10828](https://github.com/eslint/eslint/issues/10828) - Support specifying extensions in the config
- [eslint/eslint#11223](https://github.com/eslint/eslint/issues/11223) - Add --ext to `eslintrc` or other config files
