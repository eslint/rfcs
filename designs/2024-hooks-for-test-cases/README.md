- Repo: eslint/eslint
- Start Date: 2024-08-12
- RFC PR: https://github.com/eslint/rfcs/pull/122
- Authors: [Anna Bocharova](https://github.com/RobinTail)

# Hooks for test cases

## Summary

<!-- One-paragraph explanation of the feature. -->

I propose adding an optional `before` (and `after`) properties to the test cases that the ESLint `RuleTester` runs. This
feature will allow developers to prepare specific environments needed for certain rules before running each test case.

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
The proposed `before` hook provides the place to keep the environment preparation, such as mocking the returns of the
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

The proposed feature introduces the `before` and `after` properties to each test case in the ESLint rule tester.
These properties are an optional functions that runs before and after the linting process for that specific test
case accordingly.

### Implementation

- Change the test case schema in the ESLint rule tester to include the new properties.
  - The properties are functions that take no arguments and return `void`.
- Modify the `RuleTester`:
  - It should check for the presence of the `before` function within each test case;
  - If the `before` function exists, it should be executed before running the test;
  - When the `after` function exists, it should be executed after running the test regardless of its result.

### Example

Here is an example of how test case definitions would look with the new properties:

```javascript
const readerMock = jest.fn();
jest.mock('fs', () => ({
  readFileSync: readerMock,
}));

new RuleTester().run("my-custom-rule", myCustomRule, {
  valid: [
    {
      code: '/* valid code example */',
      options: [/* plugin options */],
      before: () => {
        // Mock `package.json` or other necessary setup
        readerMock.mockReturnValueOnce(JSON.stringify({
          dependencies: { "some-package": "^1.0.0" }
        }));      
      },
      after: () => {
        readerMock.mockReset();
      }
    }
  ],
  invalid: [
    {
      code: '/* invalid code example */',
      errors: [{ messageId: "someErrorId" }],
      before: () => {
        // Mock different `package.json` setup
        readerMock.mockReturnValueOnce(JSON.stringify({
          devDependencies: { "another-package": "^2.0.0" }
        }));
      },
      after: () => {
        readerMock.mockReset();
      }
    }
  ]
});
```

### Corner Cases

- `before` or `after` throws `Error`: then the test case should fail, and the error should be reported;
- `after` should be executed even when `before` throws in order to minimize potential impact on other test cases;
- Developer must ensure that the `before` function does not inadvertently affect global state in a way that impacts
  other test cases.

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The new `before` property should be formally documented in the following ways:

- ESLint Rule Tester Documentation: Update the official ESLint `RuleTester` documentation to include detailed
  information about the new property. This documentation should explain:
  - The purpose and use cases of the `before` property;
  - How to implement the `before` function in test cases;
  - Example demonstrating the usage of the `before` property for both `valid` and `invalid` test cases;

### Announcement

To ensure that the ESLint community is aware of the new feature and understands its motivation and usage, a formal
announcement should be made on the ESLint blog. This announcement can include:

- Introduction to the Feature: Explain what the `before` property is and provide context as to why it was introduced;
- Motivation: Describe the motivation for introducing the `before` property, including the challenges with the current
  approach and the anticipated benefits of the new feature;
- Examples and Use Cases: Provide concrete examples and use cases to show how the `before` property can be used to
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

- Checking duplicates among the test cases will not work when `before` or `after` property is present:
  - This drawback comes from the nature of the serialization approach used for comparing test cases;
- Potential for Misuse:
  - Despite being an optional feature, there remains a risk that some users might misuse the `before` property
    by inadvertently modifying global state, leading to flaky tests.
- Performance:
  - There is a risk that developers might overload the `before` property, leading to performance decrease.
- Redundancy: confusion with `beforeEach`:
  - It is important to emphasize the key difference between these entities in the documentation:
    - `beforeEach` is for **consistent** actions applicable to **all** tests;
    - `before` is for **specific** actions applicable to **particular** test.
- Limited Use Case:
  - The necessity for the `before` might be limited to specific rules and plugins. The broader ESLint user
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

My current workaround is assigning a custom `it` function to the `RuleTester` that looks up for a mock by test name:

```javascript
const readerMock = jest.fn();
jest.mock('fs', () => ({
  readFileSync: readerMock
}));

const mockedEnvironments = {
  "test case one": { dependencies: { "some-package": "^1.0.0" } },
  "test case two": { devDependencies: { "another-package": "^2.0.0" } }
}

RuleTester.it = (name, ...rest) => {
  readerMock.mockReturnValue(
    JSON.stringify( mockedEnvironments[name] )
  );
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

None

## Help Needed

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

I may require recommendations on placing the `after` handling and advices on changes to the documentation.

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
    logic to differentiate setups, leading to harder-to-maintain and less readable code. The `before` property,
    on the other hand, allows for context-specific configurations directly within each test case, making the
    tests easier to read and maintain.
- **How would the new `before` property affect existing test cases?**
  - The `before` property is an opt-in feature. Existing test cases will remain unaffected as they do not utilize
    this new property. Only test cases that specifically define the `before` function will be impacted.
    Since `RuleTester` invalidates the unknown props in test cases, no existing users should be affected.
    This ensures backward compatibility and does not force any changes on users who prefer existing testing methodology.
- **Will the `before` property introduce performance overhead?**
  - While the execution of individual `before` functions could introduce some overhead, it is comparable to the
    overhead introduced by `beforeEach` hook. Users are encouraged to keep their `before` functions lightweight and
    efficient. The benefit of having isolated and context-specific setups typically outweighs the minor performance
    costs, especially in complex testing scenarios.
- **What if users require a teardown function?**
  - They could use the corresponding `after` property for that purpose.
- **What are the specific use cases where `before` is more beneficial than existing solutions?**
  - The `before` property is particularly beneficial in scenarios where:
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
