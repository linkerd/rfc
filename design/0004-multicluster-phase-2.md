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
generally simplify some of the machinery and anotation surface area.  This
document actually describes 3 independant proposals.  These proposals are not
mutually exclusive and we are free to implement some, all, or none of them.

# Propsal 1: Service Selctors

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
  they will be moved to a config file:
  * mirror.linkerd.io/gateway-identity
  * mirror.linkerd.io/multicluster-gateway
  * mirror.linkerd.io/probe-path
  * mirror.linkerd.io/probe-period
  * mirror.linkerd.io/probe-port
* The `linkerd multicluster install` command no longer installs the service
  mirror controler and instead just installs the components necessary for being
  a target cluster (gateway and service account).
* The `linkerd multicluster link` command will now output the following
  resources:
  * The cluster-credentials secret.  This secret will continue to hold the
    kubeconfig file for access to the target cluster's API.  However, we can
    eliminate the following annotations from the secret since we will be moving
    this configuration into a configmap:
    * mirror.linkerd.io/cluster-name
    * mirror.linkerd.io/remote-cluster-domain
    * mirror.linkerd.io/remote-cluster-l5d-ns
  * A service-mirror-config ConfigMap.  This ConfigMap will control how the
    service mirror controller operates.  The ConfigMap will contain a config
    file with the following fields:
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
    config ConfigMap.

The biggest consequence of the above changes is that we now run one service
mirror controller for each linked target cluster.  This allows to eliminate
the watching of secrets and dynamically creating service watchers.  When the
service mirror controller starts up, it reads the provided configmap, loads
the secret specified in the configmap, and then gets to work mirroring services
from the target cluster which match the given selectors.  If the configmap is
updated, the service mirror controller will reload its config.

The primary benefit of this approach is that it solves [#4481](https://github.com/linkerd/linkerd2/issues/4481)
by allowing the source cluster to select the servies that it wants to mirror.
It also has many secondary benefits:
* It simplifies the relationship between components by eliminating the coupling
  between exported services and the gateway.
* It centralizes and makes explicit much of the configuration by moving it from
  disparate annotations into a single config file.
* It solves [#4495](https://github.com/linkerd/linkerd2/issues/4495) because
  gateways are no longer defined in terms of exported services
* It solves [#4521](https://github.com/linkerd/linkerd2/issues/4521) by
  getting rid of all of the annotations in the target cluster
* It solves [#4584](https://github.com/linkerd/linkerd2/issues/4584) since users
  will be able to manually configure things like the gateway address in the 
  config file.

# Proposal 2: Dumber Service Mirror Controller Reconciliation

TODO

# Proposal 3: Endpointless Mirror Services and ExternalName Gateway Mirrors

TODO
