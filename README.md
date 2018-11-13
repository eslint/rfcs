# ESLint RFCs

Many changes, including bug fixes and documentation, improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

However, some changes are "substantial", and we ask that these be put
through a bit of a design process and produce a consensus among the ESLint Technical Steering Committee (TSC).

The "RFC" (request for comments) process is intended to provide a
consistent and controlled path for new features to enter the project.

## When you need to follow this process

You need to follow this process if you intend to make "substantial"
changes to any part of the ESLint project or its documentation. What constitutes a
"substantial" change is evolving based on community norms, but may
include the following.

    - A new ESLint command line option.
    - A new feature of ESLint core.
    - A refactoring of existing ESLint core functionality.
    - Any breaking change.

In addition to these, the TSC may request an RFC for any other change that it deems "substantial" based on the size or scope of the request.

If you submit a pull request to implement a new feature without going
through the RFC process, it may be closed with a polite request to
submit an RFC first.

## Gathering feedback before submitting

It's often helpful to get feedback on your concept before diving into the
level of API design detail required for an RFC. A good first step is to contact the [mailing list](https://groups.google.com/group/eslint) or [chatroom](https://gitter.im/eslint/eslint) to get thoughts from the community and the ESLint team before proceeding.

## How to submit an RFC

To submit a new RFC, follow these steps:

1. [Fork](https://github.com/eslint/rfcs/fork) the RFC repo.
1. Copy the appropriate template file from the `templates` directory into the `designs` directory.
1. Rename the file to begin with the current year and include a meaningful description, such as `2018-typescript-support.md`.
1. If you want to include images in your RFC, place them in the `images` directory and prefix the filename with the same filename as the Markdown file (so `2018-typescript-support-figure-1.png`, for example).
1. Fill in the RFC. Please fill in every section in the template with as much detail as possible.
1. Submit a pull request to this repo with all of your files.
1. You will receive feedback both from the ESLint community and from the ESLint team. You should be prepared to update your RFC based on this feedback. The goal is to build consensus on the best way to implement the suggested change.
1. When all feedback has been incorporated, the ESLint TSC will determine whether or not to accept the RFC.
1. RFCs that are accepted will be merged directly into this repo; RFCs that are not accepted will have their pull requests closed without merging.

## The RFC life-cycle

Once an RFC is merged into this repo, then the authors may implement it and submit a pull request to the appropriate ESLint repo without opening an issue. Note that the implementation still needs to be reviewed separate from the RFC, so you should expect more feedback and iteration. 

If the RFC authors choose not to implement the RFC, then the RFC may be implemented by anyone. There is no guarantee that RFCs not implemented by their author will be implemented by the ESLint team.

Changes to the design during implementation should be reflected by updating the related RFC. The goal is to have RFCs to look back on to understand the motivation and design of shipped ESLint features.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

**Thanks to the [Ember RFC process](https://github.com/emberjs/rfcs) for the inspiration for ESLint's RFC process.**