# RFCs

The RFC (request for comments) process is intended is intended to provide a consistent and controlled path for changes to the Open Data Contract Standard (ODCS) so that all stakeholders can be confident about the direction of the project.

## Table of Contents
[Table of Contents]: #table-of-contents

  - [Before creating an RFC]
  - [The process]

## Before creating an RFC
[Before creating an RFC]: #before-creating-an-rfc

Although there is no single way to prepare for submitting an RFC, it is generally a good idea to pursue feedback from other project developer beforehand, to ascertain that the RFC may be desirable; having a consistent impact on the project requires concerted effort toward consensus-building.

The most common preparations for writing and submitting an RFC include talking the idea over on our Slack channel and using a Google Doc to create the draft.

## The process

- Fork the RFC repo [RFC repository]
  - Copy `0000-template.md` to `xxxx-my-feature.md`,` where "my-feature" is descriptive, and the number is the next in sequence.
  - Fill in the RFC.
  - Submit a pull request. As a pull request the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.
  - Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments. Feel free to reach out to the RFC assignee in particular to get help identifying stakeholders and obstacles.
  - RFCs rarely go through this process unchanged, especially as alternatives and drawbacks are shown. You can make edits, big and small, to the RFC to clarify or change the design, but make changes as new commits to the pull request, and leave a comment on the pull request explaining your changes. Specifically, do not squash or rebase commits after they are visible on the pull request.
  - When ready, RFCs will be voted on by the TSC at its monthly meeting.
  - Update the RFC with the decision taken by the TSC.
  - Merge in the RFC.

## Acknowledgments

* [Rust RFCs](https://github.com/rust-lang/rfcs) - Rust RFC process.
* [ghostinthewires](https://github.com/ghostinthewires/Rfcs-Template/tree/master) - A RFC process template.
