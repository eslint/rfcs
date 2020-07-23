- Repo: eslint/eslint
- Start Date: 2020-06-04
- RFC PR: https://github.com/eslint/rfcs/pull/56
- Authors: Ilya Volodin

# Generic AST support

## Summary

Allow ESLint to traverse and lint generic (non estree compliant AST) and allow parsers to specify information about AST that would drive ESLint's traversal and features.

## Motivation

ESLint has cemented itself as a go to tool for JavaScript/TypeScript ecosystem for style checking and linting. However, there are a lot of connected languages/technologies that are used every day by ESLint users that are either lacking linters, or require yet another tool to be setup and configured for the repository. Examples include HTML, CSS, SCSS, LESS, GraphQL, JSON, etc.
With addition of config overrides users can now setup separate configuration per glob, so that would allow them to have a single instance of ESLint and a single configuration that would allow linting of the entire repository, including all non javascript/typescript files.

## Detailed Design

### Initial assumptions
* This RFC does not expect or implies that existing core or plugin rules will work for any files that are parsed through non-estree compliant parsers. Every parser that chooses to output custom AST will have to provide new rules that will work for this AST in the form of plugins.
* This RFC doesn't propose full feature parity for new parser with existing ESLint capabilities. For example, it doesn't propose a new way to provide custom Code Path Analyzer, or support inline directives for controlling ESLint's flow for custom ASTs. However, those functionalities can be added later with a separate RFC(s).

Currently ESLint has a lot of hardcoded assumptions about AST. For example, it assumes that nodes are identified by the property called `type` and that property exists on every node. It assumes that top-most node is always `Program` and that AST supports code-path-analysis (which is very specific to JavaScript). This RFC proposes to modify expected output from the `parseForESLint` method of the custom parsers to include additional property called `parserSupportOptions` of type object with four new properties (for now):
 - nodeIdentifier: function | string | undefined - Specifies a function that can retrieve the name of the ASTNode property that identifies node. Alternatively, can be a string that just returned the name of the property that identifies all nodes. If the property is not provided, ESLint will default to `type` for backwards compatibility.
 - codePathAnalysis: "off" | "default" - Indicates AST's support for built-in CodePathAnalysis support. Defaults to `”default”` for backwards compatibility.
 - rootNode: string - Provides the name of the top level node in the AST. Defaults to `Program` for backwards compatibility.
 - metadata: { language: string } - Used for informational purposes only. Identifies name of the language that this parser supports.

This RFC also proposes modifying existing returned property "scopeManager" that's already returned by `parseForESLint`. Currently, while documentation states, that this property is not required, ESLint will crash if returning `null`. Instead, it can be modified to return tri-state value of type `"off" | "default" | ScopeManager`. If "off" is returned, scope analysis will be completely disabled for files parsed by custom parser. "default" will specify to use ESLint's default scope manager (eslint-scope). Providing an object of type `ScopeManager` will require ESLint to use it instead of analyzing code through eslint-scope.

 This change would allow basic support of linting non-estree compliant ASTs. This will still not enable full feature parity with ESLint's abilities when linting JavaScript. In the future new properties might be added to support parsing comments to enable inline directives for controlling rules. Other enhancements might include ability to provide replacement for CodePathAnalysis and scopes creation.

## Documentation

This will be documented as part of the page that describes [working with custom parsers](https://eslint.org/docs/developer-guide/working-with-custom-parsers). In addition, blog post would let other users know that they can start creating parsers and rules for new languages. Although, it might be a good idea to wait until there's an example in the wild that could be pointed to as a reference.

## Drawbacks

The only drawback that I can think of, is that JavaScript specific rules will not work for other ASTs. Each language will most likely have to implement their own set of rules, and they will be incompatible across different parsers. While this might cause some confusion for users, I do believe this makes sense and there is no need to try to somehow generalize rules to support multiple ASTs.

## Backwards Compatibility Analysis

This change will be fully backwards compatible. In the future, default nodeIdentifier described above could be removed and moved into espree instead.

## Alternatives

None that I can think of, other then not to support any non-compliant estree ASTs.

## Open Questions

* Is it OK to release incomplete support for other AST types? Inline directives being the biggest obstacle, from my perspective.
* Should `nodeIdentifier` be passed into rules through `context` to allow creations of very generic rules that can support multiple languages/ASTs?
* Is `parserSupportOptions` descriptive enough name? I wasn't able to come up with anything better.

## Help Needed

I have a working branch with a large refactoring effort to remove hardcoded `type` nodeIdentifier. Everything is working and all tests are passing, except for one in code-path-analysis. Unfortunately, I wasn't able to figure out what is causing this problem. Code is available here: https://github.com/ilyavolodin/eslint/tree/ast
Another problem, is that I wasn't able to find a way to pass `nodeIdentifier` into formatters, since they live higher in the stack then execution of the custom parser provided. I need suggestions on how to fix this. Not all parsers require this information, but some might.

## Related Discussions

* https://github.com/eslint/eslint/pull/13305
