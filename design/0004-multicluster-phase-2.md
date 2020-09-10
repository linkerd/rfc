- Contribution Name: multicluster_phase_2
- Implementation Owner: Alex Leong
- Start Date: 2020-06-16
- Target Date:
- RFC PR:
- Linkerd Issue:
- Reviewers:

# Summary

[summary]: #summary

This document describes the design of the next phase of the multicluster work
which will allow source clusters to control which services they import and
generally simplify some of the machinery and anotation surface area.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

The problem is well described in [#4481](https://github.com/linkerd/linkerd2/issues/4481).
To summarize: source clusters should have control over which serviecs the import
from the target cluster.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

I propose the following changes:

* Services in the target cluster will no longer use an annotation to indicate
  that they are exported.  Instead, the source cluster will be able to select
  the services that it wishes to import using a label selector.  This means that
  the following annotations will be eliminated:
  * mirror.linkerd.io/gateway-name
  * mirror.linkerd.io/gateway-ns
  
  This also eliminates arbitary constraint that an exported service needed to be
  fronted by a single gateway.  In reality, the target cluster may have any
  number of gateways and each of these has the ability to route to any service
  on the cluster because of the sharing of trust roots.
* The following annotations are no longer needed on the gateway service because
  they will be moved to a custom resource:
  * mirror.linkerd.io/gateway-identity
  * mirror.linkerd.io/multicluster-gateway
  * mirror.linkerd.io/probe-path
  * mirror.linkerd.io/probe-period
  * mirror.linkerd.io/probe-port
* The `linkerd multicluster install` command no longer installs the service
  mirror controler and instead just installs the components necessary for being
  a target cluster (gateway and service account).
  * It will also install the `link.multicluster.linkerd.io` CRD (see below).
* The `linkerd multicluster link` command will now output the following
  resources:
  * The cluster-credentials secret.  This secret will continue to hold the
    kubeconfig file for access to the target cluster's API.  However, we can
    eliminate the following annotations from the secret since we will be moving
    this configuration into a custom resource:
    * mirror.linkerd.io/cluster-name
    * mirror.linkerd.io/remote-cluster-domain
    * mirror.linkerd.io/remote-cluster-l5d-ns
  * A `link` custom resource.  This `link` will control how the
    service mirror controller operates.  The resource will contain a the
    following fields:
    * target-cluster-name: the name to use for the remote cluster.  this will be
      appended to all mirror services names and to the gateway mirror name.
    * target-cluster-domain: the cluster domain of the target cluster.  this
      will be appended to the authority of requests sent to the target cluster.
    * target-cluster-l5d-ns: the namespace of the linkerd control plane in the
      remote cluster.  Used to fetch the remote configmap when doing the
      healthcheck that compares the local and remote trust roots.
    * cluster-credentials-secret: the name of the above secret to use for target
      cluster API access
    * gateway-address: the address (dns name or IP) and port of the target cluster
      gateway
    * gateway-identity: the identity of the target cluster gateway.
    * probe-spec: the 3 Ps (path, port, period) to use for gateway probes
    * service-selectors: an array of [LabelSelectors](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#labelselector-v1-meta).  These will select which
      services from the target cluster to mirror.
  * The service mirror controller deployment.  This service mirror controller
    container's args will be populated with the name of the service mirror
    `link` resource.

The biggest consequence of the above changes is that we now run one service
mirror controller for each linked target cluster.  This allows to eliminate
the watching of secrets and dynamically creating service watchers.  When the
service mirror controller starts up, it reads the provided `link` resource, loads
the secret specified in the `link`, and then gets to work mirroring services
from the target cluster which match the given selectors.  If the `link` is
updated, the service mirror controller will reload its configuration.

The primary benefit of this approach is that it solves [#4481](https://github.com/linkerd/linkerd2/issues/4481)
by allowing the source cluster to select the servies that it wants to mirror.
It also has many secondary benefits:
* It simplifies the relationship between components by eliminating the coupling
  between exported services and the gateway.
* It centralizes and makes explicit much of the configuration by moving it from
  disparate annotations into a single `link` resource.
* It solves [#4495](https://github.com/linkerd/linkerd2/issues/4495) because
  gateways are no longer defined in terms of exported services
* It solves [#4521](https://github.com/linkerd/linkerd2/issues/4521) by
  getting rid of all of the annotations in the target cluster
* It solves [#4584](https://github.com/linkerd/linkerd2/issues/4584) since users
  will be able to manually configure things like the gateway address in the 
  `link` resource.

It's worth noting that all of the above could be achieved with a single service
mirror controller in the source cluster that detects any `link` reousrces which
are created and does the mirroring for all linked clusters.
However, I think splitting the service mirror controller up so that a separate
instance handles each target cluster has some compelling benefits:
* More fine grained operational control: operators can bounce, upgrade, disable,
  or debug individual service mirror controllers for a specific target cluster
  without worrying about how that will affect the mirroring for other target
  clusters.
* Code simplification: a single Go process no longer needs to maintain nested
  watches where a watch on secrets starts and stops watches on target clusters.
  Each process is responsible for a single target cluster, greatly simplifying
  the code.

The downsides of this approach are that:
* Running more service mirror pods incurs more resource overhead, but this
  should be negligable.
* Running more service mirror deployments incurs more operational overhead
  since these deployments need to be monitored and maintained.  Thankfully,
  Prometheus makes monitoring a collection of processes just as easy as
  monitoring a single process, assuming we use labels in a consistent way.  Any
  observability tooling or dashboards that we build for service mirror
  observability should have no problem dealing with metrics from multiple
  processes.  We can also develop tooling to make upgrading easier, as discussed
  later.

## UX

This proposal makes very few changes to the UX flow.


Install the multicluster gateway, service account, and `link` CRD in both clusters.

```
linkerd --context=target multicluster install | kubectl --context=target apply -f -
linkerd --context=source multicluster install | kubectl --context=source apply -f -
```

Link the clusters.  This will install the cluster credentails secret, `link`
reousrce, and service mirror controller in the source cluster.

```
linkerd --context=target multicluster link --cluster-name=foobar | kubectl --context=source apply -f -
```

This will create:
* secret/cluster-credentials-foobar
* link.multicluster.linkerd.io/foobar
* deployment/service-mirror-foobar
* (also the necessary RBAC for the service mirror controller)

The generated `link` resource will specify the default label selector of
`multicluster.linkerd.io/exported=true`.
Alternatively, users can specify their own label selector:

```
linkerd --context=target multicluster link --cluster-name=foobar -l "export-me-please=true" | kubectl --context=source apply -f -
```

Now, to actually export the service, it's just a matter of applying the correct label:

```
kubectl --context=target label svc/my-cool-svc multicluster.linkerd.io/exported=true
```

The `service-mirror-foobar` service mirror controller in the source cluster will
see that a service in the remote cluster matches its label selector and will
mirror that service by creating a `my-cool-svc-foobar` mirror service in the
source cluster.

Active links can be seen using `kubectl get`.  This replaces the `multicluster gateways` command:


```
kubectl --context=source get links
NAME   NUM SVC   ALIVE   LATENCY_P50   LATENCY_P95   LATENCY_P99
foobar       3    True         275ms         385ms         397ms
```


## Upgrading

To upgrade the service mirror controller, simply run the link step again with a
newer version of the CLI:

```
linkerd --context=target multicluster link --cluster-name=foobar | kubectl --context=source apply -f -
```

Alternatively, we can add an `update` subcommand to `multicluster` which finds
all service mirror controllers running in the source cluster and outputs updated
manifests for all of them:

```
linkerd --context=source multicluster upgrade | kubectl --context=source apply -f -
```

## Unlinking

To unlink a target cluster, we add an `unlink` command which outputs manifests
for the `link` resource, the service mirror controller deployment, the cluster
credentials, and all of the mirror services and gateway mirror for the given
target cluster.  These can be piped to `kubectl delete`.

```
linkerd --context=source multicluster unlink --cluster-name=foobar | kubectl delete -f -
```

(Open question: will this introduce a race condition where deleted mirror services
could be re-created by the service mirror controller before it shuts down?  Should
the service mirror controller instead watch for delete events of its `link` resource
and react to that event by deleting all of its mirror services and then deleting
its own deployment?  Is this too magical?)

## Uninstalling

Uninstalling involves unlinking all target clusters and deleting all the resources
created by `linkerd multicluster install`.

```
linkerd --context=source multicluster unlink --all-clusters | kubectl delete -f -
linkerd --context=source multicluster install | kubectl delete -f -
```

(Open question: can the above two commands be combined into one `multicluster uninstall` command?)

# Out of Scope and Future Work

These items are out of scope because they do not provide a clear enough benefit
or are lower priority than the work described above.

* Treating mirror services as a special case in the destination controller:
  * The destination controller could return the gateway address for lookups against mirror services
  * This means we do not need to create or maintain Endpoints objects for
    mirror services.
  * This avoids needing to maintain redundant gateway information accross all
    mirror services.
  * See https://github.com/linkerd/linkerd2/issues/4535 for more
* Use External Names when Gateway Address is a Hostname:
  * When the gateway address is a hostname, we have the service mirror controller
    resolve the hostname (and periodically re-resolve it) and use the resolved
    IP address in mirror services and the gateway mirror.
  * In order to move this DNS resolution into the proxy we could update the
    destination proxy-api to allow destination lookups to return hostnames.
  * By making the gateway mirror service into an external name service, we could
    use the gateway address hostname as the gateway mirror external name.  The
    destination controller would then be able to return this hostname for queries
    on the gateway mirror and the proxy would be responsible for resolving DNS.
  * If the mirror services were also external name services OR if they were
    special cased endpointless services as described in proposal 2, then the
    destination controller could also return the gateway address hostname for
    queries on the mirror services and the proxy would be responsible for
    resolving DNS.
  * See https://github.com/linkerd/linkerd2/issues/4590 for more
* More aggressive reconciliation:
  * See https://github.com/linkerd/linkerd2/issues/4575 for more