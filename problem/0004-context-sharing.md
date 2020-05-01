- Contribution Name: context_sharing
- Implementation Owner: Alex Leong
- Start Date: 2020-04-21
- Target Date:
- RFC PR:
- Linkerd Issue:
- Reviewers:

# Summary

[summary]: #summary

When one meshed pod talks to another, we would like for the source proxy to be
able to communicate additional information to the destination proxy.  This would
allow the destination proxy to expose better metrics and avoid duplicating work
already done by the source proxy.  This mechanism also gives us flexibility in
case further coordination between proxies is desired in the future.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

When one meshed pod talks to another, communication passes from the application
container in the source pod to the outbound proxy in the source pod, to the
inbound proxy in the destination pod, and finally to the destination container.
The traffic passes through two different Linkerd proxies, but these proxies do
not coordinate with each other[^1], instead they both independently act on the
request.  This means that information which is known only to one of the proxies
is not available to the other.  For example, the source proxy knows certain
Kubernetes metadata about itself such as the name of the pod and pod owner that
it is running in, but this source metadata is not available to the destination
proxy.  This would facilitate the addition of source metadata on server metrics
as described in [RFC #3](https://github.com/linkerd/rfc/pull/15).  Another
example is the destination proxy does not know what protocol the source proxy
detected for the traffic.

[^1]: Certain metadata about HTTP requests such as the canonical destination is
communicated between proxies via special `l5d-` prefixed HTTP headers which are
added by the source proxy and stripped by the destination proxy.

We would like to introduce some mechanism to share context between proxies so
that:

* the destination proxy can expose metrics which include source proxy metadata
* the destination proxy can avoid duplicating work already performed by the
  source proxy such as protocol detection

Furthermore, this mechanism should be available regardless of protocol.  That is,
it should work for HTTP traffic, TCP traffic, and any other protocol that is
added in the future.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

# Prior art

[prior-art]: #prior-art

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities
