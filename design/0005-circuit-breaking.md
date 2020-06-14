# Title

- Contribution Name: circuit-breaking
- Implementation Owner: @Ashish-Bansal, Linkerd Team
- Start Date: 2020-06-14
- Target Date: N.A.
- Linkerd Issue:
  [linkerd/linkerd2#2846](https://github.com/linkerd/linkerd2/issues/2846)
- Reviewers: @adleong

## Summary

[summary]: #summary

We would like to implement circuit breaking functionality which would allow end
users to configure mesh such that they can limit the impact of failures from
faulty service over others and prevent cascading issues. By configuring circuit
breaking, they will be able to remove failing service instances from load
balancing and survive through it in a more graceful manner.

## Problem Statement

In a service mesh, services often make calls to other services. In case upstream
service is unable to respond due to any reason, it's possible that these
failures can cascade to other services in the cluster.

If we are aware that specific service instance is not responding, we can call it
unhealthy and it's better if we just stop sending "traffic"[0] to that instance
rather than getting just another expected failure.

Users should also be able to set max-ejection-percentage which provides max
number of service instances which can be removed from load balancing for
specific service.

Along with that, if we detect that "almost all" the instances of service are not
responding, then users should be able to cut off traffic to whole service as
well. To quantize "almost all" term, user should be able to configure how many
percentage of service instances would be removed from the load balancing list
before whole traffic to that service would be circuit-breaked. This is in
constrast with max-ejection-percentage and would be mutually exclusive.

User should also be able to select ejection policy based on which any instance
would be ejected from load balancing.

It would also be somewhat helpful in case of canary deployments, when there's
some regression and new build is raising failures.

[0] The term 'traffic' means any kind of communication happening between two
services. It can be HTTP requests or gRPC messages over HTTP or L4 TCP
connections. But the circuit breaking functionality itself won't be protocol
agnostic. Its implementation would be protocol specific and it's upto community
which protocol they want to add support for and how circuit breaking will make
sense in that protocol.

## Design proposal

[design-proposal]: #design-proposal

Right now if I just consider ServiceProfile spec, IMO it seems to represent more
of profile-for-http-service rather than profile-for-generic-service. May be it
makes sense given that it supports only HTTP and grpc over HTTP as of now. Given
current architecture, I feel it's best to provide circuit breaking functionality
through service profiles. That means it would be limited to above two kind of
services as of now.

For TCP connections, according to original github
[issue](https://github.com/linkerd/linkerd2/issues/2846#issue-447856936), it
seems like linkerd2 does support automatic circuit breaking for TCP errors but
I'm not sure of its behaviour -- whether TCP error means connection refused or
resets or timeouts or something else. Anyway, it seems like it's not
configurable and I'm not sure if making it configurable has any important
usecases.

In future, for any other protocol support for circuit breaking, one will first
need to incorporate the protocol into ServiceProfile itself and then implement
the circuit breaking functionality.

Regarding ejection of the service instance, once we decide any instance needs to
be ejected in accordance with mentioned configuration, it would be ejected from
all the proxies i.e. we **won't** provide user ability to configure it just for
proxy which raised those failures due to which ejection got triggered.

Let me explain above piece with an example. Let's say there are two services A
and B which calls C service. In case B->C gets success rate of 100% but A->C
gets success rate of 50%, it won't be possible for user to configure
service-mesh such that it would trigger circuit breaking only for A->C traffic.
Instead either circuit breaking will happen for both A and B (through
aggregation of their stats) or none of them.

This certainly has its own disadvantages but I would be somewhat inclined
towards having it simpler unless usecases for making it per-proxy over-weighs
other one. So, for now, I propose that every ejection policy would be based on
taking aggregated metrics over all the proxy instances to mark any instance for
ejection.

For HTTP/gRPC, I feel like taking advantage of linkerd provided golden metrics
for ejection policies. Ejection can be based on success-rate,
percentile-latencies for a given time window. Circuit breaker settings would be
applied per service + protocol instead of per-route. In case circuit breaker is
triggered for entire service, then we will return 503 for HTTP with some custom
header and gRPC message for gRPC.

If we go with the above design, may be we can keep most of the ejection logic in
control plane i.e. in the destination API, we can just remove ejected endpoints
from list given to discovery.

Other than above mentioned details, here's the top level changes I could think
of with this approach -

1. Updating ServiceProfile CRD and its validation.
2. Updating destination service to use circuit breaking configurations specified
   by user to perform ejection. It can use prometheus metrics for evaluation and
   update endpoints as required.
3. Sharing some information through destination API to linkerd2-proxy so that it
   can return adequate response to the requests.
4. AFAIK metrics exposed by each proxy doesn't container upstream service
   instance's endpoint.
5. Exposing metrics related to circuit breaking from control plane.

There's lot more things which needs to be discussed but I would first like to
get some feedback if basic idea sounds sane to everyone.

### Prior art

[prior-art]: #prior-art

Other service meshes seems to be performing this in data plane instead, may be
because it aligned well with their architecture. All the service meshes built
over top of enovy, uses envoy's inbuilt capability of circuit breaking. In
[envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier),
they use two terms - "Circuit Breaking" and "Outlier Detection", which I feel
seems inter-related upto some extent and can be merged into single. Envoy
supports both cluster-wide as well as per-proxy outlier detectors.

With the above proposed design, we won't be able to support per-proxy outlier
detectors cleanly, since we are abstracting almost all circuit-breaking logic
from data plane. Also I guess getting metrics about other proxies in the cluster
can potentially be a pain point if we move this logic into data plane.

### Unresolved questions

[unresolved-questions]: #unresolved-questions

ToDo.

### Future possibilities

[future-possibilities]: #future-possibilities

ToDo.
