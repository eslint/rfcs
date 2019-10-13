- Start Date: 2019-10-13
- RFC PR: (leave this empty, to be filled in later)
- Authors: Toru Nagashima ([@mysticatea](https://github.com/mysticatea))

# Drop supports for Node.js 8.x and 11.x

## Summary

This RFC drops supports for Node.js 8.x and 11.x because of end-of-life. (See [nodejs/release](https://github.com/nodejs/release#readme) to check the release schedule of Node.js)

## Motivation

We get the capability to use new features and syntaxes by dropping the support for end-of-life versions of Node.js. Especially, [RFC40](https://github.com/eslint/rfcs/pull/40) wants to use async iteration, but Node.js 8.x doesn't support that syntax.

## Detailed Design

This proposal updates the `engines` field of our `package.json` (and updates our CI scripts).

```diff
   "engines": {
-    "node": "^8.10.0 || ^10.13.0 || >=11.10.1"
+    "node": "^10.12.0 || >=12.0.0"
   }
```

### Why is it `^10.12.0`?

> **Resolution:** We will choose Node version support based on the lowest point release supporting our desired features.
>
> [2019-March-14 ESLint TSC Meeting Notes: Decide how to manage support for minor versions of Node](https://github.com/eslint/tsc-meetings/blob/137fbe2be55499cd75f28a008900803078d22dfd/notes/2019/2019-03-14.md#decide-how-to-manage-support-for-minor-versions-of-node)

Therefore, we should check the added features in Node.js `10.x`.

I found two features we want to use.

- [10.10.0](https://nodejs.org/en/blog/release/v10.10.0/) ... the `withFileTypes` option of `fs.readdir`/`fs.readdirSync`. We can reduce the calls of `fs.statSync` in our `FileEnumerator` class with this option. It may improve performance (especially on Windows).
- [10.12.0](https://nodejs.org/en/blog/release/v10.12.0/) ... `module.createRequireFromPath(filename)` function. We can remove [our polyfill](https://github.com/eslint/eslint/blob/24ca088fdc901feef8f10b050414fbde64b55c7d/lib/shared/relative-module-resolver.js#L20-L29) with this version.

## Documentation

We need write an entry in the migration guide because this is a breaking change.

## Drawbacks

Users lose the capability to run ESLint on old Node.js. They may have to update their CI scripts.

## Backwards Compatibility Analysis

This is a breaking change clearly.

Users lose the capability to run ESLint on old Node.js. They may have to update their CI scripts.

## Alternatives

N/A.

## Related Discussions

- https://github.com/eslint/eslint/issues/10981 - Allow use node version >8
- https://github.com/eslint/eslint/issues/11022 - Decide how to manage support for minor versions of Node
