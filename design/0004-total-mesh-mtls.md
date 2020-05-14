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
ubuiquitous, automatic, transparent mTLS for all meshed traffic, regardless of protocol.

### Goals

1. All meshed traffic is private;
2. All meshed traffic is associated with Linkerd identities;
3. Provide diagnostic tools to inspect the mTLS status of connections.

### Non-Goals

- Implement authorization policies
- Provide auditability guarantees

## Design Proposal

[design-proposal]: #design-proposal

### Outbound Proxy

No changes are needed to the inbound proxy. We continue to terminate mTLS on
all inbound traffic regardless of the overlying protocol.

On the outbound side, however, we accommodate services with comprising pods
of multiple identities. So we cannot continue to forward traffic to
`ClusterIPs`. This requires introducing a [TCP load balancer][tcplb] so that
we have an efficient strategy to distribute connections over endpoints.

#### Discovery

The control plane's _Destination.Get_ endpoint already supports resolving
both `cluster-ip:port` and `pod-ip:port` addresses to a balancer endpoint
(with metadata and identity).

The outbound proxy provides a p2c/fewest-connections load balancer to
distribute connections over `ClusterIp` service endpoints.

#### Metrics

Metrics for these connections should include per-endpoint metric labels, as
addressed by the Metrics API proposal.

#### Server-Speaks-First Protocols

Another open proposal outlines support for service-speaks-first-protocols.
This other proposal should influence the implementation of discovery in this
proposal.

### Diagnostic Tools

TODO
