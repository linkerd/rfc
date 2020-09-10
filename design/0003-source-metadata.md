- Contribution Name: Source Metadata for Traffic Metrics
- Implementation Owner: Alex Leong
- Start Date: 2020-04-14
- Target Date: 
- RFC PR: 
- Linkerd Issue: 
- Reviewers: 

# Summary

[summary]: #summary

Linkerd's metrics API lacks the ability to query for server-side metrics when
doing a resource-to-resource query.  When metrics for a single resource are
requested, the returned metrics are always measured on the server-side.  But
when metrics for traffic between two resources is requested, the returned
metrics are always measured on the client-side.  This behavior is both
unintuitive and limiting.  Users are frequently unaware or surprised that these
two types of queries are measured differently.  Without any way to measure
resource-to-resource traffic metrics on the server-side, it is impossible to
compare client-side to server-side metrics for this type of traffic to identify
network introduced latency or errors.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

Linkerd's metrics API requests have this structure:

```
message StatSummaryRequest {
  ResourceSelection selector = 1;
  string time_window = 2;

  oneof outbound {
    Empty none = 3;
    Resource to_resource   = 4;
    Resource from_resource = 5;
  }

  bool skip_stats = 6;  // true if we want to skip stats from Prometheus
  bool tcp_stats = 7;
}
```

If the `outbound` field is set to `none` then metrics are measured on the
server-side of the `selector` resource.  However, if the `outbound` field is
set to `to_resource` then the metrics are measured on the client-side of the
`selector` resource.  Finally, if the `outbound` field is set to `from_resource`
then the metrics are measured on the client-side of the `from_resource`.

This API is confusing for a few reasons:

* Some types of queries are measured on the server-side while others are
  measured on the client-side.
* Some types of queries are measured from the `selector` resource while others
  are not.

More importantly, some types of queries are not possible: it is not possible to
query for resource-to-resource traffic measured on the server-side.  This
limitation is significant because it means that is it is impossible to compare
client-side to server-side metrics for this type of traffic to identify network
introduced latency or errors.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

We will change the semantics of the StatSummary API to behave in a more
predictable and consistent way.  Specifically, we will rename the `outbound`
field to `edge` and change the semantics to be that traffic is always measured
at the `selector` resource.  This means that when `edge` is set to `none`,
traffic will be measured on the server-side of the selector resource (no change
from today's behavior), when `edge` is set to `to_resource`, traffic is measured
on the client-side of the `selector` resource (no change from today's behavior),
and when `edge` is set to `from_resource`, traffic is measured on the
server-side of the `selector` resource.

An alternative to modifying the semantics of the StatSummary API is to create a
new metrics API that would eventually replace StatSummary.  This has the added
benefit of giving us the opportunity to simplify this API by allowing us to
drop support for features which are not core to traffic metrics such as meshed
pod count, as well as moving traffic split metrics into its own command.  This
approach may also avoid unpredictable behavior when using mismatched CLI and
control plane versions.  However, building a new API would require a larger
effort than tweaking the existing one.

In order to satisfy the new semantics, our Prometheus data must be rich enough
to be able to select traffic metrics from specific sources when measuring on
the server (inbound) side.  The inbound proxy does not attach any label to its
metrics that allow selection by traffic source.  This is because it simply has
no knowledge of the source resource. It knows the peer socket address of the
source, but has no mechanism to convert that address into a Kubernetes resource
name or type. This is in contrast to the `dst_X` metadata which the proxy gets
from the Destination controller when doing service discovery look-ups.  We will
add a corresponding `src_X` label that identifies the source resource.

We will use the [Downward
API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#the-downward-api)
to create and mount a volume in meshed pods which contains a file with the pod
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

The outbound stack will also share the source labels with the inbound stack of
the destination proxy.  Remember when we said we'd come back to inbound?

In addition to populating the `dst_` labels, the inbound stack will also use the
source labels, prepend `src_` to them, and add them to the label scope.
Thus, the complete inbound metrics would actually look like:


```
request_total{
  direction="inbound",
  dst_resource_name="web",
  dst_resource_kind="deployment",
  src_resource_name="vote-bot",
  src_resource_kind="deployment",
}
```

The details of how the source labels will be shared between the source outbound
proxy and the destination inbound proxy are out-of-scope of this RFC but are
discussed in the [Context Sharing RFC](https://github.com/linkerd/rfc/pull/20).

Note that all of the changes described here are additive to the existing
Prometheus labels and would not introduce any backwards incompatibility.

This change does increase the cardinality of Prometheus time-series since
inbound metrics will now be scoped by source resource.  This will roughly bring
the cardinality of the inbound metrics to the same as the outbound metrics.

Note also that this implementation means that source metadata will NOT be
available when the source resource is not meshed.

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

In Istio configuration which do not use mixer, source context is communicated
through a base64 encoded map of key-value pairs which in an HTTP header.  This
approach requires reading and decoding this header on a per-request basis, even
though we know that the source metadata will be the same for all requests on a
single connection.  By communicating source metadata at the connection level,
we can avoid doing this work for each request.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

The details of how to share source metadata will be discussed in 
[another RFC](https://github.com/linkerd/rfc/pull/20).

# Future possibilities

[future-possibilities]: #future-possibilities

Make use of this new metrics API in the Linkerd CLI and dashboard.
