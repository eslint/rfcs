- Repo: eslint/espree
- Start Date: 2020-12-13
- RFC PR: https://github.com/eslint/rfcs/pull/72
- Authors: Mike Reinstein (https://github.com/mreinstein)

# Adding es module support

## Summary

Provide an ecmascript module for espree.


## Motivation

Historically there have been a number of competing module standards: commonjs, requireJs, iife wrapped script tags, etc.

This has contributed to a very complicated tooling scenario for node/npm/javascript based projects in general:
* every module has it's own unique build system, with no consistency across modules.
* projects are packed with complex configurations, much of which is plumbing related to packaging/bundling.

Over time, browsers have adopted the ecma module standard. Today, support is pretty good:
https://caniuse.com/es6-module

Node also added support in v12.17.1 and up, and has officially marked the API stable:
https://nodejs.org/dist/latest-v15.x/docs/api/esm.html#esm_modules_ecmascript_modules


By adopting this new standard, we can achieve a few things:

* eventual simplification of espree's build process by eliminating the need for multiple module formats
* make it easier to directly importing espree in browser environments, deno, etc. Skypack is a CDN that already enables this, but they have internal logic that looks at every package in npm and tries to build an es module. https://www.skypack.dev/

As more projects adopt this standardized module format, the hope is a lot of this "connective glue" will fall away, leaving us with leaner modules and less build tooling across our stack.


## Detailed Design

### Nomenclature

* `ESM` ecmascript module. The standardized module format for javascript
* `CJS` commonjs. Node's historical module format (i.e., `module.exports = ...`, `const a = require('foo')`)
* `UMD` universal module definition. A legacy format for specifying javascript code as a module in a way that tries to guarantee compatibility with many loader formats including requireJS, commonjs, and global script tags.
* `IIFE` immediately invoked function expressions. A legacy strategy for encapsulating javascript into a module that won't leak global variables into the execution environment. It is still the primary way of packaging global script tags for inclusion in a web page. `UMD` is usually wrapped in an `IIFE`
* `CDN` content delivery network. can be thought of as a set of servers that host static content all around the globe to deliver assets quickly and at scale.


### build process

Today, espree is written in commonjs (CJS) format, and a universal module definition (UMD) build is created too.


today's build process:
```
┌-------------------┐               ┌-----------------┐
│                   │               │                 │                package.json:
│     espree.js     |               │ build/espree.js │
│ (CJS entry point) ├──BUILD_STEP──▶│   (UMD bundle)  │     CJS ---▶   "main": "espree.js",
│                   │               │                 │
└-------------------┘               └-----------------┘
```


The strategy that probably makes the most sense in this initial work is to do what is known as a "dual package" solution, where
espree provides an ESM, and a CJS module.

Providing a dual package shouldn't affect any existing espree users, but provies an ESM option for users that want it.


proposed build process in this RFC:
```
                                  ┌-------------------┐
                                  │                   │
                                  │  dist/espree.cjs  │
                                  │ (CJS entry point) │                package.json:
┌-------------------┐             │                   │
│                   │             └--▲----------------┘     CJS ---▶   "main": "dist/espree.cjs",
│     espree.js     │                │                                 "exports": {
│ (ESM entry point) ├───BUILD_STEP───┘                      ESM ---▶     "import": "espree.js",
│                   │                                       CJS ---▶     "require": "./dist/espree.cjs"
└-------------------┘                                                  },
                                                                       "type": "module"
```

browserify is specifically oriented around commonjs as the input format, so I also propose replacing it with rollup, which understands and expects ESM as it's input format.

`UMD` is not part of the formal package interface and will be dropped altogether.


## Documentation

The changes should be described in the migration guide for whichever major version of espree this goes into (currently considering the upcoming 8.x line.)

Adding an `exports` field to `package.json` will break the existing API, it makes sense to formally announce this via blog post.


## Drawbacks

The javascript build/tooling ecosystem is already monstrously complex, and this change adds another output format, which further complicates things (compare the 2 graphs above to see this additional complexity visualized.)

I do believe this is a temporary scenario; as time goes on, the UMD and CJS entry points could be dropped altogether, along with the build step. But that will take months, maybe even years depending on how comfortable the community is with this change and how widely adopted the ESM format becomes.


## Backwards Compatibility Analysis

The dual package approach _should_ not break espree for anyone currently using espree.


## Alternatives

Another option is to  just drop CJS, UMD and the build system altogether, provide an ESM only implementation,
and make this a part of a semver-major version bump. (e.g., espree@8 is ESM only.)

This would essentially eliminate the build step and make things very simple, but it has a big downside:
* users that don't support ESM only projects would be stuck on `espree <= 7.x` forever (or at least until they added the ability to use ESM in their own stuff.)


## Open Questions

None yet.


## Help Needed

None yet.


## Frequently Asked Questions

> what is this "dual package" stuff?

It's a proposal by the node community on how a module may go about adopting ESM without totally breaking CJS for their users. https://nodejs.org/api/packages.html#packages_writing_dual_packages_while_avoiding_or_minimizing_hazards


> are any other well known modules using this approach?

acorn is producing a dual module, and has been the inspiration for my own work around the forthcoming espree PR https://github.com/acornjs/acorn/blob/master/acorn/package.json#L5-L11


> npm modules reference each other via commonjs, and the browser's loader expects URLs, so why are we concerned with making npm modules work in a browser?

that's true, you can't simply do `import espree from 'https://github/eslint/espree.js'` naively because it's referencing other npm modules in it, such as `acorn`.

However there are some CDNs which will transform npm modules to refer to URLs. For example, this works today:

```javascript
import espree from 'https://cdn.skypack.dev/espree'
```

By making espree a standard ESM it reduces the amount of transforms that need to be run on it to make it usable.

It's also possible that in the semi-near future, node may consider offering URL loaders. Deno is already doing this.


## Related Discussions

- [1] The discussion genesis https://github.com/eslint/espree/issues/457
