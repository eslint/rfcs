- Repo: eslint/eslint
- Start Date: 2024-08-12
- RFC PR: https://github.com/eslint/rfcs/pull/122
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
The proposed `setup` hook provides the place to keep the environment preparation, such as mocking the returns of the
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

The new `setup` property should be formally documented in the following ways:

- ESLint Rule Tester Documentation: Update the official ESLint `RuleTester` documentation to include detailed
  information about the new property. This documentation should explain:
  - The purpose and use cases of the setup property;
  - How to implement the setup function in test cases;
  - Example demonstrating the usage of the `setup` property for both `valid` and `invalid` test cases;
- Changelog: Include an entry in the ESLint changelog outlining the new property with a brief description;

### Announcement

To ensure that the ESLint community is aware of the new feature and understands its motivation and usage, a formal
announcement should be made on the ESLint blog. This announcement can include:

- Introduction to the Feature: Explain what the setup property is and provide context as to why it was introduced;
- Motivation: Describe the motivation for introducing the setup property, including the challenges with the current
  approach and the anticipated benefits of the new feature;
- Examples and Use Cases: Provide concrete examples and use cases to show how the setup property can be used to
  improve the testing process for ESLint plugins;
- Link to Documentation: Include links to the updated documentation having more detailed information.

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

Possible concerns:

- Potential for Misuse:
  - Despite being an optional feature, there remains a risk that some users might misuse the setup property
    by inadvertently modifying global state, leading to flaky tests.
- Performance:
  - There is a risk that developers might overload the setup property, leading to performance decrease.
- Redundancy: confusion with `beforeEach`:
  - It is important to emphasize the key difference between these entities in the documentation:
    - `beforeEach` is for **consistent** actions applicable to **all** tests;
    - `setup` is for **specific** actions applicable to **particular** test.
- Limited Use Case:
  - The necessity for the `setup` might be limited to specific rules and plugins. The broader ESLint user
    community may not frequently encounter these challenges, and it may seem unjustified to them.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

Adding new optional property to the test case is backwards compatible.
When the property is not set, `RuleTester` acts the same for existing users — no disruption expected.

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

My current workaround is assigning a custom `it` function to the `RuleTester` that looks up the for a mock by test name:

```javascript
const mockedEnvironments = {
  "test case one": { dependencies: { "some-package": "^1.0.0" } },
  "test case two": { devDependencies: { "another-package": "^2.0.0" } }
}

RuleTester.it = (name, ...rest) => {
  jest.mock('fs', () => ({
    readFileSync: jest.fn().mockReturnValue(
      JSON.stringify( mockedEnvironments[name] )
    ),
  }));
  it(name, ...rest);
};

new RuleTester().run("my-custom-rule", myCustomRule, {
  valid: [
    {
      name: "test case one",
      code: '/* valid code example */',
      options: [/* some options */]
    }
  ],
  invalid: [
    {
      code: '/* invalid code example */',
      errors: [{ messageId: "someErrorId" }],
      options: [/* some options */]
    }
  ]
});
```

However, I'm not satisfied with this approach due to a number of disadvantages:

- The `name` property is required to be set on each test case so that `it` could distinguish between them;
- Also, that `name` must be unique for proper lookup;
- And most importantly: mocked environment is detached from the test cases definitions:
  - They are having only that weak reference via the `name`;
  - It's harder to read, to maintain and to make some adjustments;
  - The mocked environment is, in essence, one of the arguments to the rule being tested,
    therefore it does make sense for them to be near `options` and `code`.

## Open Questions

<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them, 
    you can remove this section.
-->

- Should there also be the `teardown` property?

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

The implementation of `setup` is already ready in the mentioned pull request, however, I may need recommendations
regarding changes to the documentation.

## Frequently Asked Questions

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

- **Why not use `beforeEach` to achieve the same functionality?**
  - While `beforeEach` is effective for common setups applied across all tests in a suite, it falls short when
    different tests require unique setups. Using `beforeEach` for such cases would necessitate complex conditional
    logic to differentiate setups, leading to harder-to-maintain and less readable code. The `setup` property,
    on the other hand, allows for context-specific configurations directly within each test case, making the
    tests easier to read and maintain.
- **How would the new `setup` property affect existing test cases?**
  - The `setup` property is an opt-in feature. Existing test cases will remain unaffected as they do not utilize
    this new property. Only test cases that specifically define the `setup` function will be impacted.
    Since `RuleTester` invalidates the unknown props in test cases, no existing users should be affected.
    This ensures backward compatibility and does not force any changes on users who prefer existing testing methodology.
- **Will the `setup` property introduce performance overhead?**
  - While the execution of individual `setup` functions could introduce some overhead, it is comparable to the
    overhead introduced by `beforeEach` hook. Users are encouraged to keep their setup functions lightweight and
    efficient. The benefit of having isolated and context-specific setups typically outweighs the minor performance
    costs, especially in complex testing scenarios.
- **What if users require a `teardown` function?**
  - I can add it to my implementation.
- **What are the specific use cases where `setup` is more beneficial than existing solutions?**
  - The setup property is particularly beneficial in scenarios where:
    - Different test cases require different environmental configurations;
    - Tests need to be highly readable and maintainable, with a preparation logic kept closely to each test case;
    - Users need to avoid the complexity of conditional logic in shared hooks like `beforeEach`.

## Related Discussions

<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->

- Prior issue: https://github.com/eslint/eslint/issues/18770
- Draft implementation PR: https://github.com/eslint/eslint/pull/18771
