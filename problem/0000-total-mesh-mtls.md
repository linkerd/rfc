# mTLS for all meshed communication

- Contribution Name: total-mesh-mtls
- Implementation Owner:
- Start Date: 2020-04-16
- Target Date:
- Linkerd Issue:
  [linkerd/linkerd2#3691](https://github.com/linkerd/linkerd2/issues/3691)
- Reviewers: @grampelberg

## Problem Statement

[problem-statement]: #problem-statement

Linkerd implements an identity system for meshed communication. This system
builds on Kubernetes' `ServiceAccount`s, which manage per-workload
authentication tokens for the Kubernetes API. Linkerd uses these credentials to
bootstrap and validate TLS certificates that can be used to provide
mutually-validated TLS (mTLS) connections between meshed endpoints. This mutual
validation will serve as the basis for auditing and policy functionality.

Today, mTLS is only applied on HTTP connections. This RFC will provide a plan to
provide ubiquitous, automatic, transparent mTLS for all meshed traffic,
regardless of protocol.

### Goals

1. All meshed traffic is private;
2. All meshed traffic is associated with Linkerd identities;
3. Provide diagnostic tools to inspect the mTLS status of connections.

#### Non-goals

- Implement authorization policies
- Provide auditability guarantees
