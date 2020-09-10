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

Linkerd's `stat` API powers both the command-line and web interfaces, but
it's tailored closely to their use cases and it has evolved only to express
the features of these tools. Therefore, anyone trying to access this data
must access Linkerd's Prometheus API directly. This presents several
problems:

1. Cluster operators need guidance on how to support Linkerd's Prometheus
   data. There should be guidance on how to manage cardinality (especially in
   larger clusters) and how to support multi-tenant environemnts.
2. Consumers have inadequate documentation and tools to access Linkerd's
   metrics data. The proxy's metrics export has been treated as an
   implementation detail and not a public API. Information is exposed in such a
   way that a consumer essentially has to understand how the proxy is
   implemented to know how to interpret the data.

The goal of this RFC is to define an external-facing metrics API for Linkerd.
This API should be stable, versioned, and extensible to support additional
protocols, metrics, and metadata (via future RFCs).
