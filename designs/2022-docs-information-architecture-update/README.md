- Repo:  eslint/eslint
- Start Date: 2022-10-19
- RFC PR: (leave this empty, to be filled in later)
- Authors: Ben Perlmutter (@bpmutter)

# Docs Information Architecture Update

## Summary

This document contains proposed changes to the information architecture (IA) in the ESLint documentation website, <https://eslint.org/docs>.

On a high level, the proposed IA breaks the content into the following sections:

- Use ESLint in Your Project: A refactor of the current [User Guide](https://eslint.org/docs/latest/user-guide/)
- Extend ESLint: A refactor of the current [Developer Guide](https://eslint.org/docs/latest/developer-guide/), with some content moved to the sections Maintain ESLint and Contribute to ESLint
- Maintain ESLint: Expansion of current [Maintainer Guide](https://eslint.org/docs/latest/maintainer-guide/), with some content taken from the current Developer Guide.
- Contribute to ESLint: Expansion of the current [Contributing section](https://eslint.org/docs/latest/developer-guide/contributing/) of the Developer Guide.

## Motivation

The motivation of this proposed change to the ESLint documentation IA is to reorient it around the core personas of the ESLint community.

These personas are:

- **The User**: Someone who wants to use ESLint as it currently exists, including plugins. The proposed 'Use ESLint in Your Project' section addresses this persona's needs.
- **The Extender**: Someone who wants to extend the functionality of ESLint by creating a plugin, custom formatter, custom parser, sharable configuration, etc. The proposed 'Extend ESLint' section addresses this persona's needs.
- **The Community Contributor**: Someone who wants to make a small change to the core ESLint project, add a small change to the docs, or raise a Github issue. The proposed 'Contribute to ESLint' section addresses this persona's needs.
- **The Maintainer**: Someone who wants to maintain to the core ESLint project. The proposed 'Maintain ESLint' section addresses this persona's needs.

While making broad IA changes to reorient the documentation around these personas, I also propose smaller, more cosmetic changes to the documentation website's IA to better align it with technical documentation best practices, like making page titles verb phrases and avoiding gerunds.

As a result of these changes, all personas will be able to better navigate the ESLint documentation and better complete the tasks that they're using ESLint for.

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

## Detailed Design

N/A. All changes documentation related.

## Documentation

The newly proposed information architecture should consist of the following sections and pages:

### [REFACTOR SECTION]  Use ESLint in Your Project

- [RENAME SECTION] Use ESLint in Your Project
  - Rename of current section User Guide
- [NO CHANGE] Use ESLint in Your Project landing page
  - Current page: [User Guide landing page](https://eslint.org/docs/latest/user-guide/)
  - Hidden in sidebar
- [NO CHANGE] Getting Started
  - Current page: [Getting Started](https://eslint.org/docs/latest/user-guide/getting-started)
- [NO CHANGE] Core Concepts
  - Current page: [Core Concepts](https://eslint.org/docs/latest/user-guide/core-concepts)
- [RENAME PAGE] Configure
  - Current page: [Configuring](https://eslint.org/docs/latest/user-guide/configuring/)
  - [RENAME PAGES] Rename the child pages of Configuring as follows:
    - [NO CHANGE] Configuration Files (New)
    - [NO CHANGE] Configuration Files
    - [RENAME PAGE] Configuring Language Options -> Configure Language Options
    - [RENAME PAGE] Configuring Rules -> Configure Rules
    - [RENAME PAGE] Configuring Plugins -> Configure Plugins
    - [RENAME PAGE] Ignoring Code -> Ignore Code
  - Do not change the "Configuring" page and its children pages. Nicholas mentioned that there’s some major revamping to how configuration files work in ESLint, so let’s leave this subsection alone for now.
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

- [NO CHANGE] Extend ESLint landing page
  - Take relevant content from page [Developer Guide](https://eslint.org/docs/latest/developer-guide/)
  - Hidden in sidebar
- [NEW PAGE] Ways to Extend ESLint
  - Contain overview type content explaining the various ways that you can extend ESLint:
    - Create Plugins
    - Create Custom Rules
    - Create Custom Formatters
    - Create Custom Parsers
    - Shareable Configs
  - NOTE: Should be added, but not part of the initial IA project. Add in **Phase 4: "Extend ESLint" documentation update** of the documentation update project.
- [RENAME & REFACTOR PAGE] Create Plugins
  - Current page: [Working with Plugins](https://eslint.org/docs/latest/developer-guide/working-with-plugins)
  - As noted below, convert the "Processors in Plugins" section into separate page "Custom Processors"
  - Add overview-style content at the top of the page explaining that this page explains how to bundle together the various parts of a plugin, and then publish it.
- [RENAME PAGE] Custom Rules
  - Current page: [Working with Rules](https://eslint.org/docs/latest/developer-guide/working-with-rules)
  - NOTE: As fast follow, move some of the contents of this page focused on creating a core rule to the "Contribute to ESLint" section. Exact details TBD.
- [RENAME PAGE] Custom Formatters
  - Current page: [Working with Custom Formatters](https://eslint.org/docs/latest/developer-guide/working-with-custom-formatters)
- [RENAME PAGE] Custom Parsers
  - Current page: [Working with Custom Parsers](https://eslint.org/docs/latest/developer-guide/working-with-custom-parsers)
  - Rename of "Working with Custom Parsers"
- [NEW PAGE] Custom Processors
  - Take most of content from the section [Processors in Plugins section on Work with Plugins Page](https://eslint.org/docs/latest/developer-guide/working-with-plugins#processors-in-plugins)
- [RENAME PAGE] Share Configurations
  - Current page: [Shareable Configs](https://eslint.org/docs/latest/developer-guide/shareable-configs)
- [NO CHANGE TO PAGE] Node.js API Reference
  - Current page: [Node.js API](https://eslint.org/docs/latest/developer-guide/nodejs-api)

### [NEW SECTION] Contribute to ESLint

New top-level section of the docs based on the [Contributing](https://eslint.org/docs/latest/developer-guide/contributing/) section. Breaking this out as a separate section because I think the "casual community contributor" is a fairly separate persona from the "extender" and "maintainer", so it should be treated as such in the information architecture.

- [MOVE & RENAME PAGE] Contribute to ESLint section landing page
  - Current page: [Contributing](https://eslint.org/docs/latest/developer-guide/contributing/)
  - Take relevant content from page [Developer Guide](https://eslint.org/docs/latest/developer-guide/)
  - Hidden in sidebar
- [NEW PAGE] Code of Conduct
  - Takes current content from the [Read the Code of Conduct section on the Contributing page](https://eslint.org/docs/latest/developer-guide/contributing/#read-the-code-of-conduct). Worth breaking out into its own page so content not only on the hidden landing page.
- [MOVE & RENAME PAGE] Report Bugs
  - Current page: [Bug Reporting](https://eslint.org/docs/latest/developer-guide/contributing/reporting-bugs)
- [MOVE & RENAME PAGE] Propose a New Rule
  - Current page: [Proposing a New Rule](https://eslint.org/docs/latest/developer-guide/contributing/new-rules)
- [MOVE & RENAME PAGE] Propose a Rule Change
  - Current page: [Proposing a Rule Change](https://eslint.org/docs/latest/developer-guide/contributing/#:~:text=Proposing%20a-,Rule%20Change,-Want%20to%20make)
- [MOVE & RENAME PAGE] Request a Change
  - Current page: [Change Requests](https://eslint.org/docs/latest/developer-guide/contributing/changes)
- [MOVE PAGE] Architecture
  - Current page: [Architecture](https://eslint.org/docs/latest/developer-guide/architecture/)
  - Note: in a previous conversation with Nicholas, he mentioned to me that this page is pretty out-of date.
  - There should probably be a future unit of work to refactor this page.
  - However, it’s not included in the information architecture updates.
- [MOVE & REFACTOR PAGE] Set up Development Environment
  - Consolidate the pages [Getting the Source Code](https://eslint.org/docs/latest/developer-guide/source-code) and [Set up a Development Environment](https://eslint.org/docs/latest/developer-guide/development-environment)
  - Getting the source code is basically just a step in the process of setting up the development environment. Therefore, these two pages should be consolidated.
  - Also the Directory Structure section on the [Getting the Source Code](https://eslint.org/docs/latest/developer-guide/source-code#directory-structure) page should be moved toward the bottom of the new Set up a Development Environment page.
- [MOVE PAGE] Run the Tests
  - Current page: [Unit Tests](https://eslint.org/docs/latest/developer-guide/unit-tests)
- [MOVE & RENAME PAGE] Work on Issues
  - Current page: [Working on Issues](https://eslint.org/docs/latest/developer-guide/contributing/working-on-issues)
- [MOVE, EXPAND & RENAME PAGE] Submit a Pull Request
  - Current page: [Pull Requests](https://eslint.org/docs/latest/developer-guide/contributing/pull-requests)
  - Add the content from [Signing the CLA section](https://eslint.org/docs/latest/developer-guide/contributing/#signing-the-cla) from the Contributing overview page.
  Expand the content in [Step 7: send the pull request](https://eslint.org/docs/latest/developer-guide/contributing/pull-requests#step-7-send-the-pull-request) with this content.
- [NEW PAGE] Report Security Vulnerabilities
  - Takes current content from the [Reporting a Security Vulnerability section on the Contributing page](https://eslint.org/docs/latest/developer-guide/contributing/#reporting-a-security-vulnerability). Worth breaking out into its own page so that content not only on the hidden landing page and for discoverability and SEO for an important subject.
  - NOTE: Should be added, but not part of the initial IA project. Probably can be done as a one-off task or group in with one of the project phases.
- [MOVE PAGE] Governance
  - Current page: [Governance](https://eslint.org/docs/latest/maintainer-guide/governance)

### [REFACTOR SECTION] Maintain ESLint

- [NO CHANGE] Maintain ESLint section landing page
  - Current page: [Maintainer Guide](https://eslint.org/docs/latest/maintainer-guide)
  - Hidden in sidebar
- [MOVE & RENAME PAGE] Manage Issues
  - Current page: [Managing Issues](https://eslint.org/docs/latest/maintainer-guide/issues)
- [MOVE & RENAME PAGE] Review Pull Requests
  - Current page: [Reviewing Pull Requests](https://eslint.org/docs/latest/maintainer-guide/pullrequests)
- [MOVE & RENAME PAGE] Manage Releases
  - Current page: [Managing Releases](https://eslint.org/docs/latest/maintainer-guide/releases)
- [NEW PAGE(S)] Any other technical pages for maintainers
  - I don’t have any thoughts on what these should be right now. I would defer to the core maintainers on what they think other maintainers should know.
  - I think that these additional technical pages would make sense to include in **Phase 5: "Maintain ESLint" documentation updates** of [the documentation update project](https://github.com/eslint/eslint/issues/16365).

### New Sidebar (summary)

With the above changes, the left-hand side navigation will look like:

- Use ESLint in Your Project (landing page)
  - Getting Started
  - Core Concepts
  - Configure
    - Configuration Files (New)
    - Configuration Files
    - Configure Language Options
    - Configure Rules
    - Configure Plugins
    - Ignore Code
  - Command Line Interface Reference
  - Rules Reference
  - Formatters Reference
  - Integrations
  - Migrate to v8.x
- Extend ESLint (landing page)
  - Create Plugins
  - Custom Rules
  - Custom Formatters
  - Custom Parsers
  - Custom Processors
  - Share Configurations
  - Node.js API Reference
- Contribute to ESLint (landing page)
  - Code of Conduct
  - Report Bugs
  - Propose a New Rule
  - Propose a Rule Change
  - Request a Change
  - Architecture
  - Set up a Development Environment
  - Run the Tests
  - Work on Issues
  - Submit a Pull Request
  - Report Security Vulnerabilities
  - Governance
- Maintain ESLint (landing page)
  - Manage Issues
  - Review Pull Requests
  - Manage Releases

### HTTP Redirects

The proposed changes would include the following HTTP redirects from current content locations to the locations proposed in the above outline.

| Current URL | Proposed URL |
| :---: | :---: |
| <https://eslint.org/docs/latest/user-guide/> | <https://eslint.org/docs/latest/use/> |
| <https://eslint.org/docs/latest/user-guide/core-concepts> | <https://eslint.org/docs/latest/use/core-concepts> |
| <https://eslint.org/docs/latest/user-guide/configuring/> | <https://eslint.org/docs/latest/use/configure/> |
| <https://eslint.org/docs/latest/user-guide/configuring/configuration-files-new> | <https://eslint.org/docs/latest/use/configure/configuration-files-new> |
| <https://eslint.org/docs/latest/user-guide/configuring/configuration-files> | <https://eslint.org/docs/latest/use/configure/configuration-files> |
| <https://eslint.org/docs/latest/user-guide/configuring/language-options> | <https://eslint.org/docs/latest/use/configure/language-options> |
| <https://eslint.org/docs/latest/user-guide/configuring/rules> | <https://eslint.org/docs/latest/use/configure/rules> |
| <https://eslint.org/docs/latest/user-guide/configuring/plugins> | <https://eslint.org/docs/latest/use/configure/plugins> |
| <https://eslint.org/docs/latest/user-guide/configuring/ignoring-code> | <https://eslint.org/docs/latest/use/configure/ignore> |
| <https://eslint.org/docs/latest/user-guide/command-line-interface> | <https://eslint.org/docs/latest/use/command-line-interface> |
| <https://eslint.org/docs/latest/user-guide/formatters/> | <https://eslint.org/docs/latest/use/formatters/> |
| <https://eslint.org/docs/latest/user-guide/integrations> | <https://eslint.org/docs/latest/use/integrations> |
| <https://eslint.org/docs/latest/user-guide/migrating-to-8.0.0> | <https://eslint.org/docs/latest/use/migrate-to-8.0.0> |
| <https://eslint.org/docs/latest/user-guide/**> (any other pages not in sidebar) | <https://eslint.org/docs/latest/use/**> |
| <https://eslint.org/docs/latest/developer-guide/architecture/> | <https://eslint.org/docs/latest/contribute/architecture/> |
| <https://eslint.org/docs/latest/developer-guide/source-code> |  <https://eslint.org/docs/latest/contribute/development-environment> |
| <https://eslint.org/docs/latest/developer-guide/development-environment> |  <https://eslint.org/docs/latest/contribute/development-environment> |
| <https://eslint.org/docs/latest/developer-guide/unit-tests> | <https://eslint.org/docs/latest/contribute/tests> |
| <https://eslint.org/docs/latest/developer-guide/working-with-rules> | <https://eslint.org/docs/latest/extend/custom-rules> |
| <https://eslint.org/docs/latest/developer-guide/working-with-plugins> | <https://eslint.org/docs/latest/extend/plugins> |
| <https://eslint.org/docs/latest/developer-guide/working-with-custom-formatters> | <https://eslint.org/docs/latest/extend/custom-formatters> |
| <https://eslint.org/docs/latest/developer-guide/working-with-custom-parsers> | <https://eslint.org/docs/latest/extend/custom-parsers> |
| <https://eslint.org/docs/latest/developer-guide/nodejs-api> | <https://eslint.org/docs/latest/integrate/nodejs-api> |
| <https://eslint.org/docs/latest/developer-guide/contributing/> | <https://eslint.org/docs/latest/contribute/> |
| <https://eslint.org/docs/latest/developer-guide/contributing/reporting-bugs> | <https://eslint.org/docs/latest/contribute/report-bugs> |
| <https://eslint.org/docs/latest/developer-guide/contributing/new-rules> | <https://eslint.org/docs/latest/contribute/propose-new-rule> |
| <https://eslint.org/docs/latest/developer-guide/contributing/rule-changes> | <https://eslint.org/docs/latest/contribute/propose-rule-change> |
| <https://eslint.org/docs/latest/developer-guide/contributing/changes> | <https://eslint.org/docs/latest/contribute/request-change> |
| <https://eslint.org/docs/latest/developer-guide/contributing/working-on-issues> | <https://eslint.org/docs/latest/contribute/work-on-issue> |
| <https://eslint.org/docs/latest/developer-guide/contributing/pull-requests> | <https://eslint.org/docs/latest/contribute/pull-requests> |
| <https://eslint.org/docs/latest/developer-guide/**> (any other pages not in sidebar) | <https://eslint.org/docs/latest/extend/**> |
| <https://eslint.org/docs/latest/maintainer-guide/> | <https://eslint.org/docs/latest/maintain/> |
| <https://eslint.org/docs/latest/maintainer-guide/issues> | <https://eslint.org/docs/latest/maintain/manage-issues> |
| <https://eslint.org/docs/latest/maintainer-guide/pullrequests> |  <https://eslint.org/docs/latest/maintain/review-pull-requests> |
| <https://eslint.org/docs/latest/maintainer-guide/releases> | <https://eslint.org/docs/latest/maintain/manage-releases> |
| <https://eslint.org/docs/latest/maintainer-guide/governance> | <https://eslint.org/docs/latest/contribute/governance> |
| <https://eslint.org/docs/latest/maintainer-guide/**> (any other pages not in sidebar) | <https://eslint.org/docs/latest/maintain/**> |

## Drawbacks

Potential drawbacks of this documentation update include:

- Breaking user experience from the current documentation
- Users potentially do not find the new information architecture more clear
- Opportunity cost. Time spent on the information architecture updates could be spent on more impactful documentation tasks.

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

- Do not change the IA. Only update documentation within the existing IA as part of the [documentation update project](https://github.com/eslint/eslint/issues/16365).

<!--
    What other designs did you consider? Why did you decide against those?

    This section should also include prior art, such as whether similar
    projects have already implemented a similar feature.
-->

## Open Questions

1. Is there a way to automatically change all the internal links that'll be broken by these updates?
   - A: [No (link with explanation)](https://github.com/eslint/rfcs/pull/97#discussion_r1003895871)
1. Can you create a side navigation folder that is not also a page?
   - A: [No (link with explanation)](https://github.com/eslint/rfcs/pull/97#discussion_r1003896032)
1. How to make section landing pages navigable from the side table of contents? (For example, the proposed page "Use ESLint in Your Project landing page")
   - [Cannot for top level landing pages for design decision reasons (link with explanation)](https://github.com/eslint/rfcs/pull/97#discussion_r1007459580)

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
- I may need some help with updating the side navigation.
- Investigate if there's a way to automate changing the internal link paths.

<!--
    This section is optional.

    Are you able to implement this RFC on your own? If not, what kind
    of help would you need from the team?
-->

## Frequently Asked Questions

### Why Rename Pages with Active Verb Phrases?

This outline proposes renaming various titles to use active verb phrases instead of the current gerund-based page name structure (ex. changing a page title from "Managing Issues" → "Manage Issues"). This is following a general technical writing best practice of avoiding gerunds in titles. For more information, refer to [this blog post about the subject](https://www.google.com/url?q=https://www.writethedocs.org/blog/newsletter-october-2022/%23gerunds-in-headings&sa=D&source=docs&ust=1665539063991039&usg=AOvVaw2HTT7RUQYPCcUenwVv0-Zt).

### Will This Information Architecture Update Include Creating New Content?

No. Additional content is to be created in Phases 3+ of the [documentation update project](https://github.com/eslint/eslint/issues/16365). This includes documentation for the [newly proposed "Integrator" persona](https://github.com/eslint/rfcs/pull/97#discussion_r1012138932),
to be completed in phase 6 of the documentation update project.

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
