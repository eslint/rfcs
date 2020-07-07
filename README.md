# ESLint RFCs

Many changes, including bug fixes and documentation improvements, can be
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

* A new ESLint command line option.
* A new feature of ESLint core.
* A refactoring of existing ESLint core functionality.
* Any breaking change.

In addition to these, the TSC may request an RFC for any other change that it deems "substantial" based on the size or scope of the request.

If you submit a pull request to implement a new feature without going
through the RFC process, it may be closed with a polite request to
submit an RFC first.

## Gathering feedback before submitting

Before submitting an RFC, open an issue in the repository where you'd like to make the change (for example, to make a change to ESLint, open an issue in the [eslint/eslint](https://github.com/eslint/eslint) repository). The purpose of opening the issue is to gather feedback from the community and the ESLint team to ensure that there is enough interest to put together an RFC.

Once the ESLint team has indicated that their is enough interest, you'll be asked to create an RFC to describe the change you'd like to make in more detail.

**Note:** Please do not submit a pull request with your prototype or proof-of-concept. Instead, include a link to your fork or branch when you submit your RFC (see next section).

## How to submit an RFC to this repo

To submit a new RFC, follow these steps:

1. [Fork](https://github.com/eslint/rfcs/fork) the RFC repo.
1. Create a directory inside of the `designs` directory. The directory name should begin with the year and include a meaningful description, such as `designs/2018-typescript-support`.
1. Copy the appropriate template file from the `templates` directory into the appropriate `designs` subdirectory (such as `designs/2018-typescript-support/README.md`). Be sure to name your file `README.md` so it is easily viewable in the GitHub interface.
1. If you want to include images in your RFC, place them in the same directory as the `README.md`.
1. Fill in the RFC. Please fill in every section in the template with as much detail as possible.
1. Submit a pull request to this repo with all of your files. This begins the approval process (detailed below).
1. RFCs that are accepted will be merged directly into this repo; RFCs that are not accepted will have their pull requests closed without merging.

## The RFC Approval Process

When an RFC is submitted, it goes through the following process:

1. **Initial commenting period (21-90 days)** - the community and ESLint team are invited to provide feedback on the proposal. During this period, you should expect to update your RFC based on the feedback provided. Very few RFCs are ready for approval without edits, so this period is important for fine-tuning ideas and building consensus. (A PR in the initial commenting period has the **Initial Commenting** label applied.) The initial commenting period must last at least 21 days and not more than 90 days to allow the team and community enough time to comment. The TSC may decide to reject the RFC without promoting it to the final commenting period.
1. **Final commenting period (7 days)** - when all feedback has been addressed, the pull request author requests a final commenting period where ESLint TSC members provide their final feedback and either approve of the pull request or state their disagreement. (A PR in the final commenting period has the **Final Commenting** label applied.) ESLint TSC members are notified through GitHub when an RFC has passed into the final commenting period.
1. **Approval and Merge** - if the TSC reaches consensus on approving the RFC, the pull request will be merged. If consensus is not reached on the pull request then the RFC will be discussed at the next TSC meeting to determine whether or not to move forward.

**Note:** If two or more RFCs attempt to address the same problem or need, the TSC will consider all of the competing RFCs together to determine which RFC to approve. Alternately, the TSC may request that competing RFCs be merged in order to create a hybrid solution.

## The RFC Lifecycle

Once an RFC is merged into this repo, then the authors may implement it and submit a pull request to the appropriate ESLint repo without opening an issue. Note that the implementation still needs to be reviewed separate from the RFC, so you should expect more feedback and iteration. 

If the RFC authors choose not to implement the RFC, then the RFC may be implemented by anyone. There is no guarantee that RFCs not implemented by their author will be implemented by the ESLint team.

Changes to the design during implementation should be reflected by updating the related RFC. The goal is to have RFCs to look back on to understand the motivation and design of shipped ESLint features.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

When a pull request has implemented an RFC, the RFC should be updated with a link
to the PR implementing it.

**Thanks to the [Ember RFC process](https://github.com/emberjs/rfcs) for the inspiration for ESLint's RFC process.**
