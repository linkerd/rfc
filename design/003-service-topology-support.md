# Title

- Contribution Name: Service Topology Support
- Implementation Owner: [@Matei207](https://github.com/Matei207)
- Start Date: 2020-05-11
- Target Date: 2020-07-27
- RFC PR: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/0000)
- Linkerd Issue: [linkerd/linkerd2#4325](https://github.com/linkerd/linkerd2/issues/4325)
- Reviewers: 

## Summary

[summary]: #summary

Service topology support will provide topology-related metadata about services that can be used to make more advanced load-balancing and routing decisions. Adding support for this feature will enable the linkerd control plane to keep track of endpoints through endpoint slices, collect topology preferences from services and locality metadata associated with an endpoint, such as the hostname it is on, the zone and the region. Based on topology values from a source host, this metadata can then be used to assign a weight to an endpoint and have more control over routing.

## Problem Statement (Step 1)

[problem-statement]: #problem-statement

[Service topology](https://kubernetes.io/docs/concepts/services-networking/service-topology/) has been introduced as an alpha feature as part of Kubernetes `v1.17`. The new changes allow Kubernetes adopters to make better load balancing decisions (especially in the context of multi-clusters) and potentially achieve lower network latencies by forwarding traffic based on locality-awareness. In conjunction with [endpoint slices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/), service topology enables service owners to define a policy for traffic routing based on source and destination `Node` labels. A service can now specify as part of its `spec` a preference-ordered list of topology keys; the keys are used to sort endpoints when accessing the respective service. Traffic will be directed to an endpoint (and thus node) whose value for the topology key matches the source Node's label for that value, if there are no matching endpoints for that service given that topology key value, the next topology key in the preference-ordered list of the service is considered, and so on. If no match is found, traffic will be rejected. Currently, there are three supported labels that can be used in a service's `topologyKeys` field:
  1. `kubernetes.io/hostname`: routes traffic only to endpoints on the same node;
  2. `topology.kubernetes.io/zone`: routes traffic to endpoints in the same zone;
  3. `topology.kubernetes.io/region`: routes traffic to endpoints in the same region.

Finally, there is also a "wildcard" label `*` that implies any topology. Depending on the use case, service owners can combine the four labels in their preference-ordered list to have better control of the service routing.

Linkerd makes its [load balancing](https://linkerd.io/2018/11/14/grpc-load-balancing-on-kubernetes-without-tears/) decisions, so it does not support this feature out of the box. The current implementation of service topology is done at the `kube-proxy` level which resolves the `ClusterIP` of the service to an endpoint chosen based on the `topologyKeys` field of a service. By adding support for this feature, we can get access to valuable metadata related to the topology of a cluster and make use of it when making our own load balancing decisions. Some benefits of doing this would come in the form of advanced routing decisions in multicluster situations, control over egress costs between regions or better handling of failures. 

Toplogy metadata that is obtained from supporting this feature allows for weighting endpoints based on locality.

(_Adapted from [#4325](https://github.com/linkerd/linkerd2/issues/4325)_)
(_N.B: I use locality and topology interchangeably here, when used in the same context to me both mean the same thing, apologies in advance if the language barrier created any confusion._)

## Design proposal (Step 2)

[design-proposal]: #design-proposal

### Summary
---

To implement support for this feature, my main assumption is that most of the work will be carried out in the `destination service` on the control plane side. Although the feature does not add a lot of changes in terms of the Kubernetes API, several changes have to be made. The first and most obvious to me is supporting the `endpointSlices` feature that was introduced in Kubernetes `v1.16`, since this will enable endpoints to hold topology metadata.

All meshed pods have a proxy that will first establish all outbound endpoints of a service the pod is targeting. Through this change, we will introduce a new weight to each endpoint based on locality, which can afterwards be used to add subsequent features that build on locality-awareness. I propose that the weights be added to each endpoint on the proxy side. To ensure backwards compatibility, there should be a separate gRPC call that can be made to the destination service to get a service's topology preference, and the host's (node) topology labels. The call would include the FQDN/IP of a service, and a topology identifier (similar to the context namespace used for service profiles). 

The destination service will resolve the FQDN/IP in its usual fashion; it will now also hold in cache the endpoint slices of the service (which have topology metadata). As part of every service that is cached, the destination service should add (if they exist) the topology keys in the order of preference of the owner. Based on the locality identifier of the source, the topology preference of the service, and the available endpoints, each endpoint can have a topology weight assigned to it. The proxy receives its endpoint(s) as usual and can assign weights based on the information it receives from the separate topology call and the topology metadata associated with each endpoint(s).

I will address assumptions and suggested changes for each component involved in the rest of this section, and finish with the current constraints that I have identified.

### Proxy
---

To make use of service topology, we need to know what the source of the traffic is. The proposed changes for the proxy focus on adding support for the topology identifier and support for the new weight associated with an endpoint.

* **Identifier**: after spending some time looking through the proxy and its interaction with the destination service, I found that it sends a [context token](https://github.com/linkerd/linkerd2-proxy-api/blob/master/proto/destination.proto) which is used to infer the namespace when associating a service profile with a service.
    1. At injection time, put a new environment variable in the pod spec with the name of the node the pod is scheduled on. This would work similarily to how the context namespace is set.
    2. In the proxy, get the value from the environment variable.

* **Endpoint weights**: in `api-resolve` crate change the metadata to support endpoint topology labels. I propose the topology labels are either set as an explicit, separate value, or be included as labels as part of the existing `labels` map.  

* **Other changes**: add support for the gRPC call mentioned in the summary, to retrieve topology preference associated with a service and topology labels of the node the proxy is running on.

### Destination Service
---

The destination service needs to support the new `endpointSlices` feature, the `service topology` feature and make changes to the `gRPC` api server to support the new proxy call.

* **Endpoint slices**: to implement support for this, there should be a new watcher in place to register all `endpointSlices` resources. In practice, there might be some complications. The destination api should use either endpoint slices or endpoints, both at the same time would result in useless calls and duplicate information (I assume). I will discuss this a bit more in depth in the [constraints subsection](#constraints). 
    - Each endpoint slice can hold 100 endpoints for a given service. Also, each endpoint slice has a generated name based on the service name. To construct the service id in the respective handler function we need to make use of the `service-name` annotation of the endpoint slice rather than its name directly (otherwise each service would be created more than once but under different names).
    - Every address in the address set should now retain topology metadata. Since the order is not specific, it can be incorporated in the labels map that is already part of the struct.

* **Service topology**: extend the current `servicePublisher` implementation to include the topology preference as described in the service spec. This implies making changes to the handler functions of the watcher.


* **Adding weights to addresses**: to add weights to an address, we need access to the service topology, the source node (its topology labels) and the endpoint topology. If the source node name is sent by the proxy, the node's topology values can be trivially accessed using the `K8s API`. There are two edge cases that exist under my assumptions however, which makes weighting the addresses a bit harder than it should be on the destination service side. First, two source proxies can listen on the same service, because of this, I think it is infeasible to add the weighting logic on subscription. Second, we have to also consider updates -- while on update we can weight a endpoint based on the current endpoints that already exist in a stream, there is a high chance there might be no available endpoints, so inferring weights from existing endpoints when updating also does not sound feasible (even if there is a 1:1 relation between a source and a service stream).
  * My suggested approach here is to assign weights to endpoints on the proxy side. This could be done by having a one time `GetTopology()` call to the destination service with the service FQDN/IP and the topology token. The destination service can send to the proxy its node's topology labels (based on the sent token), and the service topology preference. This can be kept in memory and calculate weights for endpoints on the fly. The advantage here is that updates will be easy to process and it would ensure backwards compatibility by being slightly isolated from the rest of the discovery api. Endpoints would arrive from the stream would their own topology labels, since those would be set by the watcher.
  
* **Destination Service Server**: make changes to the api bindings to accept new topology call from the proxy side.



### Constraints
[constraints]: #constraints
---

* Both `service topology` and `endpoint slices` are not active in a cluster by default. Since they are alpha (and beta) features respectively, they need to be activated by a feature gate. Earlier in the design section I mentioned that we shouldn't use an endpoint watcher and an endpoint slices watcher at the same time. My suggestion is to include a feature flag in the control plane config map. We can access the config map when we create a new server, based on this, when we fire up the watchers we can choose based on the flag which watcher to create.

  Being able to tell from the cluster state if a feature flag is active (i.e service topology and endpoint slices) would greatly simplify this and reduce the need for configuration. From what I know, the only way to check for a feature flag is to ssh on the master node -- this is pretty cumbersome not to mention unfeasible. So I think the best way to go about it is to explicitly set it in the config map and have support for endpoint slices off by default. The flag would only tell us which type of endpoint watcher to create. The endpoint topology logic will be in the dedicated handler functions, and when we add a service in, if its spec does not include any topologyKeys (which it won't since the feature is off) then they will simpliy not be added. This will not have a side effect on how the current implementation of the destination service works.

* This is a soft constraint and we shouldn't worry about it, but it's worthwhile to mention in case existing adopters want to play with the service topology feature. Service topology and `externalTrafficPolicy` are mutually exclusive. Validation is enforced by Kubernetes when a resource is created/updated, meaning we do not need explicit checks for either `externalTrafficPolicy` or `topologyKeys`.

* Another soft constraint, this is a limitation on the feature adoption.  The [KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20181024-service-topology.md#service-topology-scalability) for service topology proposes once the feature graduates to beta to include a `PodLocator` service to give us the locality of a pod. This would bypass all of the needs to keep track of the source node or to filter pods. It could change the implementation details of this RFC?

* (WIP) this list is not exhaustive and as I collect feedback I might notice more corner cases or constraints

### Prior art

[prior-art]: #prior-art

* [Blog post on feature preview](https://imroc.io/posts/kubernetes/service-topology-en/): this is the only in-depth article that discusses the `service topology` feature. It is a preview and contains some information on how the feature works.
* [Service topology PR](https://github.com/kubernetes/kubernetes/pull/72046)
* [Service topology KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/20181024-service-topology.md): motivation of proposal along with implementation details.
* I have seen `service topology` as a feature request for other service meshes (i.e istio), but so far no RFCs/proposals have been made on the topic. Similarily, I went through a few pages of commits that include the phrase "service topology" on GitHub and could not find anything relevant.

### Unresolved questions

[unresolved-questions]: #unresolved-questions

- There are two unresolved questions as of now, both related to the same topic.
  * What would be the best way to set weights to the endpoints, and where should this happen? Based on my assumptions, I found out the best place to be on the proxy side, but it might be that I am missing something.
  * Similarily, how should the weights be set? To avoid configuration, I think it should not be up to each adopter to set weights but rather have them derived from the topology preference and endpoints availability. From my (limited) optimisation experience, this can be done by trial and error but I'm sure there are better ways of agreeing on what weights should be attributed to each endpoint.

### Future possibilities

[future-possibilities]: #future-possibilities

* Other than what was mentioned in the [problem statement](#problem-statement), it might be worthwhile to integrate this with existing CLI commands to get a better view of what is happening in the mesh. For example, `linkerd edges` could display the topology associated with a src/dst pod pair.

