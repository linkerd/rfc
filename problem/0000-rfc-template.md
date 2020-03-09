- Contribution Name: Thrift Support
- Implementation Owner: Peter Novotank <peter.novotnak@reddit.com>, Linkerd2 Authors
- Start Date: 6/3/2020
- Target Date: 6/15/2020
- RFC PR: [linkerd/rfc#9](https://github.com/linkerd/rfc/pull/9)
- Linkerd Issue: [linkerd/linkerd2#0000](https://github.com/linkerd/linkerd2/issues/0000)
- Reviewers: (Who should review the code deliverable? ex.@olix0r)

# Summary

[summary]: #summary

Linkerd2-proxy and control plane support for the Thrift protocol.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

- Add protocol support to the linkerd2-proxy for at least TFramedTransport and TBinaryProtocol 
- Support routing, retries and metrics for Thrift.

In many organizations, Thrift is used as the internal RPC protocol. In order to support 
this use-case, linkerd2 needs to support routing and rate limiting. These features will 
allow us to support advanced routing, as well as allow users to configure their mesh to 
prevent retry storms and provide metrics.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

**Note**: This should be completed as part of `Step 2`.

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

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this functionality exist in other software and what experience has their community had?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some
  relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other
software, provide readers of your RFC with a fuller picture. If there is no prior art, that is
fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from
other software.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

# Future possibilities

[future-possibilities]: #future-possibilities

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.
