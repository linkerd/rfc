# External Traffic Classification

- Contribution Name: `external_traffic_classification`
- Implementation Owner:
- Start Date: 2020-09-14
- Target Date:
- RFC PR: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/0000)
- Linkerd Issue:
  [linkerd/linkerd2#4003](https://github.com/linkerd/linkerd2/issues/4003)
- Reviewers:

## Summary

[summary]: #summary

Presently, any traffic that arrives to a pod from outside the service mesh
(e.g. from an ingress or other load-balancer) is categorized only by its source
IP address, and there is no way to further break down rps/error rates beyond
that. Since multiple sources might share an internal load balancer (for example
a GCP CloudComposer or DataFlow job talking to an internal LB deployed for a
service in a GKE cluster), it would be extremely helpful to see in tap or the
linkerd web console what the actual source of the traffic is.

## Problem Statement (Step 1)

[problem-statement]: #problem-statement

### What problem are you trying to solve

Users should be able to see a named source for traffic which originates
from inside areas of their control (e.g. a non-kubernetes service running
in the same network as the cluster) but which does not originate inside
a kubernetes cluster running linkerd.

Users should be able to configure prometheus dashboards/alerts based on
traffic sources even if the traffic source is on the other side of a load
balancer.

### Requirements

- It should not be possible for malicious or incompetent external actors
  to make linkerd produce confusing or incorrect data in the console, cli or
  prometheus
    - there must be a way to verify that the proposed label comes from a trusted source
    - there should be defined behavior in the case of a label overlap (ie if
      external traffic claims to be the same source as a service inside the mesh)

## Design proposal (Step 2)

[design-proposal]: #design-proposal

**Note**: This should be completed as part of `Step 2`.

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear
- It is reasonably clear how the contribution would be implemented
- Corner cases are dissected by example
- Dependencies on libraries, tools, projects or work that isn't yet complete
- Use Cases
- Goals
- Non-Goals
- Deliverables

### Prior art

[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A
few examples of what this can include are:

- Does this functionality exist in other software and what experience has their
  community had?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other software, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us
whether they are brand new or if it is an adaptation from other software.

### Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?

### Future possibilities

[future-possibilities]: #future-possibilities

This is also a good place to "dump ideas", if they are out of scope for the RFC
you are writing but otherwise related.
