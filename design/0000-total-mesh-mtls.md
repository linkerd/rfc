# mTLS for all meshed communication

- Contribution Name: total-mesh-mtls
- Implementation Owner:
- Start Date: 2020-04-16
- Target Date:
- Linkerd Issue: [linkerd/linkerd2#3691](https://github.com/linkerd/linkerd2/issues/3691)
- Reviewers: @grampelberg

## Problem Statement

[problem-statement]: #problem-statement

Linkerd implements an identity system for meshed communication. This system builds on Kubernetes'
`ServiceAccount`s, which manage per-workload authentication tokens for the Kubernetes API.
Linkerd uses these credentials to bootstrap and validate TLS certificates that can be used to
provide mutually-validated TLS (mTLS) connections between meshed endpoints. This mutual
validation will serve as the basis for auditing and policy functionality.

Today, mTLS is only applied on HTTP connections. This RFC will provide a plan to provide
ubuitous, automatic, transparent mTLS for all meshed traffic, regardless of protocol.

### Goals

1. All meshed traffic is private;
2. All meshed traffic is associated with Linkerd identities;
3. Provide diagnostic tools to inspect the mTLS status of connections.

### Non-Goals

* Implement authorization policies
* Provide auditability guarantees

[design-proposal]: #design-proposal

<!-- TODO/SKETCH -->

### Outbound Proxy

- For outbound connections where the protocol could not be detected as HTTP:
  - Maintain a fewest-connections load balancer for each unique IP:PORT
  - Endpoints provided by resolving IP:PORT via the Destination controller.
    - Endpoint metadata includes peer identity, as per HTTP resolution.
  - If protocol detection was disabled via configuration (i.e. because the connection is
    server-speaks-first), a hint must be forwarded to the inbound proxy so that it can bypass

#### Client-speaks-first protocols

#### `LoadBalancer` IPs

### Inbound Proxy

### Destination Controller

- Support a `Service` annotation that denotes a list of server-speaks-first ports.

### Proxy Injection Controller

<!--

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

# Unresolved questions

[unresolved-questions]: #unresolved-questions

transparent

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

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


# Future possibilities

[future-possibilities]: #future-possibilities

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

  -->
