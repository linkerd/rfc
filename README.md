# Linkerd RFCs

Many changes, including bug fixes and documentation improvements can be implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a bit of a design process and produce a consensus among the Linkerd community.

The "RFC" (request for comments) process is intended to provide a consistent and controlled path for contributions to enter the project, so that all stakeholders can be confident about the project direction and maintainability of the codebase.

## When you need to follow this process

[when you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to Linkerd2 (Linkerd2-proxy, Linkerd Control Plane) or the RFC process itself. What constitutes an "substantial" is a foundational concern for the Linkerd project.

Similarly, any technical effort (refactoring, major architectural change) that will impact a large section of the development community should also be communicated widely. The Linkerd RFC process is suited for this even if it will have zero impact on the typical user or operator.

## Before creating an RFC

[before creating an rfc]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality proposals, proposals for previously-rejected features, or those that don't fit into the near-term roadmap, may be quickly rejected, which can be demotivating for the unprepared contributor. Laying some groundwork ahead of the RFC can make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is generally a good idea to pursue feedback from other project developers beforehand, to ascertain that the RFC may be desirable; having a consistent impact on the project requires concerted effort toward consensus-building.

The most common preparations for writing and submitting an RFC include talking the idea over on our [Linkerd Slack, #contributors](https://slack.linkerd.io), or discussing the topic on our [CNCF developer mailing list](https://lists.cncf.io/g/cncf-linkerd-dev). You may file issues on this repo for discussion, but these are not actively looked at by the teams.

As a rule of thumb, receiving encouraging feedback from long-standing project developers is a good indication that the RFC is worth pursuing.

## What the process is

[what the process is]: #what-the-process-is

In short, to get a major feature added to Linkerd, one must first get the RFC merged into the RFC repository as a markdown file. At that point the RFC is "active" and may be implemented with the goal of eventual inclusion into Linkerd.

* Fork the [RFC repository](https://github.com/linkerd/rfc) * Copy `rfc-template.md` to `active/my-contribution.md` (where "my-contribution" is

 descriptive.).

* Fill in the RFC. Put care into the details: RFCs that do not present convincing motivation, demonstrate lack of understanding of the design's impact, or are disingenuous about the drawbacks or alternatives tend to be poorly-received.

* Submit a pull request. As a pull request the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.

* Each pull request will be labeled with the most relevant reviewer, who will lead its triage.

* Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments. Feel free to reach out to the RFC assignee in particular to get help identifying stakeholders and obstacles.

* The team will discuss the RFC pull request, as much as possible in the comment thread of the pull request itself. Offline discussion will be summarized on the pull request comment thread.

* RFCs rarely go through this process unchanged, especially as alternatives and drawbacks are shown. You can make edits, big and small, to the RFC to clarify or change the design, but make changes as new commits to the pull request, and leave a comment on the pull request explaining your changes. Specifically, do not squash or rebase commits after they are visible on the pull request.

## The RFC life-cycle

[the rfc life-cycle]: #the-rfc-life-cycle

Once an RFC becomes "active" then authors may implement it and submit the feature as a series of pull requests to the Linkerd2 repo. Being "active" is not a rubber stamp, and in particular still does not mean the feature will ultimately be merged; it does mean that in principle all the major stakeholders have agreed to the feature and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is "active" implies nothing about what priority is assigned to its implementation, nor does it imply anything about whether a maintainer has been assigned the task of implementing the feature. While it is not _necessary_ that the author of the RFC also write the implementation, it is by far the most effective way to see an RFC through to completion: authors should not expect that other project developers will take on responsibility for implementing their accepted feature.

Modifications to "active" RFCs can be done in follow-up pull requests. We strive to write each RFC in a manner that it will reflect the final design of the feature; but the nature of the process means that we cannot expect every merged RFC to actually reflect what the end result will be at the time of the next major release.

In general, once accepted, RFCs should not be substantially changed. Only very minor changes should be submitted as amendments. More substantial changes should be new RFCs, with a note added to the original RFC. Exactly what counts as a "very minor change" is up to the team to decide.

## Implementing an RFC

[implementing an rfc]: #implementing-an-rfc

Some accepted RFCs represent vital features that need to be implemented right away. Other accepted RFCs can represent features that can wait until some arbitrary developer feels like doing the work. Every accepted RFC has an associated issue tracking its implementation in the Linkerd2 repository; thus that associated issue can be assigned a priority via the triage process that the team uses for all issues in the Linkerd2 repository.

The author of an RFC is not obligated to implement it. Of course, the RFC author (like any other developer) is welcome to post an implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an "active" RFC, but cannot determine if someone else is already working on it, feel free to ask (e.g.by leaving a comment on the associated issue).

### Help this is all too informal!

[help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the present circumstances. As usual, we are trying to let the process be driven by consensus and community norms, not impose more structure than necessary.

## License

[license]: #license

This repository is currently licensed under:

* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0) * https://github.com/linkerd/linkerd2/blob/master/LICENSE

### Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be licensed as above, without any additional terms or conditions.

