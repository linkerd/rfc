- Automated Releases
- @kleimkuhler, @alpeb
- 2020-02-17
- 2020-03-03
- todo: RFC PR: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/0000)
- Linkerd Issue: [linkerd/linkerd2#3021](https://github.com/linkerd/linkerd2/issues/3021)
- Reviewers: @grampelberg

# Summary

[summary]: #summary

The goal for this project is to make the Linkerd2 release process as automated
as possible. The process currently requires that the responsible member take
multiple manual steps in order for the release artifacts to be properly built,
tested, and updated for installations.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

The manual steps required in order to perform a release can lead to any number
of errors by fault of the responsible member. These include errors such as: 
- forgetting to perform a particular step
- completing the steps in an incorrect order
- dealing with flakiness related to the timing of when steps are completed

Additionally, the member must "babysit" the individual steps in order to start
the next one. This can be time consuming.

Instead of addressing these points individually, the release process should be
re-evaluated as a whole in order to utilize the CI workflow to automate the
process where possible.

Once the release process is changed, the responsible member will still be
required to summarize the change notes and receive feedback by opening a pull
request. Once that pull request is merged, the member will create a release
tag which triggers the release process. From there, the building, testing, and
uploading of the release artifacts will be automated. Once those steps
complete, the member will send out announcement emails to the CNCF mailing
lists.

# Design proposal (Step 2)

**Waiting until completion of Step 1**

[design-proposal]: #design-proposal

**Note**: We suggest waiting to do this work, until Step 1 of the process is complete and you have has received feedback.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear
- It is reasonably clear how the contribution would be implemented
- Corner cases are dissected by example
- Dependencies on libraries, tools, projects or work that isn't yet complete
- Use Cases
- Goals
- Non-Goals
- Deliverables

# Prior art

[prior-art]: #prior-art

The proposed change is similar to the release process for [linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy)
which has been working well.

This change is different than the above in the following ways:
- In order to run integration tests, it must tag the images with the release
  tag and push those images to a docker registry so that they can be tested on
  a cloud provider.
- It will deploy helm chart updates if integrations tests are successful.
- It may be responsible for updating hard-coded versions in other repositories.
- (Dependent on the above) It may be responsible for asserting the website has
  updated the install script for the release.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- If giving CI push access to [website](https://github.com/linkerd/website) should be avoided, what is the best way
  for the hard-coded version to be updated as an automated step in the process?

# Future possibilities

[future-possibilities]: #future-possibilities

- Create a script that validates a release tag is the correct reference
  format.
- Create a script that generates the next tag for an edge or stable release
  tag.
- Automate the sending of announcement emails to the CNCF mailing lists
