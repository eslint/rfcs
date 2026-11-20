- Repo: eslint/eslint
- Start Date: 2026-11-19
- RFC PR: https://github.com/eslint/rfcs/pull/156
- Authors: Nicholas C. Zakas

# Reduce allocations in `SourceCode#traverse()`

## Summary

This RFC proposes changing `SourceCode#traverse()` to return a compact
`TraversalSteps` value backed by two parallel arrays in addition to its current `Iterable<TraversalStep>`, with the intent of deprecating `Iterable<TraversalStep>`. A `TraverseStepsBuilder` in `@eslint/plugin-kit` will provide
an API for creating this representation. The traversal consumer will process
the arrays with an indexed loop, reducing allocations and iterator overhead.
This is a breaking API change and is intended for a major release.

## Motivation

An analysis of ESLint's performance found that `SourceCode#traverse()` creates
many short-lived objects. Each traversal step currently has a step object, and
its `args` array adds another object, so every node visit incurs a cost of at least four
objects, two for each step. These allocations increase garbage
collection work and slow linting.

Using an iterator avoids eagerly returning an array of steps, but iterator
overhead makes it slower in practice, which is why we always return an array.
This RFC proposes a structure that creates only two arrays for the traversal itself 
(in addition to arrays needed for function-call arguments), and allows the consumer
to use a conventional indexed loop.

## Detailed Design

### `TraverseSteps` representation

`SourceCode#traverse()` will be updated to also return a `TraverseSteps` object with parallel
`targets` and `details` arrays:

```ts
interface TraverseSteps {
    targets: Array<unknown>;
    details: Array<number | Array<unknown> | undefined>;
}
```

Each index in `targets` and `details` describes one step:

| Step | `targets[i]` | `details[i]` |
| --- | --- | --- |
| Enter a node | The node | `1` |
| Exit a node | The node | `2` |
| Call a traversal function | The function name | Its argument array, or `undefined` when there are no arguments |

For node steps, `1` denotes the enter phase and `2` the exit phase. A call step
is identified by its string target; a node step has a node target. These correspond to how the same information is represented in the current approach. The arrays
must always have the same length, and entries at a given index must describe
the same step.

When measured locally, this approach saved 9-10% of run time vs. the current approach.

### Builder

`@eslint/plugin-kit` will provide a `TraverseStepsBuilder` to create the
representation without requiring language implementations to manage its
parallel arrays directly:

```js
const steps = new TraverseStepsBuilder();

steps.enter(node);
steps.exit(node);
steps.call("functionName", [arg1, arg2]);

return steps;
```

The builder will append the target and its corresponding detail together for
each operation. `enter(node)` and `exit(node)` add the node and its phase.
`call(name, args)` adds the function name and arguments; calls without
arguments can omit `args`.

For backwards compatibility, the object returned from `TraverseStepsBuilder` will have a `Symbol.iterator` method that yields the same `TraversalStep` object format that consumers currently expect, meaning that will fulfill the `Iterable<TraversalStep>`. This is necessary so that updated language plugins will continue to work with older versions of ESLint. These plugins, however, will not realize any gains for this change and, in fact, will get a bit slower.

### Consuming steps

The traversal consumer will recognize `TraverseSteps` and process its entries
in order with an indexed loop. It will dispatch node targets according to
their phase and invoke named traversal functions with the associated arguments.
The sequential representation does not prevent future support for asynchronous
rule listeners; the consumer can await each listener while preserving order.

In ESLint, this happens primarily in [`SourceCodeTraverser#traverseSync()`](https://github.com/eslint/eslint/blob/48207902fccd37450823b8fd07736d84449ec8e7/lib/linter/source-code-traverser.js#L288), which will be updated to also accept the new `TraversalSteps` format.

`SourceCodeTraverser#traverseSync()` will continue to accept the existing `Iterable<TraversalStep>`
format during migration. This lets language implementations move to the builder
independently. The iterable form is a compatibility input to the traversal
consumer; callers must not rely on the new `SourceCode#traverse()` return value
itself being iterable.

### Parent arguments

Several language plugins (`@eslint/markdown`, `@eslint/css`, `@eslint/shell`, `@eslint/json`) actually pass a parent node as the second argument to each rule listener (`(node, parent)`. This use
case is not supported in this RFC. No rule in these plugins use that second argument; they instead use `sourceCode.getParent(node)`, so making a change does not affect functionality. This will technically be a breaking change for the more mature language plugins but it should be a relatively painless one.

### API rollout

The [`SourceCodeBase#traverse()`](https://github.com/eslint/rewrite/blob/bf814c02dacf89735a94e4fb078ac483c2bcafd8/packages/core/src/types.ts#L1391) return type will change as will the return type and runtime value of `SourceCode#traverse()`. Direct
callers and tests that iterate the return value must be updated to use
`targets` and `details` (or consume steps through the traversal engine). This
is an intentional breaking change, so the change should ship in a major
release.

## Documentation

- Update the `SourceCode#traverse()` API documentation and type declarations to
  describe `TraversalSteps`, its parallel arrays, and the enter/exit phase
  values.
- Document `TraverseStepsBuilder` in `@eslint/plugin-kit`, including its
  `enter()`, `exit()`, and `call()` methods.
- Update language-plugin and custom-rule guidance to explain that parent nodes
  should be obtained with `sourceCode.getParent(node)`.

## Drawbacks

- Changing the return value breaks direct callers that expect an iterable.
- Language implementations and tests must be updated, including those that
  currently pass parent nodes as visit arguments.
- The parallel arrays are less self-describing than individual step objects and
  rely on preserving their alignment.
- Function calls that require arguments still need argument arrays.
- Language plugins that update and then are used with older versions of ESLint will actually get a bit slower due to the iterator protocol overhead. I think this is an acceptable tradeoff for improved performance in later versions of ESLint.

## Backwards Compatibility Analysis

This proposal is backwards compatible for ESLint itself. It will continue working with all existing language plugins even if they don't make this change.

This proposal is also backwards compatible for language plugins that use `TraverseStepsBuilder` to create the new return value.

## Alternatives

1. **Keep the iterable of step objects.** This avoids changing the API shape
   but retains iterator overhead and allocations for each step and its argument
   array.
2. **Return an array of step objects.** Iterating an array is straightforward,
   but still allocates an object for every step and an argument array where
   needed.
3. **Use a single flat array.** This could encode targets and details together,
   but would require consumers to decode a less explicit layout. Two parallel
   arrays make the step data and its alignment easier to inspect.

## Open Questions

- What benchmark should be used to quantify the allocation and runtime
  improvement before implementation is considered complete?
- Do we want to provide a fallback iterable (`Symbol.iterator` method) in the object returned from `TraversalStepsBuilder`? It could return the same `TraversalStep` format as we do now and that would prevent breakage in other consumers and older ESLint versions, although all of the performance gains would be completely lost.

## Help Needed

None.

## Frequently Asked Questions

None.

## Related Discussions

- [eslint/eslint#21556: Change Request: Update `SourceCode#traverse()` return value](https://github.com/eslint/eslint/issues/21556)
