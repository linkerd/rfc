# Implementing Traffic Specs and Enhancing Traffic Splits

- Contribution Name: smi-traffic-specs-and-split-enhancement
- Implementation Owner:
- Start Date: 8-7-2020
- Target Date:
- RFC PR:
- Linkerd Issue:
[linkerd/linkerd2#3165](https://github.com/linkerd/linkerd2/issues/3165)
- Reviewers: @grampelberg

## Summary

[summary]: #summary

Service Mesh Interface (SMI) defines `Traffic Specs` which allows users to
define the characteristics of traffic flowing through the mesh. This can be
used in conjunction with other SMI components such as `Traffic Access Control`
and `Traffic Split`. Since traffic specs are not useful by themselves, this
enhancement would also cover implementing `matches` in the `TrafficSlit`
resource that is already supported in Linkerd. This would let Linkerd support
a/b testing implemented by tools like [flagger](https://docs.flagger.app/).

## Problem Statement (Step 1)

[problem-statement]: #problem-statement

Progressive delivery models such as, canary releases, a/b testing and ring
deployments are becoming more common as tooling reduces friction to adoption.
All but the most basic of these models requires some amount of header based
routing to enable targeting for specific user segments. Currently this is not
supported natively in Linkerd and ingress implementations are sometimes not
compatible. This means moving to progressive delivery models is inhibited by
choosing Linkerd.
