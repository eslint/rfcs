- Repo:  eslint/eslint
- Start Date: 2022-10-19
- RFC PR: (leave this empty, to be filled in later)
- Authors: Ben Perlmutter (@bpmutter)

# Docs Information Architecture Update

## Summary

This document contains proposed changes to the information architecture in the ESLint documentation website, <https://eslint.org/docs>.

On a high level, the proposed information architecture breaks the content into the following sections:

- Use ESLint in Your Project: A refactor of the current [User Guide](https://eslint.org/docs/latest/user-guide/)
- Extend ESLint: A refactor of the current [Developer Guide](https://eslint.org/docs/latest/developer-guide/), with some content moved to the section Maintain ESLint and Community Contributions
- Maintain ESLint: Expansion of current [Maintainer Guide](https://eslint.org/docs/latest/maintainer-guide/), with some content taken from the current ‘Developer Guide’.
- Community Contributions: Expansion of the current [Contributing section](https://eslint.org/docs/latest/developer-guide/contributing/) of the Developer Guide.

## Motivation

TODO: need to think on this one more than 'cuz nicholas said so'
<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

## Detailed Design

TODO: note abt how this isn't bulk of RFC. more like technical considerations
<!--
   This is the bulk of the RFC.

   Explain the design with enough detail that someone familiar with ESLint
   can implement it by reading this document. Please get into specifics
   of your approach, corner cases, and examples of how the change will be
   used. Be sure to define any new terms in this section.
-->

## Documentation

The newly proposed information architecture should consist of the following sections and pages:

<!--
    How will this RFC be documented? Does it need a formal announcement
    on the ESLint blog to explain the motivation?
-->
### [REFACTOR SECTION]  Use ESLint in Your Project

- [RENAME SECTION] Use ESLint in Your Project
  - Rename of current section User Guide
- [NO-PAGE CHANGE] Getting Started
  - Current page: [Getting Started](https://eslint.org/docs/latest/user-guide/getting-started)
  - NOTE: I think there’s further work that can be done to improve the getting started experience, but do not want to touch the subject as part of the information architecture changes project. Investigate further in **Phase 3: “Use ESLint in Your Project” documentation update** of the documentation update project.
- [DO NOTHING TO SUBSECTION] Configuring
  - Current page: [Configuring](https://eslint.org/docs/latest/user-guide/configuring/)
  - Do not change the “Configuring” page and its children pages. Nicholas mentioned that there’s some major revamping to how configuration files work in ESLint, so let’s leave this subsection alone for now.
- [RENAME PAGE] Command Line Interface Reference
  - Current page: [Command Line Interface](https://eslint.org/docs/latest/user-guide/command-line-interface)
- [RENAME PAGE] Rules Reference
  - Current page: [Rules](https://eslint.org/docs/latest/rules/)
- [RENAME PAGE] Formatters Reference
  - Current page: [Formatters](https://eslint.org/docs/latest/user-guide/formatters/)
- [NO CHANGE] Integrations
  - Current page: [Integrations](https://eslint.org/docs/latest/user-guide/integrations)
- [RENAME PAGE] Migrate to v8.x
  - Current page: [Migrating to v8.0.0](https://eslint.org/docs/latest/user-guide/migrating-to-8.0.0)

### [REFACTOR SECTION]  Extend ESLint

- [NEW PAGE] Overview
  - NOTE: Should be added, but not part of the initial IA project. Add in **Phase 4: “Extend ESLint” documentation update** of the documentation update project.
  - Contain overview type content explaining the various ways that you can extend ESLint:
    - Create Plugins
    - Create Custom Rules
    - Create Custom Formatters
    - Create Custom Parsers
    - Shareable Configs
  - Alternatively, repurpose the current section landing page[Developer Guide](https://eslint.org/docs/latest/developer-guide/) to be exposed in the side navigation and with the above content.
- [RENAME & REFACTOR PAGE] Create Plugins
  - Current page: [Working with Plugins](https://eslint.org/docs/latest/developer-guide/working-with-plugins)
  - As noted below, convert the “Processors in Plugins” section into separate page “Custom Processors
  - Add overview-style content at the top of the page explaining that this page explains how to bundle together the various parts of a plugin, and then publish it.
- [RENAME PAGE] Custom Rules
  - Current page: [Working with Rules](https://eslint.org/docs/latest/developer-guide/working-with-rules)
- [RENAME PAGE] Custom Formatters
  - Current page: [Working with Custom Formatters](https://eslint.org/docs/latest/developer-guide/working-with-custom-formatters)
- [RENAME PAGE] Custom Parsers
  - Current page: [Working with Custom Parsers](https://eslint.org/docs/latest/developer-guide/working-with-custom-parsers)
  - Rename of “Working with Custom Parsers”
- [NEW PAGE] Custom Processors
  - Take most of content from the section [Processors in Plugins section on Work with Plugins Page](https://eslint.org/docs/latest/developer-guide/working-with-plugins#processors-in-plugins)
- [RENAME PAGE] Share Configurations
  - Current page: [Shareable Configs](https://eslint.org/docs/latest/developer-guide/shareable-configs)
- [NO CHANGE TO PAGE] Node.js API
  - Current page: [Node.js API](https://eslint.org/docs/latest/developer-guide/nodejs-api)

### [REFACTOR SECTION] Maintain ESLint

- [NEW PAGE] Overview
  - NOTE: Should be added, but not part of the initial IA project. Add in **Phase 5: “Maintain ESLint” documentation update** of the documentation update project.
  - Alternatively, repurpose the current section landing page[Maintainer Guide](https://eslint.org/docs/latest/maintainer-guide) to be exposed in the side navigation and with the above content.
- [MOVE PAGE] Architecture
  - Current page: [Architecture](https://eslint.org/docs/latest/developer-guide/architecture/)
  - Note: in a previous conversation with Nicholas, he mentioned to me that this page is pretty out-of date.
  - There should probably be a future unit of work to refactor this page.
  - However, it’s not included in the information architecture updates.
- [MOVE & REFACTOR PAGE] Set up Development Environment
  - Consolidate the pages [Getting the Source Code](https://eslint.org/docs/latest/developer-guide/source-code) and [Set up a Development Environment](https://eslint.org/docs/latest/developer-guide/development-environment)
  - Getting the source code is basically just a step in the process of setting up the development environment. Therefore, these two pages should be consolidated
  - Also the Directory Structure section on the [Getting the Source Code](https://eslint.org/docs/latest/developer-guide/source-code#directory-structure) page should be moved toward the bottom of the new Set up a Development Environment page.
- [NEW PAGE(S)] Any other technical pages for maintainers
  - I don’t have any thoughts on what these should be right now. I would defer to the core maintainers on what they think other maintainers should know.
  - I think that these additional technical pages would make sense to include in **Phase 5: “Maintain ESLint” documentation updates**of [the documentation update project](https://github.com/eslint/eslint/issues/16365).  
  - NOTE: Maybe should be added, but not part of the initial IA project.
- [MOVE PAGE] Run the Tests
  - Current page: [Unit Tests](https://eslint.org/docs/latest/developer-guide/unit-tests)
- [NEW SECTION] Project Management
  - [MOVE & RENAME PAGE] Manage Issues
    - Current page: [Managing Issues](https://eslint.org/docs/latest/maintainer-guide/issues)
  - [MOVE & RENAME PAGE] Review Pull Requests
    - Current page: [Reviewing Pull Requests](https://eslint.org/docs/latest/maintainer-guide/pullrequests)
  - [MOVE & RENAME PAGE] Manage Releases
    - Current page: [Managing Releases](https://eslint.org/docs/latest/maintainer-guide/releases)
  - [MOVE PAGE] Governance
    - Current page: [Governance](https://eslint.org/docs/latest/maintainer-guide/governance)

### [NEW SECTION] Community Contributions

New top-level section of the docs based on the [Contributing](https://eslint.org/docs/latest/developer-guide/contributing/) section. Breaking this out as a separate section because I think the “casual community contributor” is a fairly separate persona from the “extender” and “maintainer”, so it should be treated as such in the information architecture.

- [MOVE & RENAME PAGE] section landing page
  - Current page: [Contributing](https://eslint.org/docs/latest/developer-guide/contributing/)
- [MOVE & RENAME PAGE] Report Bugs
  - Current page: [Bug Reporting](https://eslint.org/docs/latest/developer-guide/contributing/reporting-bugs)
- [MOVE & RENAME PAGE] Propose a New Rule
  - Current page: [Proposing a New Rule](https://eslint.org/docs/latest/developer-guide/contributing/new-rules)
- [MOVE & RENAME PAGE] Propose a Rule Change
  - Current page: [Proposing a Rule Change](https://eslint.org/docs/latest/developer-guide/contributing/#:~:text=Proposing%20a-,Rule%20Change,-Want%20to%20make)
- [MOVE & RENAME PAGE] Request a Change
  - Current page: [Change Requests](https://eslint.org/docs/latest/developer-guide/contributing/changes)
- [MOVE & RENAME PAGE] Work on Issues
  - Current page: [Working on Issues](https://eslint.org/docs/latest/developer-guide/contributing/working-on-issues)
- [NEW PAGE] Report Security Vulnerabilities
  - Tales current content from the [Reporting a Security Vulnerability section on the Contributing page](https://eslint.org/docs/latest/developer-guide/contributing/#reporting-a-security-vulnerability). Worth breaking out into its own page for discoverability and SEO for an important subject.
  - NOTE: Should be added, but not part of the initial IA project. Probably can be done as a one-off task or group in with one of the project phases.
- [MOVE & RENAME PAGE] Submit a Pull Request
  - Current page: [Pull Requests](https://eslint.org/docs/latest/developer-guide/contributing/pull-requests)

## Drawbacks

TODO: breaking user experience
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

HTTP redirects from existing content URL paths to the new content URL paths must be included as part of this project.

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

1. Is it possible to have an always-opened folder named ‘Reference’ in which we put all these pages under? I.e similar to the “User Guide” folder, but as a subsection of the “User Guide” folder.
2. TODO: more about if containers can not be pages as well.
<!--
    This section is optional, but is suggested for a first draft.

    What parts of this proposal are you unclear about? What do you
    need to know before you can finalize this RFC?

    List the questions that you'd like reviewers to focus on. When
    you've received the answers and updated the design to reflect them,
    you can remove this section.
-->

## Help Needed

I would likely need help with the following tasks:

- Technical implementation details of how to handle redirects
- Investigation of if there's a way to automate changing the internal link paths

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

## Frequently Asked Questions

### Why Rename Pages with Active Verb Phrases?

This outline proposes renaming various titles to use active verb phrases instead of the current  gerund-based page name structure (ex. changing a page title from “Managing Issues” → “Manage Issues”). This is following a general technical writing best practice of avoiding gerunds in titles. For more information, refer to [this blog post about the subject](https://www.google.com/url?q=https://www.writethedocs.org/blog/newsletter-october-2022/%23gerunds-in-headings&sa=D&source=docs&ust=1665539063991039&usg=AOvVaw2HTT7RUQYPCcUenwVv0-Zt).

### Will This Information Architecture Update Include Creating New Content?

No. Additional content is to be created in Phases 3+ of the [documentation update project](https://github.com/eslint/eslint/issues/16365).

<!--
    This section is optional but suggested.

    Try to anticipate points of clarification that might be needed by
    the people reviewing this RFC. Include those questions and answers
    in this section.
-->

## Related Discussions

This work corresponds to **Phase 2: High-level information architecture update** of the [documentation update project](https://github.com/eslint/eslint/issues/16365).
<!--
    This section is optional but suggested.

    If there is an issue, pull request, or other URL that provides useful
    context for this proposal, please include those links here.
-->
