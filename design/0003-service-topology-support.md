# Title

- Contribution Name: Service Topology Support
- Implementation Owner: [@Matei207](https://github.com/Matei207)
- Start Date: 2020-05-11
- Target Date: 2020-07-27
- RFC PR: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/23)
- Linkerd Issue: [linkerd/linkerd2#4325](https://github.com/linkerd/linkerd2/issues/4325)
- Reviewers: [@grampelberg](https://github.com/grampelberg) [@adleong](https://github.com/adleong)

## Summary

[summary]: #summary

Service topology support will provide topology-related metadata about services that can be used to make more advanced load-balancing and routing decisions. Adding support for this feature will enable the linkerd control plane to keep track of endpoints through endpoint slices, collect topology preferences from services and locality metadata associated with an endpoint, such as the hostname it is on, the zone and the region. Based on topology values from a source host, this metadata can then be used to assign a priority to an endpoint and have more control over routing.

## Problem Statement (Step 1)

[problem-statement]: #problem-statement

[Service topology](https://kubernetes.io/docs/concepts/services-networking/service-topology/) has been introduced as an alpha feature as part of Kubernetes `v1.17`. The new changes allow Kubernetes adopters to make better load balancing decisions (especially in the context of multi-clusters) and potentially achieve lower network latencies by forwarding traffic based on locality-awareness. In conjunction with [endpoint slices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/), service topology enables service owners to define a policy for traffic routing based on source and destination `Node` labels. A service can now specify as part of its `spec` a preference-ordered list of topology keys; the keys are used to sort endpoints when accessing the respective service. Traffic will be directed to an endpoint (and thus node) whose value for the topology key matches the source Node's label for that value, if there are no matching endpoints for that service given that topology key value, the next topology key in the preference-ordered list of the service is considered, and so on. If no match is found, traffic will be rejected. Currently, there are three supported labels that can be used in a service's `topologyKeys` field:
  1. `kubernetes.io/hostname`: routes traffic only to endpoints on the same node;
  2. `topology.kubernetes.io/zone`: routes traffic to endpoints in the same zone;
  3. `topology.kubernetes.io/region`: routes traffic to endpoints in the same region.

Finally, there is also a "wildcard" label `*` that implies any topology. Depending on the use case, service owners can combine the four labels in their preference-ordered list to have better control of the service routing.

Linkerd makes its [load balancing](https://linkerd.io/2018/11/14/grpc-load-balancing-on-kubernetes-without-tears/) decisions, so it does not support this feature out of the box. The current implementation of service topology is done at the `kube-proxy` level which resolves the `ClusterIP` of the service to an endpoint chosen based on the `topologyKeys` field of a service. By adding support for this feature, we can get access to valuable metadata related to the topology of a cluster and make use of it when making our own load balancing decisions. Some benefits of doing this would come in the form of advanced routing decisions in multicluster situations, control over egress costs between regions or better handling of failures. 

Toplogy metadata that is obtained from supporting this feature allows for prioritising endpoints based on locality.

(_Adapted from [#4325](https://github.com/linkerd/linkerd2/issues/4325)_)
(_N.B: I use locality and topology interchangeably here, when used in the same context to me both mean the same thing, apologies in advance if the language barrier created any confusion._)

## Design proposal (Step 2)

[design-proposal]: #design-proposal

### Summary
---

To implement support for this feature, most of the work will be carried out in the `destination service` on the control plane side. Although the feature does not add a lot of changes in terms of the Kubernetes API, several changes have to be made. The first and most obvious is to support the `EndpointSlices` feature that was introduced in Kubernetes `v1.16`, since this will enable endpoints to hold topology metadata.

On the proxy side of all meshed pods, the outbound traffic is load balanced between all available endpoints for a targeted service. Through this change, we will introduce a new priority associated with each endpoint based on its locality. If a service has a topological preference defined in its `spec` through the newly added `topologyKeys`, then the proxy will receive only the endpoints of the highest priority that are available and satisfy the `topologyKeys` preference. In order to determine the priority of endpoints, the proxy will have to send an additional contextual token as part of its service discovery request: the `nodeName` of the host the meshed pod is deployed on. When a proxy will first resolve the FQDN or IP of the outbound service, the `Destination.Get` call will include the topology context token, along with the rest of the payload.

Whenever the destination service receives a request from a proxy, it resolves the FQDN or IP of a service, creates a listener (as an `EndpointTranslator`) and creates a subscription on the service. At first, all available endpoints for the service are sent back as a `WeightedAddressSet`, all subsequent endpoint creations or deletions are propagated through the listener back to the proxy. In order to do this, the destination service makes use of watchers to cache and register any Kubernetes resource changes (such as modified endpoints, newly created endpoints or services that are deleted). Through this feature, every service that is registered by the watchers and cached would include additional `topologyKeys` labels. Likewise, endpoints would now have their topological information stored, this would be done by adding support for the `EndpointSlices` feature. By using the proxy's topology context token (the `nodeName` mentioned previously), the `EndpointTranslator`, which acts as the listener implementation, can now assign priorities to each endpoint and only send the endpoint that satisfy the highest priority required by the service and available in the cluster.

I will address assumptions and suggested changes for each component involved in the rest of this section, and finish with the current constraints that I have identified.

### Proxy
---

The changes to the proxy are minimal. To support service topology, we simply need to know what source of the traffic is.

* **Identifier**: using the downward API, we can set in the `spec` of the pod the `nodeName` the pod is running on as an environment variable. The `nodeName` can then be sent by the proxy as part of its `Destination.Get` call.

* **proxy-api**: make changes to the gRPC service definition/bindings to include the `nodeName` as part of the `Destination.Get` call.


### Destination Service
---

The destination service needs to support the new `EndpointSlices` feature, the `service topology` feature and assign a priority to each endpoint.

* **Endpoint Slices**: has to be supported as a feature, to implement service topology. In order to add support for it, there should be a new watcher in place that registers all `EndpointSlices` resources. I propose that either an `EndpointSlices` watcher is used, or an `Endpoints` watcher, but not both at the same time, so that information does not have to be reconciliated. Furthermore, [constraint-#1](#constraints) addressed the support for this feature; to implement `EndpointSlices` we need to figure out if they are enabled in the cluster.
    - Each endpoint slice can hold 100 endpoints for a given service. Also, each endpoint slice has a generated name based on the service name. To construct the service id in the respective handler function we need to make use of the `service-name` annotation of the endpoint slice rather than its name directly (otherwise each service would be created more than once but under different names).
    - Every address in the address set should now retain topology metadata. Since the order is not specific, it can be incorporated in the labels map that is already part of the struct.
    - To satisfy [constraint-#1](#constraints), we can use a similar approach to how the access of `ServiceProfile` is asserted: by using the `Discovery` client, we can check to see if `EndpointSlices` is a supported resource in the cluster -- if it is, we can go ahead and create a watcher for it.


* **Service topology**: extend the current `servicePublisher` implementation to include the topology preference as described in the service spec. This implies making changes to the handler functions of the watcher. Like `EndpointSlices`, it must satisfy [constraint-#1](#constraints), for services it is easier to handle, however, since no topology routing will be used, and no endpoints will be prioritised if the service lacks the `topologyKeys`.

* **Adding priorities to endpoints**: to assign a priority to an endpoint, we need to have the service's `topologyKeys`, the source node (to get its locality) and the locality of the endpoint itself. The service will have its `topologyKeys` stored when it is cached, and the endpoint will have its locality stored if it is picked up by the `EndpointSlices` watcher. Through the proxy, we get access to the `nodeName`. All of it can be put together in the `Endpoint Translator` which is responsible for subscribing to a service and sending/updating the proxy with the list of endpoints available. Through `service topology`, the listener (translator) will now 
be able to access the source node's topology labels (based on the contextual token received from the proxy) through the Kubernetes client, and filter the list of available endpoints to only send back those that satisfy the top-most priority. In this sense, if a service prefers only pods on its host, the available endpoints will be filtered and only those that satisfy this particular priority will be sent; should there be none, it falls back to the next priority in the service's topological preference. Whenever a new endpoint is created/updated, it is sent to the proxy only if it satisfies the priority rule.
  
* **Destination Service Server**: make changes to the api bindings to accept new topology call from the proxy side.


### Constraints
[constraints]: #constraints
---

* Both `service topology` and `endpoint slices` are not active in a cluster by default. Since they are alpha (and beta) features respectively, they need to be activated by a feature gate. Earlier in the design section I mentioned that we shouldn't use an endpoint watcher and an endpoint slices watcher at the same time. The current suggestion is to verify through the `Discovery` Kubernetes client and see if `EndpointSlices` resource exists in the cluster, similar to how `ServiceProfile` CRD is checked.

* This is a soft constraint and we shouldn't worry about it, but it's worthwhile to mention in case existing adopters want to play with the service topology feature. Service topology and `externalTrafficPolicy` are mutually exclusive. Validation is enforced by Kubernetes when a resource is created/updated, meaning we do not need explicit checks for either `externalTrafficPolicy` or `topologyKeys`.

* Another soft constraint, this is a limitation on the feature adoption.  The [KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20181024-service-topology.md#service-topology-scalability) for service topology proposes once the feature graduates to beta to include a `PodLocator` service to give us the locality of a pod. This would bypass all of the needs to keep track of the source node or to filter pods. It could change the implementation details of this RFC?

* (WIP) this list is not exhaustive and as I collect feedback I might notice more corner cases or constraints

### Prior art

[prior-art]: #prior-art

* [Blog post on feature preview](https://imroc.io/posts/kubernetes/service-topology-en/): this is the only in-depth article that discusses the `service topology` feature. It is a preview and contains some information on how the feature works.
* [Service topology PR](https://github.com/kubernetes/kubernetes/pull/72046)
* [Service topology KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20181024-service-topology.md): motivation of proposal along with implementation details.
* I have seen `service topology` as a feature request for other service meshes (i.e istio), but so far no RFCs/proposals have been made on the topic. Similarily, I went through a few pages of commits that include the phrase "service topology" on GitHub and could not find anything relevant.
* [Priority levels based on locality in envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority)

### Unresolved questions

[unresolved-questions]: #unresolved-questions


### Future possibilities

[future-possibilities]: #future-possibilities

* Other than what was mentioned in the [problem statement](#problem-statement), it might be worthwhile to integrate this with existing CLI commands to get a better view of what is happening in the mesh. For example, `linkerd edges` could display the topology associated with a src/dst pod pair.

