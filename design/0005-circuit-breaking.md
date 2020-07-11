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

---

It would also be somewhat helpful in case of canary deployments when there's
some regression and new build is raising failures.

[Example](https://drive.google.com/open?id=1FxKi-egyhHumCvlHvTI4P114z7qOLPeN)

---

Users should also be able to set max-ejection-percentage which provides max
number of service instances which can be removed from the load balancing for
specific service.

[Example](https://drive.google.com/open?id=1jSMaogdvzxjv0v2dBoL7OYGHmsCusXn-)

---

Network Parition - There can be lot of scenarios in network partition but to
provide a glimpse how circuit breaking would behave in case of network issue -
[Example](https://drive.google.com/open?id=154I2ZGlGc-rQJ0YVeek17dJy7raAvd-G)

---

Protocol Specific features -

1. For HTTP/gRPC based communication, users should be able to configure circuit
   breaking either per endpoint or whole service.
   [Example](https://drive.google.com/open?id=1JK65u9Mnmkmy_X8fczeh6RO6O-vNiTtY)
   User can configure ejection based on many different metrics. These metrics
   would be calculated over a given time range.
   - Success Rate
   - Percentile Latencies
   - Max Requests
   - Consecutive Failures
2. For TCP based communication -
   [Example](https://drive.google.com/open?id=1yAAurygsZQ-r58TEA0QiFf9Sz15jqRJV)

---

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
services as of now. Circuit breaking logic would be applied in the data plane
based on the metrics recorded inside the proxy.

Since circuit breaking needs to be handled per protocol, we will leave TCP for
now and can revisit the same in future.

In future, for any other protocol support for circuit breaking, one will first
need to incorporate the protocol into ServiceProfile itself and then implement
the circuit breaking functionality.

For HTTP/gRPC, we already have golden metrics available in the proxy. Along with
that, we can add additional metrics and use those for circuit breaking logic. In
case no instance is available to serve request due to circuit breaking, then we
will return 503 for HTTP with some custom header and gRPC message for gRPC.

ServiceProfile interface would have following additions for the HTTP and gRPC -

### Circuit Breaker

```yaml
routes:
  - name: "/foo"
    circuitBreaker:
      # Circuit Breaker Kind
      # (Required)
      kind: ...

      # Max percentage of instances that can be ejected.
      # Default - 100%
      maxEjectionPercentage: 10

      # The min duration of time in seconds for which an instance is marked
      # as ejected.
      # (Required)
      baseEjectionTime: 60

      # Ejection Time Inflation
      # How to increase ejection time on every consecutive ejection.
      # (Optional. Default - None)
      ejectionTimeInflation: ...
```

### Circuit Breaker Kind

```yaml
kind: successRate
# Sliding window in seconds over which we will calculate given metric
# (Required)
timeWindow: 15
successRate: 99
```

```yaml
kind: maxRequests
timeWindow: 15
maxRequests: 10000
```

```yaml
kind: percentileLatency
timeWindow: 15
# p99 or p95 or p50 in ms
p99: 500
```

```yaml
kind: consecutiveFailures
failures: 5
```

### Ejection Time Inflation

```yaml
ejectionTimeInflation:
  # The min duration of time in seconds for which an instance is marked
  # as ejected.
  # (Optional. Default - None)
  maxEjectionTime: 120

  # Increases ejection time by constant value on every ejection
  kind: constant
  ms: 1000
```

```yaml
ejectionTimeInflation:
  # Increases ejection time exponentially based on provided multiplier
  kind: exponential
  multiplier: 2
  maxEjectionTime: 120
```

---

Circuit breaking logic -

1. Get all available non-ejected upstream service instances for the route.
2. Based on existing load balancing algorithm, send request to respective
   instance. If all the instances are marked as ejected, then return appropriate
   response to caller (e.g. 503 in case of HTTP).
3. Based on response from the upstream, update metrics for that specific
   instance.
4. Update instance ejection status for inline-circuit breakers (like Consecutive
   Failures) and return response to the downstream.

In background, for non-inline circuit breakers, check if any (route, instance)
pair needs to be marked as ejected or if any instance has served ejection time
and it's ready to be brought back online. In case it's consecutive ejection,
increase ejectionTime according to `ejectionTimeInflation` settings.

---

Other than above mentioned details, here's the top level changes I could think
of with this approach -

1. Updating ServiceProfile CRD and its validation.
2. Updating destination service APIs to provide circuit breaking configuration
   to proxy instances.
3. Using existing local telemetry to apply circuit breaking configuration in
   proxy itself.

All diagrams are collectively available
[here](https://drive.google.com/drive/folders/1wCwTwi6kUjHPsLRMPBasy3cnx5Gj6SSq?usp=sharing).

### Prior art

[prior-art]: #prior-art

All the service meshes built over top of enovy, uses envoy's inbuilt capability
of circuit breaking. In
[envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier),
they use two terms - "Circuit Breaking" and "Outlier Detection", which I feel
seems inter-related upto some extent and can be merged into single. Envoy
supports both cluster-wide as well as per-proxy outlier detectors.

### Unresolved questions

[unresolved-questions]: #unresolved-questions

ToDo.

### Future possibilities

[future-possibilities]: #future-possibilities

ToDo.
