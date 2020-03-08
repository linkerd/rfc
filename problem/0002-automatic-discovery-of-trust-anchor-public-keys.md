- Contribution Name: Automatic discovery of Trust Anchor public keys
- Implementation Owner: 
- Start Date: 2020-03-09
- Target Date: 
- RFC PR: 
- Linkerd Issue: [linkerd/linkerd2#4076](https://github.com/linkerd/linkerd2/issues/4076)
- Reviewers: (Who should review the code deliverable? ex.@olix0r)

# Summary

[summary]: #summary

The `TrustAnchorsPem` provides Linkerd with the public keys of trusted CA roots. These public keys are currently located
within a json value of the `Global` key in the `linkerd-config` ConfigMap. The location of the public keys add an additional
obstacle for automation tooling in providing external root CA public keys.

An example case where externally provided trust anchors would be used is in automated cluster deployments where unique
trust anchors are desirable. The trust anchor public keys would be created during cluster creation and deployed to the cluster
where Linkerd is able to automatically discover them and cert-manager would manage the identity issuer certificate.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

As part of my team's responsibilities, we provide infrastructure for our other engineering teams. We are often standing up
new Kubernetes environments. This process is zero-touch with all infrastructure level components of the cluster already
deployed when handed over for it's intended use.

The cloud infrastructure is deployed using Terraform. As part of this initial deployment phase, Terraform also configures
cluster related resources such as client authentication and RBAC. Terraform registers the new cluster against our application
deployment infrastructure (based around ArgoCD) where additional automation kicks in to deploy the in-cluster infrastructure
such as ingress controllers, monitoring, cert-manager, storage classes, coredns configuration, etc.

We are not able to extend this automation to Linkerd without using the same identity issuer public & private keys for all clusters.
This is not desirable.

If the `TrustAnchorsPem` was separated from the Linkerd global configuration, that would allow us to provide cluster-unique trust anchors
as well as using cert-manager for automatic issuance of the identity issuer certificates. We would use an external cert-manager issuer so that
the trust anchors private keys are not stored in the cluster. These additional resources (trust anchors, cert-manager issuer, etc) are able to be
deployed using our existing automation tooling.

<!--

# Design proposal (Step 2)

[design-proposal]: #design-proposal

**Note**: This should be completed as part of `Step 2`.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear
- It is reasonably clear how the contribution would be implemented
- Corner cases are dissected by example
- Dependencies on libraries, tools, projects or work that isn't yet complete
- Use Cases
- Goals
- Non-Goals
- Deliverables

-->

# Prior art

[prior-art]: #prior-art

I'm not currently aware of any prior art.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- How Linkerd discovers the trust anchors will need to be discussed. Using label selectors against ConfigMaps
  may be an option.

# Future possibilities

[future-possibilities]: #future-possibilities

- If Linkerd is able to discover multiple ConfigMaps containing a trust anchor public key, this could allow for new
  public keys to be phased in which opens the door to rotating the trust anchors when the identity issuer certificate
  is managed by cert-manager.
