-   Start Date: 2019-07-20
-   RFC PR: https://github.com/eslint/rfcs/pull/33
-   Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Description in directive comments

## Summary

This RFC adds the description support into directive comments such as `/*eslint-disable*/`. For example, `/* eslint-disable no-new -- this class has a side-effect in the constructor and it's a library's. */`.

## Motivation

Directive comments make exceptions for static analysis. The explanation of why you made the exception here helps to maintain the code. However, ESLint doesn't have a comfortable way to write such an explanation.

## Detailed Design

ESLint ignores the part preceded by `\s-{2,}\s` in directive comments.

```js
/* eslint eqeqeq: off -- this part is ignored. */
/* eslint-disable -- this part is ignored. */
/* eslint-enable -- this part is ignored. */
// eslint-disable-line -- this part is ignored.
// eslint-disable-next-line -- this part is ignored.
/* global foo -- this part is ignored. */
/* globals foo, bar -- this part is ignored. */
/* exported foo, bar -- this part is ignored. */

"Longer is OK."
// eslint-disable-line -------- this part is ignored.

"Multiline."
/* eslint-disable
    a-rule,
    another-rule
    --------
    this part is ignored.
    this part is ignored. */

"Not affect to '--' which is not surrounded by spaces."
/* eslint spaced-comment: [error, { exceptions: ["--"] }] */
```

### Implementation

```js
// Use only the part before the first `--`.
const [value] = comment.value.split(/\s-{2,}\s/u)
```

## Documentation

-   The "[Disabling Rules with Inline Comments](https://eslint.org/docs/user-guide/configuring#disabling-rules-with-inline-comments)" section should describe `--`.

## Drawbacks

If a complex directive comment has `\s-{2,}\s` in the body, this feature breaks it.

## Backwards Compatibility Analysis

This is a breaking change technically, but I believe the effect is very limited.

## Alternatives

-   Another comment around directive comments.
    ```js
    /* explanation */
    /* eslint-disable */
    f()
    ```
    In some cases, it's challenging to write another comment near directive comments. For example, there is `//eslint-disable-line` comment on a long line.

### Prior Arts

-   [istanbul](https://github.com/istanbuljs/istanbuljs)
    > ```
    > /* istanbul ignore <word>[non-word] [optional-docs] */
    > ```
    >
    > https://github.com/gotwarlost/istanbul/blob/master/ignoring-code-for-coverage.md
-   [PHP_CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer)
    > ```
    > // phpcs:disable PEAR,Squiz.Arrays -- this isn't our code
    > ```
    >
    > https://github.com/squizlabs/PHP_CodeSniffer/wiki/Advanced-Usage#ignoring-parts-of-a-file
-   [TSLint](https://github.com/palantir/tslint) doesn't have this feature, but there is a discussion: https://github.com/palantir/tslint/issues/1484.

## Related Discussions

-   https://github.com/eslint/eslint/issues/11806
