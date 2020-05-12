# Protocol Detection Configuration

- Contribution Name: Protocol Detection Configuration
- Implementation Owner:
- Start Date: 2020-05-04
- Target Date:
- RFC PR:
- Linkerd Issue:
- Reviewers: @olix0r @adleong

## Summary

[summary]: #summary

Instead of requiring users to configure the protocol for every port being used
in an application, Linkerd tries to detect the protocol being used. For
protocols that the proxy knows about (HTTP) and client speaks first protocols,
this works pretty well. Unfortunately, for server speaks first protocols, the
proxy is unable to detect what protocol is being used. This causes delays in
negotiating new connections and in some cases surprising failure behavior when
adding Linkerd to new services.

Today, when an application is using an unsupported protocol, the solution is to
skip either a specific inbound/outbound port or an entire range. When a user
skips ports on a per-pod basis, the `iptables` rules that redirect incoming and
outgoing traffic are changed slightly. Any traffic for these skipped ports
actually skips the proxy. This traffic cannot be mTLS'd and there are no metrics
available.

## Problem Statement

[problem-statement]: #problem-statement

### What problem are you trying to solve?

Users should be able to get the benefit of lower level protocols (L4) when
Linkerd is unable to detect the higher level protocol (L7). This allows users to
get all the benefits they would expect from mTLS, TCP metrics and multi-cluster
routing. To do this, users need to specify whether protocol detection should be
skipped on a per-port basis.

### Requirements

- Allow ports to skip the proxy entirely (inbound and outbound), maintaining
  current behavior.
- Allow ports to skip only protocol detection inside the proxy (inbound and
  outbound).
- Support port ranges as well as individual ports.
- Surface metrics at the TCP level.
- Work for applications where both the client and server are meshed.
- Do not require configuration on both the client and server.

### Out of Scope

- Configuration of the protocol on a port.
- Skipping protocol detection for un-meshed services.

### Option 1: `appProtocol`

In Kubernetes 1.18, a new field was introduced to `Service` - `appProtocol`.
This allows you to hint what protocol a specific service port uses.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
      appProtocol: mysql
  selector:
    app: mysql
  clusterIP: None
```

If appProtocol is specified and not on the list of supported protocols, the
proxy would skip protocol detection.

#### Downsides

- This is currently an alpha feature (in 1.18). It is basically unusable at this
  point in time.
- Requires the destination to do discovery on what services are pointed at it
  and update configuration.

### Option 2: Annotate the service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  annotations:
    config.linkerd.io/skipDetection: 3306
  labels:
    app: mysql
spec:
  ports:
    - port: 80
    - port: 3306
      appProtocol: mysql
  selector:
    app: mysql
  clusterIP: None
```

Similar to `config.linkerd.io/skip-outbound-ports`, the
`config.linkerd.io/skipDetection` annotation would take either a range or comma
separated list of ports to skip detection on for this service.

#### Downsides

- Annotations are unstructured, not in-line and specific to Linkerd.
- Same discovery issues as option 1.

### Option 3: Annotate the destination pod

```yaml
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
    project: booksapp
  annotations:
    config.linkerd.io/skipDetection: 3306
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
        project: booksapp
    spec:
      containers:
        - name: mysql
          image: mysql:5.6
          ports:
            - containerPort: 3306
              name: mysql
```

#### Downsides

- Annotations are unstructured, not in-line and specific to Linkerd.
- Not the direction that Kubernetes is going for configuring application
  protocols

### Thrown Out

- Using port names for skipping protocol detection.
- Requiring configuration on both the client and server side.

### Open Questions

- Does the distinction between inbound and outbound matter?

### Prior art

[prior-art]: #prior-art

- [Kubernetes issue](https://github.com/kubernetes/kubernetes/issues/40244)
  tracking frustration with the inability to mention application protocols
- [KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20191227-app-protocol.md)
  for placing `appProtocol` on `Service` and `Endpoints`.
- [KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20190603-EndpointSlice-API.md)
  for `EndpointSlice`.
- [Manual Selection](https://istio.io/docs/ops/configuration/traffic-management/protocol-selection/)
  is how Istio started doing it. They have since started doing
  [protocol detection](https://istio.io/docs/ops/configuration/traffic-management/protocol-selection/#automatic-protocol-selection).

### Unresolved questions

[unresolved-questions]: #unresolved-questions

### Future possibilities

[future-possibilities]: #future-possibilities
