# Capture Egress HTTPS Metrics

- @vaniisgh
- 2020-05-16
- Linkerd Issue: [linkerd/linkerd2#3190](https://github.com/linkerd/linkerd2/issues/3190) & [linkerd/linerd2#2192](https://github.com/linkerd/linkerd2/issues/2192)
- Reviewers:

## Problem Statement (Step 1)

[problem-statement]: #problem-statement

- Outbound traffic is as important as inbound traffic, it is essential we have a
symmetrical system to log, monitor, and provide security to external requests.
- Often times the meshed service communicates with a third-party service suchas github.com,
in such cases the proxy sidecars never see the unencryptedbits. This means that the
rich metrics that Linkerd provides aren't availablefor such communications.
- Egress traffic exist whenever a service makes requests to interacts with athird pary
service. It is common now for such traffic to be encrypted. In suchcases, it only seems
natural that the service mesh should be able to inspect andcontrol the flow of the same.
- This proposal aims to provide a comprehensive solution for dealing with
egresstraffic. Through the following sections, I aim to share my understanding of
an effective, efficient as well as user-friendly solution that enables linkerdproxies
to inspect engress trafic.
- There will be a few changes that will be required on the application code,and
modifications will be made to the way Linkerd2 deals with outbound traffic.

## Design proposal (Step 2)

- The first requirement for the implementaion of the solution is to ensure that all
data between proxy sidecars and corresponding nodes is unencrypted i.e no traffic
in the mesh will be encrypted. This will require some changes on the application side
of the code.
- Now that the proxy sidecars receive unencrypted traffic we can make the gathered
metrics accessable to prometheus or other control plane components.
- Now I propose we should provide a designated service entry pod for the outbound
requests, the proxy sidecars should be configured to foreward traffic for a specific
range of traffic to the service entry pod or have a wildcard resolution of the domain
name and forward traffic to the service entry.
- The service entry pod is then accountable for handling all trafic from external
services, having a service entry seems like a great option because it requires minimal
changes in the current proxy sidecar and allows the user an array of configurations
that can be easily specified in a yml config file. Some of the contents for the
ServiceEntry config could include:
  - hosts: that the cluster is allowed to interact with (or DNS with wildcard prefix)
  - resolution: the way the IP of the request will be resolved
  - protocols supported
  - available endpoints
  - ports that should allow egress traffic
  - certs that will be needed in TLS handshakes or a VirtualService configuration that
  routes trafic to a different port where TLS origination takes place

- Goals
  - To allow linkerd to deal with egress traffic as elegantly as it deals with ingress
  traffic and provide the same kind of metrics.
  - Monitoring outbound traffic with the same tension as inbound traffic, i.e having
  access to the raw body of the outbound requests.
  - Have a single point that deals with all external traffic.

### Prior art

[prior-art]: #prior-art

Many Service Meshes employ policies to resolve and deal with egress traffic, linkerd
too has the functionality to allow such requests to b passed out. This proposal is only
meant to modify behavour so that metrics can be generated and the mesh has access to
all application traffic.

To configure egress traffic linkered can be used by following the steps outlined [here](https://linkerd.io/2017/06/20/a-service-mesh-for-kubernetes-part-xi-egress/).

I have read through Istio, Envoy & Kong 's egress policies and they seem to have an
array of options to deal with egress traffic ranging from service entries to gateways.
They all seem to stress the importance of having a good solution for logging, cross
environment configuration, observability, and HTTP auditing of outbound traffic.There
isint much new and original about the proposal, it just suggests a simple
implementation widely used to deal with egress traffic.

### Unresolved questions

[unresolved-questions]: #unresolved-questions

- Should there be a way to deal with service discovery?

### Future possibilities

[future-possibilities]: #future-possibilities

- redacting egress data and exporing it to a nonproduction environemnt. (so that developers can perform mock data generation & perfoem backtesting)
