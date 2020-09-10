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

### What problem are you trying to solve

Users should be able to get the benefit of lower level protocols (L4) when
Linkerd is unable to detect the higher level protocol (L7). This allows users to
get all the benefits they would expect from mTLS, TCP metrics and multi-cluster
routing. To do this, users need to specify whether protocol detection should be
skipped on a per-port basis.

### Requirements

- Preserve the existing port-oriented proxy bypass functionality (`skip-ports`).
- Support mTLS & TCP telemetry for arbitrary TCP protocols, including server-first protocols.
- Support both half-meshed and full-meshed client-server relationships.
- Clients do not need to document/configure the endpoints to which they connect.

### Design

This proposal assumes that Linkerd's outbound proxy implements discovery for
mTLS to arbitrary TCP endpoints.

The destination and injection controllers are modified to honor a new
annotation, which may only be set on ``Pod`` resources:

```yaml
config.linkerd.io/proxy-opaque-tcp: <port-list>
```

Where `port-list` is a comma-separated list of port numbers, port ranges,
and/or port names.

For example:

```yaml
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
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
      annotations:
        config.linkerd.io/proxy-opaque-tcp: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.6
          ports:
            - containerPort: 3306
              name: mysql
```

### Injection Controller

The injector encodes this port list in a proxy environment configuration,
`LINKERD2_PROXY_INBOUND_OPAQUE_PORTS`. When an inbound proxy receives any
inbound traffic on these ports, the connection is forwarded blindly to the
application (without mTLS, etc).

### Destination Controller

The destination controller uses this annotation when assisting the outbound
proxy's protocol discovery via the new `Destination.Strategy` API endpoint
(to be added as a part of TCP work), this annotation instructs the proxy to
skip protocol detection on this stream and to forward it opaquely. This
should be honored for both meshed and un-meshed workloads.

When resolving a `Service` address, the union of all values from all pods in
that service is considered.

When communicating with a meshed endpoint, the destination response includes
a protocol upgrade hint including the remote inbound proxy's port number and
its identity (for use with mTLS).

The outbound proxy then connects to the remote inbound port _directly_ (and
not the original destination port). The endpoint's application port (as
provided by the discovery response and **not** the `SO_ORIG_DST` lookup),
must be preserved and transmitted to the remote proxy so that it can route it
properly. This is transmitted via a new _Linkerd Connection Header_.

#### Multi-Cluster Service Mirror Controller

The service-mirror creates local endpoints that reflect the opaqueness, as
configured in the target cluster. The multi-cluster gateway then uses the
connection header to route the connection through the gateway.

### Proxy handling of inbound-targeted connections

The proxy makes assumptions about traffic that targets its inbound proxy
server (as opposed to traffic that targets an application port, but is
rerouted through the inbound proxy port). A connection that targets the
inbound proxy port:

* Is assumed to originate from other proxy processes;
* MUST use Linkerd's mTLS identity; and
* SHOULD include a _Linkerd Connection Header_ preamble.

Note that outbound proxies _must_ set a connection preamble when targeting an
inbound proxy; but, in order for Linkerd multi-cluster gateways to remain
backwards-compatible, the inbound proxy cannot assume the presence of this
preamble and should revert to protocol detection in such a case.

#### Linkerd Connection Header preamble

The connection header is a preamble that is transmitted from outbound proxies
to remote inbound proxies. This preamble consists of:

* a 12B magic marker;
* a 4B message length; and
* a variable-length protobuf-encoded message.

When the first 16B of a stream match the magic marker, the proxy strips the
entire header from the connection before proxying it to the application.

These messaged are prefaced with a magic marker prefix: `l5d.io/proxy`, or
`0x6c35642e696f2f70726f7879`.

Following that, a 4-byte unsigned integer (Big Endian) message length,
followed by a protobuf-encoded `LinkerdConnectionHeaderV1` message.

This message type includes at least an optional application port, and an
optional detection hint.

When the port is set, it identifies a port on localhost to which the traffic
should be forwarded. When unset (`0`) or set to the value of the inbound or
outbound proxy ports, the proxy handles the connection as it would a gateway
connection. I.e., the proxy r when configured as a multi-cluster gateway)

The detection hint enables the inbound proxy to bypass peek-based protocol detection.

### Examples

`pod/client` proxies a connection to `pod/mysql-0` via `service/mysql`.

The `pod/mysql-0` port is annotated with `config.linkerd.io/proxy-opaque-tcp: 3306`.

#### Meshed Service

- The client application resolves `mysql` to a service IP address, 10.0.0.2,
  and initiates a connection to 10.0.0.2:3306.
- The outbound proxy intercepts this connection via the proxy's outbound port,
  4140.
- The outbound proxy calls `Destination.Strategy` with _10.0.0.2:3306_.
- The destination controller responds indicating that the the stream is opaque
  and detection should be skipped, and that the target is a concrete service
  named `mysql.default.svc.cluster.local`.
- The proxy then resolves `mysql.default.svc.cluster.local:3306` to endpoints
  via `Destination.Get`. These endpoint response include a message indicating
  that a connection header should be used, including the endpoint's inbound
  proxy port.

#### Un-meshed client

- The client connects directly to `mysql-0:3306`
- The server's inbound proxy is configured to forward connections, without
  protocol detection, on port 3306.

#### Un-meshed server

- The client resolves the target as in the full-meshed example.
- The endpoint response does NOT configure the client to use the proxy's
  inbound port with a connection header, nor does it configure mTLS.
- The outbound proxy connects directly to `mysql-0:3306` and forwards the
  request.

### Unresolved questions

[unresolved-questions]: #unresolved-questions

### Future possibilities

[future-possibilities]: #future-possibilities

- Replace multi-cluster gateway logic to avoid per-request routing.
- This new Linkerd Protocol Header can be employed for arbitrary proxy-to-proxy
  communication, including HTTP communication, to provide source-metrics.

## Appendix

### Prior art

[prior-art]: #prior-art

- [Kubernetes issue](https://github.com/kubernetes/kubernetes/issues/40244)
  tracking frustration with the inability to mention application protocols
- [KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20191227-app-protocol.md)
  for placing `appProtocol` on `Service` and `Endpoints`.
- [KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20190603-EndpointSlice-API.md)
  for `EndpointSlice`.
- [Manual
  Selection](https://istio.io/docs/ops/configuration/traffic-management/protocol-selection/)
  is how Istio started doing it. They have since started doing [protocol
  detection](https://istio.io/docs/ops/configuration/traffic-management/protocol-selection/#automatic-protocol-selection).

### Discarded

- Using port names for skipping protocol detection.
- Requiring configuration on both the client and server side.

### Kubernetes `appProtocol`

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

`appProtocol` support can augment the pod-annotation approach in the future
(at least when un-meshed clients are not a concern).

#### Downsides

- This is currently an alpha feature (in 1.18). It is basically unusable at this
  point in time.
- Inbound proxies can not discover this configuration at inject-time, so they
  either cannot proxy connections from un-meshed client or need to perform
  discovery on inbound connections.

### Service Annotations

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  annotations:
    config.linkerd.io/proxy-opaque-tcp: 3306
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

Like `appProtocol`, service annotations are a valid option for the future,
especially if we do not care to configure inbound proxies for un-meshed
clients; but it seems like unnecessary complexity for a starting point.

#### Downsides

- Inbound proxies can not discover this configuration at inject-time, so they
  either cannot proxy connections from un-meshed client or need to perform
  discovery on inbound connections.
