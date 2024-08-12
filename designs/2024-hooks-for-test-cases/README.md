- Repo: eslint/eslint
- Start Date: 2024-08-12
- RFC PR:
- Authors: [Anna Bocharova](https://github.com/RobinTail)

# Hooks for test cases

## Summary

<!-- One-paragraph explanation of the feature. -->

I propose adding an optional `setup` property to the test cases that the ESLint `RuleTester` runs. This feature
will allow developers to prepare specific environments needed for certain rules before running each test case.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

Some rules require more awareness of the user environment, such as dependencies listed in the `package.json` file.
That information is not provided to the rule by `RuleContext`, therefore the rule has to read it from disk.

For example: [`eslint-plugin-import/no-extraneous-dependencies`](https://github.com/import-js/eslint-plugin-import/blob/09476d7dac1ab36668283f9626f85e2223652b37/src/rules/no-extraneous-dependencies.js#L23)

Generally, testing a rule using `RuleTester` implies running the linter against the variety of `code` samples and
rule `options` both `valid` and `invalid` in order to cover all possible scenarios of its operation. However, testing
a rule having behavior depending on user's `package.json` becomes more challenging, since the user environment becomes
another variable that has to be different for every test case.

For example: [see the tests of the rule mentioned above](https://github.com/import-js/eslint-plugin-import/blob/09476d7dac1ab36668283f9626f85e2223652b37/tests/src/rules/no-extraneous-dependencies.js#L21-L29)

The currently well-known approach is fixtures: having actual files on disk, but those files are separated from the
test cases, they have to be maintained (they can be renamed, deleted or changed regardless the cases).
The proposed `setup` hooks provides the place to keep the environment preparation, such as mocking the returns of the
file system methods, right within the test cases along with other case variables: `code` and `options`.

The expected outcome is a more streamlined and maintainable testing process:
- Enhanced Developer Experience: developers can quickly understand and modify tests;
- Improved Readability: Keeping the setup logic within the test case makes it easier to read and debug;
- Reduced Maintenance: Eliminating the need for numerous fixture files reduces the overhead of maintenance.

Ultimately, this feature will support more efficient and effective testing of rules that depend on user environment.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

The proposed feature introduces the `setup` property to each test case in the ESLint rule tester.
This property is an optional function that runs before the linting process for that specific test case.

### Implementation

- Change the test case schema in the ESLint rule tester to include the new property. 
  - The property is a function that takes no arguments and returns `void`.
- Modify the `RuleTester`:
  - It should check for the presence of the setup function within each test case. 
  - If the `setup` function exists, it should be executed before running the test.

### Example

Here is an example of how test case definitions would look with the new `setup` property:

```javascript
new RuleTester().run("my-custom-rule", myCustomRule, {
  valid: [
    {
      code: '/* valid code example */',
      options: [/* plugin options */],
      setup: () => {
        // Mock `package.json` or other necessary setup
        jest.mock('fs', () => ({
          readFileSync: jest.fn().mockReturnValue(JSON.stringify({
            dependencies: { "some-package": "^1.0.0" }
          })),
        }));      
      }
    }
  ],
  invalid: [
    {
      code: '/* invalid code example */',
      errors: [{ messageId: "someErrorId" }],
      setup: () => {
        // Mock different `package.json` setup
        jest.mock('fs', () => ({
          readFileSync: jest.fn().mockReturnValue(JSON.stringify({
            devDependencies: { "another-package": "^2.0.0" }
          })),
        }));
      }
    }
  ]
});
```

### Corner Cases

- `setup` throws `Error`: then the test case should fail, and the error should be reported;
- Developer must ensure that the `setup` function does not inadvertently affect global state in a way that impacts
  other test cases.
- Teardown Considerations: while a `teardown` function is not necessary for the current proposal, it may be worth
  considering for a symmetry and completeness in more complex scenarios:

```javascript
const teardown = () => {
  jest.resetAllMocks(); // Explicitly clean up any mocks or state changes
}
```

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

## Drawbacks

<!--
    Why should we *not* do this? Consider why adding this into ESLint
    might not benefit the project or the community. Attempt to think 
    about any opposing viewpoints that reviewers might bring up. 

    Any change has potential downsides, including increased maintenance
    burden, incompatibility with other tools, breaking existing user
    experience, etc. Try to identify as many potential problems with
    implementing this RFC as possible.
-->

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- Prior issue: https://github.com/eslint/eslint/issues/18770
- Draft implementation PR: https://github.com/eslint/eslint/pull/18771