# Simplifying state management in Linkerd

- Contribution Name: Simplify State Management
- Implementation Owner: @Pothulapati
- Start Date:
- Target Date: 2.9 release
- RFC PR: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/0000)
- Linkerd Issue:
  [linkerd/linkerd2#0000](https://github.com/linkerd/linkerd2/issues/0000)
- Reviewers: @adleong @alpeb

## Summary

[summary]: #summary

Currently, Linkerd state in the `linkerd-config` ConfigMap as flags parsed from `installOptions`. This has become counter intuitive after we added Helm support as `Values` struct became our main source of truth (for rendering, etc), and causing back and forth conversions between `installOptions` and `Values`. This PR aims to remove the unnecessary complexity in managing state and simplify things.

## Problem Statement (Step 1)

Currently, `linkerd-config` configMap is not only used for upgrades but also by multiple components like Proxy Injector, CLI, etc to understand the current cluster/linkerd state. This has the following problems:

- As `Values` is not the source of truth, hacks [had to be/being](https://github.com/linkerd/linkerd2/pull/4864#pullrequestreview-467046753) [made](https://github.com/linkerd/linkerd2/pull/4569) to make state (outside of flags, etc) be stored for helm in some manner for Linkerd check.
- No clear definition on which components use what configurations.
- Lots of code debt/confusion to support the above use-cases while working similarly with Helm on the UX.

## Design proposal (Step 2)

[design-proposal]: #design-proposal

This proposal aims to have a common way to store state both for the Linkerd2 and Add-On Charts independently. This means that, each add-on/package gets its own configuration secret which is used for upgrades and to perform other CLI functions i.e Check, etc. Current CLI functions, etc will be updated to depend on this secret to understand the cluster state.

### For Linkerd

This new overridden values struct contains the details directly from the `installOptions` passed through the CLI flags, and will be overridden.

Currently, `linkerd-config` is used by most components to know cluster configuration(like `cluster-domain`, etc) and components like proxy-injector needing a lot more. As the current `linkerd-config` will be removed, This information has to be passed to the components through flags. (flags are chosen over configMap, so that we don't fall into the trap of having more configuration places). This makes each component get its configuration fully during the install/upgrade time through flags, without relying on external structures.

#### Required Components Configuration

- **Destination**
  - Requires `TrustDomain`, `ClusterDomain`.
- **Identity**
  - Requires `Namespace`, `IdentityContext`.
- **Public API**
  - Requires `ClusterDomain`
- **Web**
  - Requires `ClusterDomain`.
- **Tap**
  - Requires `TrustDomain`.
- **Proxy Injector** (Has to be handled specially, as a [lot of](https://github.com/linkerd/linkerd2/blob/main/controller/gen/config/config.pb.go#L92) [configuration is needed](https://github.com/linkerd/linkerd2/blob/main/controller/gen/config/config.pb.go#L192))
  - Requires the `pb.Proxy` field to understand the default proxy fields, and also uses `pb.Global`.
  - IMO, we should have a configMap just for proxy-injector to know this information, where we just marshal `.Values.Configs` for it to read as it does now.

Except for proxy-injector, All the other components can get their information through CLI flags.

#### Workflows

- **Install**
  - Read override configuration through `installOptions` CLI flags.
  - Create a `ValuesOverride` based of the `installOptions`. (This `valuesOverride` will contain nil values with the current state and work will be done to not marshal nil values, see blockers below).
  - Read the default Values from `values.yaml`.
  - Apply the `ValuesOverride` to the default Values to produce a fully hydrated Values.
  - Render the charts using the fully hydrated Values.  Included in the charts is a new configMap which contains the marshalled `ValuesOverride`. (This can be done by creating a separate configMap resource in the code and rendering it out).

- **Upgrade**
  - Read the `ValuesOverride` `from the cluster:
    - if upgrading from a version that used the linkerd-config *configMap* then the CLI will read the linkerd-config configMap and the issuer secret and do some conversion to turn this all into a ValuesOverride
    - if upgrading from a version that used the new configuration resource then the CLI can just read the ValuesOverride directly from it.
  - Update the `ValuesOverride` from the upgrade flags.
  - Read the default Values from `values.yaml`.
  - Apply the `ValuesOverride` to the default Values to produce a fully hydrated Values.
  - Render the charts using the fully hydrated Values.  Included in the charts is a new configMap which contains the marshalled updated `ValuesOverride`.

#### Populating `installOptions` into `valuesOverride`

As `values` is a different type than `InstallOptions`, we have to individually map which fields parse into which other corresponding fields. As values are finally rendered as a `map[string]interface{}`, we can directly create the map if there is any problem.

**Expected Blockers**
- **Empty values not being omitted**
  - Currently, `Values` struct is not configured to not marshal empty fields and spits out null value fields making the configuration confusing.
  - This can be fixed by adding `omitempty` json field tag to all fields, and requires changes to templates to change expectations of some fields. See [#4903](https://github.com/linkerd/linkerd2/pull/4903)
- **Zero Values of Fields**
  - The problem here is that when a struct is marshalled, it gives back fields only when they **not are  zero values**. This could be a problem for fields like boolean, etc.
  - This does not seem like a big problem in our codebase, in our most cases **zero values are defaults**. (like with most of our Toggle flags are `EnableX`, meaning they are false by default and thus don't/will not be marshalled.)

### For Add-Ons

Currently, All the  **overridden config** is stored into `linkerd-config-addons`, where the configuration for add-ons is stored by directly marshalling the add-on objects. This is not used in Helm in any manner, but helps the CLI to know the add-on state to perform checks, etc.

Though this served us fine but we felt that it would be better to follow the same new pattern as that of Linkerd and have each add-on get its own configuration secret where its **overridden configuration** is stored, and retrieved during upgrades.

### Linkerd Check

Currently, Linkerd check needs `config.All` to work, and uses this to understand the cluster state i.e ha, issuer cert checks, and even for multi-cluster to check cluster anchors.

Linkerd Check can be updated to not use the `config.All` for checks, and instead read the default `Values` struct with `valuesOverride` on top, and use that to understand the cluster state.

#### Helm Support

Linkerd CLI checks for Helm, were always a bit confusing because some checks **depended** on `install` flags, etc of `linkerd-config` which are not populated with Helm.

The CLI can be updated to get the overridden configuration of Helm by reading its release secret and override that on top of the defaults, and use that to understand the cluster state. This process is same as that of the CLI install, except the CLI has to be aware of how to understand how to retrieve and decode the helm install secret.

### First Upgrade

To felicitate the first upgrade, The current upgrade logic where we use `linkerd-config` should not be removed, and should work as expected. Additional, Code will be present which checks if the new configMap is present, and overrides it on top.

Once the upgrade has been performed, The install would have a updated `linkerd-config` (with no cli flags) and the new configMap, and from the next release the previous upgrade logic can be cleaned up.

### Implementation Plan

- Update the CLI code to render a additional secret which has all the overridden configuration including the issuer tls secrets, etc.
- Update the control-plane components to be able to get most configuration through flags, and remove dependency on the current linkerd cm.
- Remove the generation of `linkerd-config` cm.
- Update the upgrade flow to use the secrets, and give it priority over the current `linkerd-config`.
- Update the check workflow to understand the current cluster state from the new format.
- Update Add-ons individually to have its own configuration resource, and update CLI for upgrades and checks accordingly.

### Unresolved questions

[unresolved-questions]: #unresolved-questions

- Why do conversion between `installOptions` and `Values`, Can we use something like [viper](https://github.com/spf13/viper) and have `values` be directly used to supply configuration through a file and CLI flags?
- Should we skip marshalling tls key fields into the `valuesOverride` so that we don't have to maintain redundant information? and also helps us only a configMap instead of a secret.
