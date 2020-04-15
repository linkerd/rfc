- Contribution Name: Source Metadata for Traffic Metrics
- Implementation Owner: Alex Leong
- Start Date: 2020-04-14
- Target Date: 
- RFC PR: 
- Linkerd Issue: 
- Reviewers: 

# Summary

[summary]: #summary

When a meshed pod sends outgoing HTTP requests the Linkerd proxy records metrics
such as latency and request counters and scopes those metrics by the
destination.  This is done by setting a `dst_X` label on the Prometheus metrics
where `X` is the destination workload kind.  On the other hand, when a meshed
pod receives incoming HTTP requests, there is no equivalent scoping of the
metrics by source.  In other words, there is no `src_X` Prometheus label, making
it impossible to break down metrics for incoming traffic by source.  This RFC
proposes adding `src_X` Prometheus labels for incoming HTTP traffic.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

Linkerd is not able to add `src_X` labels today because it simply has no
knowledge of the source resource.  It knows the peer socket address of the
source, but has no mechanism to convert that address into a Kubernetes resource
name or type.  This is in contrast to the `dst_X` metadata which the proxy
gets from the Destination controller when doing service discovery look-ups.

This asymmetry in metadata can be very limiting when doing queries.  It is
impossible to determine who the clients of a resource are by looking at that
resource's metrics alone.  Instead, we need to query the outbound metrics of all
other resource to find a client with the appropriate `dst_X` label.  Not only
does this make the query awkward, it also means that resource-to-resource
metrics can only be observed on the client side, never on the server side.  This
limits our ability to measure network latency.

Adding source metadata to HTTP traffic metrics would enable improvements in the
Linkerd Grafana dashboard, 3rd party tools that consume Linkerd's Prometheus
metrics, the controller's StatSummary API, and consequently the `linkerd stat`
CLI command and Linkerd dashboard.  These improvements are out of the scope of
this proposal.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

We will use the [Downward
API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#the-downward-api)
to create and mound a volume in meshed pods which contains a file with the pod
owner's resource name and kind.  This is very similar to [the approach used to
populate pod labels on span
annotations](https://github.com/linkerd/linkerd2/pull/4199).  These values can
be extracted from pod labels such as `linkerd.io/proxy-deployment`,
`linkerd.io/proxy-daemonset`, etc.  We can create a simple startup script in the
proxy container to preprocess these pod labels into the labels that we would
like the proxy to use.  For example, the Downward API would create a volume with
a file containing:

```
app="web-svc"
linkerd.io/control-plane-ns="linkerd"
linkerd.io/proxy-deployment="web"
pod-template-hash="5cb99f85d8"
```

Our startup script would rewrite this to keep only the labels that we want and
to put them in the desired format:

```
resource_name="web"
resource_kind="deployment"
```

The proxy would read this file to get a list of labels to apply to its traffic
metrics.  The way the proxy uses these labels differs for the inbound and
outbound stacks.


For the inbound stack, the proxy will prepend the `dst_` prefix to these labels
and use them to scope the inbound metrics.  This will result in metrics such as:

```
request_total{
  direction="inbound",
  dst_resource_name="web",
  dst_resource_kind="deployment",
  ...
}
```

The `dst_` prefix is used like this because on the inbound side, the destination
is the local resource.  Source labels are also added to the inbound metrics but
we'll come back to that in a moment.

For the outbound stack, the proxy will prepend the `src_` prefix to the labels
and use them to scope the outbound metrics.  The corresponding `dst_` labels
will be populated by the dst metadata from the destination controller.  This
will result in metrics such as:

```
request_total{
  direction="outbound",
  src_resource_name="web",
  src_resource_kind="deployment",
  dst_resource_name="emoji",
  dst_resource_kind="deployment",
  ...
}
```

The outbound stack will also encode the source labels in an HTTP header of the
outgoing request called `l5d-src-labels`.  An example of the value of this
header would be: `resource_name=web,resource_kind=deployment`.  Encoding this
source metadata on the request allows it to used by the inbound stack of the
destination proxy.  Remember when we said we'd come back to inbound?

In addition to populating the `dst_` labels, the inbound stack will also read
the `l5d-src-labels` HTTP header from the request, prepend `src_` to them, and
add them to the label scope.  Thus, the complete inbound metrics would actually
look like:


```
request_total{
  direction="inbound",
  dst_resource_name="web",
  dst_resource_kind="deployment",
  src_resource_name="vote-bot",
  src_resource_kind="deployment",
}
```

Note that all of the changes described here are additive to the existing
Prometheus labels and would not introduce any backwards incompatibility.

This change does increase the cardinality of Prometheus time-series since
inbound metrics will now be scoped by source resource.  This will roughly bring
the cardinality of the inbound metrics to the same as the outbound metrics.

Note also that this implementation means that source metadata will NOT be
available when the source resource is not meshed.

There are no known blockers or prerequisites before this work can be started.

# Prior art

[prior-art]: #prior-art

The Istio mixer telemetry architecture showcases a different approach to
populating source metadata.  In that architecture, rather than having data plane
proxies expose metrics to Prometheus directly, the metrics are ingested by the
control plane and post-processed.  Source IP addresses are resolved to source
resources before being stored.

It would be difficult to incorporate this approach in Linkerd without
introducing significant complexity.  The fact that Prometheus is able to scrape
Linkerd proxies directly is a great property.  Alternatively, having the Linkerd
proxy resolve IP addresses to resources would either require an API call which
would introduce latency in the critical data path or would require an in-process
cache which would increase the proxy memory footprint.

There is also precedent in Linkerd for proxies to communicate metadata to one
another using the `l5d-*` headers.  For example, the fully qualified authority
is communicated between proxies using the `l5d-dst-canonical` header.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

Existing metrics stack modules in the proxy have a fixed label scope per stack.
However, this proposal describes labels set dynamically from the incoming
request. Is this feasible in the proxy stack architecture?

# Future possibilities

[future-possibilities]: #future-possibilities

Make use of these new metrics in the Grafana dashboards, StatSummary API, Linkerd
CLI and Linkerd dashboard.
