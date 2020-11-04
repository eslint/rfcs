-   Repo: eslint/eslint
-   Start Date: 2020-11-04
-   RFC PR: (leave this empty, to be filled in later)
-   Authors: Giuseppe Gurgone

# eslint-ignore-next-line directive

## Summary

I propose to introduce an alias for the `eslint-disable-next-line` directive called `eslint-ignore-next-line`.

## Motivation

Especially for next-line disable directives, I rarely remember it and use `-ignore-` instead of `-disable-`.

```js
// The following are equivalent

// eslint-ignore-next-line no-console
console.log("eslint");

// eslint-disable-next-line no-console
console.log("eslint");
```

## Detailed Design

The change would be trivial and backwards compatible. I would implement it myself if you approve this RFC.
