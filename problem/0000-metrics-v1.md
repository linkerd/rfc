# Linkerd Metrics v1

- Contribution Name: metrics-v1
- Implementation Owner:
- Start Date: 2020-04-23
- Target Date:
- RFC PR:
- Linkerd Issue:
- Reviewers: @grampelberg @adleong

## Problem Statement

[problem-statement]: #problem-statement

Linkerd's `stat` API powers both the command-line and web interfaces, but it's tailored closely
to their use cases and it has evolved only to express the features of these tools. Because of its
narrow focus, integrators tend to read data directly from Prometheus, which features an
expressive query language and a variety of external integrations/extensions. This presents an
implicit, undocumented relationship between the proxy's metrics exposition structure and traffic
in Kubernetes.

The goal of this RFC is to define an external-facing API for Linkerd that:

1. The semantics of which are defined by this RFC;
2. Is stable/versioned;
3. Supports Linkerd's UIs as well as arbitrary external clients;
4. Supports Kubernetes RBAC;
5. Supports extensions for additional protocols, metrics, and metadata; and
6. Supports arbitrary workload hierarchies (i.e for custom operators).

## Design Notes

### API

The metrics API is read-only.

In order to support RBAC, all requests should be scoped to the resources on which metrics are
measured.

It is time-oriented. It should have both time-series and summary capabilities.

Each Histogram should be given sufficient, published defaults.

#### TCP

- Open Count, by
  - Interface: `inbound | outbound`
  - Workload
  - Service
  - Service profile route
  - Peer Identity
  - Peer K8s Labels? How?
- Close Count, by the above as well as an errno.
- Connect Latency Histogram, by the above
- Connection Age Histogram, by the above
- Bytes Transferred, by the above

#### HTTP

- Request count, by
  - Interface: `inbound | outbound`
  - Authority
  - Workload
  - Service
  - Service profile route
  - Peer Identity
  - Peer K8s Labels? How?
- Response count, by
  - The above, plus
  - Status Code; [100, 599]
  - Classification: success | failure
- Response Latency Histogram
  - Should users be able to query at arbitrary percentiles?

##### gRPC

How do we represent the gRPC-specific metadata that Linkerd exposes? It seems like it should be
distinct from (but in addition to) the HTTP-level abstractions? Or should HTTP metrics have a
hole to punch through arbitrary metadata?

Should we provide gRPC stats scoped by gRPC service name? We can infer these via known path
structure when the appropriate headers are present.

#### K8s services

One of the main complexities in our metrics comes from handling Service-specific metadata.
Currently all service-specific metadata is measured on the client-side, even when the querier
asks about server-side resources. It is effectively impossible to query for server-side metrics
for a service, since we allow pods to exist in more than one service. This is
[fixable][rfc-src-metrics], however, if servers are able to discover metadata for inbound
traffic.

It seems like we should not support querying service metrics as a resource scope, though, because
we cannot measure traffic _on_ a Service. We can measure outbound traffic from a set of client
pods to a Service; and we can measure inbound traffic to a set of server pods that is targetted
at a Service. I think our API should be explicit about these constraints.

[rfc-src-metrics]: https://github.com/linkerd/rfc/pull/15

### Prior Art

#### Linkerd Stat

Linkerd's [`Stat`][stat] API is resource oriented, but it only exposes an _outbound_ side, so
there's no way to

Why is `skip_stats` part of the API?

The `tcp_stats` flag is a legacy of the CLI-specific nature of the command... it seems like we
should either support a list of protocols or have protocol-specific endpoints.

[l5d-stat]: https://github.com/linkerd/linkerd2/blob/dacf87e084a55757aab9cfa8556053495e43e207/proto/public.proto#L348-L427

#### SMI

[SMI's metrics][smi-spec] appear to be stringly typed, in that their are a few well-defined
(http-specific?) metrics, but the base set of metrics is insufficient to satisfy Linkerd's CLI
needs.

However, the way it's resource-oriented is very appealing.

It does not yet expose any time-series functionality.

[smi-spec]: https://github.com/servicemeshinterface/smi-spec/blob/d5dd526a1a19c784065dc3fbac05a52decc01735/traffic-metrics.md

#### Prometheus

Can we expose a [Prometheus API][prom-api] scoped under each resource? E.g.
`.../v1/deploy/deployname-xyzxb/query?...` and `.../v1/deploy/deployname-xyzxb/metrics`?

We would conceivably parse, validate/authorize, and translate prometheus queries to be issued to
the backing store. We would define a well-defined set of public metrics/labels.

This could hit the sweet spot of interopability with scoping for access control.

[prom-api]: https://prometheus.io/docs/prometheus/latest/querying/api/
