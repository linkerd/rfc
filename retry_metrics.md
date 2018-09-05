# RFC: Best Practices for Structuring Modular Request Metrics

## Problem

The Linkerd proxy records metrics associated with requests across various
dimensions.  For example, the `request_total` counter counts the number of
requests and has labels such as `authority`, `dst_deployment`, and so on.  The
format of these metrics and labels are described in the [Linkerd 2
docs](https://linkerd.io/2/proxy-metrics/).

However, we would like to have the ability to record these metrics at multiple
different points in the request's lifecycle.  For example, we may wish to record
metrics both before and after performing retries.  The result is that we would
have one set of metrics which describe the "logical" requests made by the caller
and another set of metrics which describe the "physical" requests sent to the
service.  When retries occur, a single "logical" request can map to multiple
"physical" requests.

This RFC concerns itself with how these metrics should be named and labeled.
We wish to structure these metrics such that it is easy to differentiate
between them in queries and that follows Prometheus best practices.  We would
also like a solution that generalizes well to having more locations where
request metrics are recorded e.g. before load balancing (per-service), after
load balancing (per-endpoint), after connection pooling (per-connection), etc.

## Option 1: Distinct Label Names

One solution would be to use completely distinct names for the labels when
recording logical requests from the label names when recording physical
requests.  For example, the `request_total` counter could be incremented with
labels like `logical_authority`, `logical_dst_deployment`, and so on when
counting the logical request (before retries) and incremented with labels like
`physical_authority`, `physical_dst_deployment`, and so on when counting the
physical request.

## Option 2: Distinguishing Label

Another solution would be to use a single label to distinguish the location
where the metric is recorded.  For example, the label `type=logical` can be used
when recording logical requests (before retries) and the label `type=physical`
can be used when recording physical requests.

Care must be taken to always specify the `type` label in queries, otherwise
the query will aggregate across types and double-count requests.

## Option 3: Distinct Metric

Yet another solution would be to use a completely separate metric for logical
requests and for physical requests.  For example, `logical_request_total` and
`physical_request_total`.