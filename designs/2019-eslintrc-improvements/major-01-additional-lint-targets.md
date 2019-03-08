# Config File Improvements: Additional Lint Targets

## Summary

This proposal adds the ability to specify additional target files in configuration files. This enhancement will solve the pain that people have to use `--ext` option with wanted file extensions even if they use plugins.

## Motivation

People have to use `--ext` option or glob patterns to check wanted files even if they use plugins to support additional file types.

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

## Detailed Design

> Proof of Concept (implementation): https://github.com/eslint/eslint/tree/proof-of-concept/config-array-in-eslintrc

This proposal enhances [there](README.md#additional-lint-targets).

When `FileEnumerator` checks if a file is matched by the given extensions, additionally, it checks the file is matched by any of `files` property of configs (except `files` patterns which end with `*` to avoid unlimited).

> [lib/lookup/file-enumerator.js#L338](https://github.com/eslint/eslint/blob/153640180a8944af3a1c488462ed30d0c215f5ed/lib/_lookup/file-enumerator.js#L338) in PoC.

<table><td>
💡 <b>Example</b>:
<pre lang="yml">
overrides:
  - files: "*.ts"
    extends: "plugin:@typescript-eslint/recommended"
</pre>

With the above config, `eslint .` command will check `*.ts` files additionally.
</td></table>

If a plugin has file extension processors, [it yields config array elements which have `overrides` property](README.md#yield-file-extension-processor-element), so this enhancement affects file extension processors.

<table><td>
💡 <b>Example</b>:
<pre lang="yml">
plugins:
  - vue # has `.vue` processor
</pre>

With the above config, `eslint .` command will check `*.vue` files additionally.
</td></table>

The ignoring configuration (`.eslintignore`) is prior to this enhancement. If `.eslintignore` contains the additional target files, ESLint just ignores those as same as currently.

## Documentation

This enhancement needs migration guide because of a breaking change.

- If your config contains `overrides` property or plugins which have file extension processors, `eslint` command with directory paths lints the files which are matched automatically. This may increase errors of your command.<br>
  If you don't want to add file types to check, please use glob patterns instead of directory paths.

This enhancement, so it needs to update some documents.

- In the description of `--ext` CLI option, it should say that your config file may add file types automatically.
- In the description of `overrides` property, it should say that the `overrides[].files` property adds target files automatically.
- In the description of `plugins` property, it should say that plugins which have file extension processors add target files automatically.
- In the "working with plugins" page, it should say that file extension processors add target files automatically.

## Drawbacks

- This is a breaking change.
- Implicit behavior may make people confused.

## Backwards Compatibility Analysis

In the following situation, `eslint` command increases errors.

- Their configuration has `overrides` property (with `files` property which doesn't end with `*`) or plugins which have file extension processors.
- Their `eslint` command is using directory paths without `--ext` option.

I guess that the impact is limited because people use `--ext` option or glob patterns if they use configuration which affects files other than `*.js`.

## Alternatives

- To add `extensions` property that behaves as same as `--est` option. Explicit reduces confusion. However, plugins and shareable configs cannot add the `extensions` property without the breaking change that drops old ESLint support.

## Open Questions

-

## Frequently Asked Questions

-

## Related Discussions

- [eslint/eslint#11223](https://github.com/eslint/eslint/issues/11223)
