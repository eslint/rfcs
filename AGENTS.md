# Contributor Guidance

## RFC development and review

- RFCs live in `designs/`. Create a directory named with the year and a concise description, such as `designs/2026-example-feature`.
- Copy `templates/design.md` into that directory as `README.md`. Complete the template sections with enough detail to explain the proposal; put any accompanying images in the same directory.
- Update the RFC in response to review feedback. The README describes an initial commenting period followed by a final commenting period before approval.
- When an RFC is implemented, update it with a link to the implementation pull request.

## Validation

This repository contains RFC Markdown and templates, not an application test suite. No automated test or build command is defined here. For documentation changes, review the rendered Markdown, verify links and image paths, and check that RFCs follow `templates/design.md`.

## Pull requests and commits

Pull request titles and commit messages must follow the Conventional Commits format (for example, `docs: clarify RFC guidance`).

### Writing Pull Request Descriptions

Follow `.github/PULL_REQUEST_TEMPLATE.md`: include a concise summary, preferably pasted from the proposal, and add relevant discussion links under **Related Issues** when applicable. The Related Issues section is optional.

### RFC Writing Guidelines

When submitting a new RFC, follow these guidelines:

- Follow the template in `templates/design.md`. Ensure each section is filled out.
- Use everyday words and avoid jargon. When a technical term is necessary (for example, "code path analysis" or "scope manager"), add a short phrase explaining what it does the first time it appears.
- Assume the reader does not have deep knowledge of the work to be performed. Your explanation must not only explain the design but also teach it to the reader.
- When explaining a problem, give concrete examples; when explaining a solution, tie it back to the specific problems they solve.
- Prefer short paragraphs to nested bullet lists. Use bullets only for things that really are a list, such as several independent behavior changes.
- When explaining the detailed design, refer to specific files, classes, functions, etc.

## AI Disclosure Requirement

ESLint's [AI Usage Policy](docs/src/contribute/ai-policy.md) requires that AI-assisted contributions be disclosed. Whenever you (an AI) create content on GitHub for this project, you **must** include a disclosure, using the name of the model actually producing the content (for example, `Claude Opus 5`):

- **Issues** — add a bold sentence to the top of body:

    **This issue was created with AI (Claude Opus 5).**

- **Pull requests** — add a bold sentence to the top of the description:

    **This pull request was created with AI (Claude Opus 5).**

- **Comments, review comments, and review summaries** — add a final paragraph in bold:

    **Created by AI (Claude Opus 5).**

This applies to every AI-authored submission without exception, including follow-up comments on an existing thread. Note two related policy points: AI-generated PRs are only considered for issues labeled `accepted`, and maintainer feedback is expected to be answered by a human rather than fed back into an AI.
