-   Start Date: 2019-05-18
-   RFC PR: (leave this empty, to be filled in later)
-   Authors: Igor Novozhilov [@IgorNovozhilov]

# Events Model in CLIEngine. Event: "fileVerify"

## Summary

<!-- One-paragraph explanation of the feature. -->

This proposal describes the implementation of event support to handle the validation of each file separately when using the `executeOnFiles()` method,
both for synchronous validation and for asynchronous validation in the future, which was proposed earlier:
[eslint/rfcs #11](https://github.com/eslint/rfcs/pull/11) - 
[README.md](https://github.com/eslint/rfcs/blob/parallel/designs/2019-parallel-linting/README.md), 
[eslint/rfcs #4](https://github.com/eslint/rfcs/pull/4) -
[README.md](https://github.com/Aghassi/rfcs/blob/async-parallel/designs/2018-async-commands/README.md)

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

The events:

-   Will allow to improve integration with ESLint when developing other applications based on it.
    You will not need to complicate the code with additional constructs as in the example:
    [eslint/pull/11733#issuecomment-493302746](https://github.com/eslint/eslint/pull/11733#issuecomment-493302746).
    In particular, the code when you enable asynchronous file processing (see 'Summary'), will remain without significant changes.

    For example, synchronous validation:

    ```js
    const CLIEngine = require("eslint").CLIEngine;
    const cli = new CLIEngine({
        useEvents: true
    });

    cli.events.on("fileVerify", result => {
        // asynchronous sending of `result` to the database.
    });

    // lint my big project
    cli.executeOnFiles(["/all/*"]);
    ```

    and asynchronous:

    ```js
    const CLIEngine = require("eslint").CLIEngine;
    const cli = new CLIEngine({
        useEvents: true,
        anyAsyncEnableOption: true
    });

    cli.events.on("fileVerify", result => {
        // now the event itself is triggered asynchronously
    });

    // lint my big project
    cli.executeOnFiles(["/all/*"]);
    ```

-   Will provide increase performance and efficiency of system resources:

    -   Distribute the load over time in the report receiver, by sending data in batches, rather than one large piece
    -   Also, the work of the ESLint will accelerate when parallel processing is implemented (see 'Summary')

-   In addition, the events will be useful inside ESLint.
    When you run a scan for a large project, all data is analyzed first and then output to the console.
    Using events, you can output file validation data to the console sequentially for each file as the files are processed,
    without the long delay that is currently present when the linting.

## Detailed Design

<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

There is already an initial example of the implementation of this proposal:
[eslint/pull/11733](https://github.com/eslint/eslint/pull/11733).

Details:

1.  Events in the `class CLIEngine` are enabled by setting the option `useEvents: true`. (Default: false)
2.  In the class constructor sets the property `events`
3.  `events` is a `EventEmitter` instance [nodejs.org/api/events](https://nodejs.org/api/events.html#events_class_eventemitter)
4.  In the method `executeOnFiles` when you receive the report data in the file triggered the event `fileVerify`

## Documentation

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->

The [Node.js API](https://eslint.org/docs/developer-guide/nodejs-api) article will have to be updated:

> Preparation of changes is in [eslint/pull/11733](https://github.com/eslint/eslint/pull/11733)

1.  Add new option for `CLIEngine`
2.  Add a new section to describe the events, and `fileVerify` event in it

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

Perhaps someone will not consider it necessary to implement events in `CLIEngine`,
as there are a lot of ready-made modules that can wrap `executeOnFiles` and manage the search of files.

## Backwards Compatibility Analysis

<!--
    How does this change affect existing ESLint users? Will any behavior
    change for them? If so, how are you going to minimize the disruption
    to existing users?
-->

These changes do not affect the existing behavior

## Alternatives

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

As an alternative solution to the problem,
may use the callback function for the `executeOnFiles` method, that is called for each file.
I did not consider this decision, as it changes the existing API, and in my opinion is less convenient.

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

<https://github.com/eslint/eslint/pull/11733>

<https://github.com/eslint/rfcs/pull/11>

<https://github.com/eslint/rfcs/pull/4>
