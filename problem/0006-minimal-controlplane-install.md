# Minimal Control Plane

- Contribution Name: minimal_control_plane
- Implementation Owner:  
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Target Date: (is there a target date of completion?)
- RFC PR: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/0000)
- Linkerd Issue:
  [linkerd/linkerd2#0000](https://github.com/linkerd/linkerd2/issues/0000)
- Reviewers: (Who should review the code deliverable? ex.@olix0r)

## Summary

[summary]: #summary

An incremental approach to installing Linkerd, with support for additional dependencies.

## Problem Statement (Step 1)

[problem-statement]: #problem-statement

Linkerd's control plane components can work independently(example: Tap, etc)
but currently, the installation logic is tightly coupled i.e in a single chart
(except for prometheus and grafana). This has the following problems:

- Components can't be skipped during installation.
- No incremental way of installing components as required.

Once the proposal is implemented, Users will be able to

- Use the Linkerd CLI or Helm to install a minimal control plane with only the
bare essentials of functionality.
- Use the Linkerd CLI or Helm to install addon charts such as Linkerd-viz to add
functionality to the minimal control plane independently.
- Use `linkerd install` or full control-plane to install a complete Linkerd control
plane including full observability, just like they can today.

## Design proposal (Step 2)

[design-proposal]: #design-proposal

### Independent Packages

The plan is to divide the current chart into the following helm packages which
can be installed independently, incrementally.

- **Main Control Plane**
  - This Helm package will be the most fundamental part of the Linkerd installation.
    Most of the current Linkerd’s configuration that are exposed through the Helm
    and CLI will be part of this chart.
  - contains **destination, identity, proxy-injector, public-API**

- **Prometheus**
  - Contains the linkerd-prometheus chart with the relevant configuration needed
    to scrape Linkerd metrics out of the box.
  - contains **prometheus**
  - depends on `main control plane`.

- **Linkerd Viz**
  - Contains all the components that provide visualisations and more monitoring
    for Linkerd
  - contains **grafana, web, tap**
  - depends on `prometheus`

- **CNI**
  - Contains the Linkerd CNI daemonset.
  - depends on `minimal control plane`.

- **Tracing**
  - Contains the tracing components in Linkerd
  - contains **OC Collector, Jaeger**
  - depends on `main control plane`.

- **Multicluster**
  - Contains the Gateway resources
  - contains **Service Mirror**
  - depends on `main control plane`.

- **Multicluster Link**
  - Contains the Service Mirror resource
  - depends on `main control plane`.

### Configuration

Induvidual packages means that, some configuration (Ex: resources, proxy
resources, etc)
are now available across different packages. Care has to be taken so that the
configuration are similar across packages and intutive for the user.

Each package gets its own `<chart-name>-config` where its state is stored and
will be used for upgrades.

### User Experience

#### CLI

CLI commands will be provided with configuration flags specific to each package,
which can be installed by running them induvidually one after another.
Each `install <chart>` will have `--config` and `--set` flags to pass configuration
like the helm way.

- Minimal Install

```bash
# install the minimal control-plane
linkerd install minimal --namespace l5d
```

- Installing Add-Ons

```bash
# install the prometheus
linkerd install prometheus

# install linkerd viz
linkerd install viz --prometheus-url "prometheus.monitoring:3000"
```

Through CLI, There will be relevant checks performed before installing the package
to make sure the minimal package is installed, and the proxy injector works correctly.

- Upgrades

Each induvidual add-on must be upgraded induvidually with the same upgrade command
but by calling induvidual add-on name. Just like the current case,
during upgrades values can be overriden by setting flags.

```bash
# upgrade the minimal control-plane
linkerd upgrade minimal --namespace l5d

# upgrade the prometheus
linkerd upgrade prometheus

# install linkerd viz
linkerd upgrade viz --prometheus-url "prometheus.monitoring:3000"
```

### Helm

- Minimal Install

The same packages can be installed through Helm one after another.

*Tenative Example*:

```bash
# install the minimal
helm install l5d linkerd-minimal --set namespace l5d
```

- Installing add-ons

```bash
# install the prometheus
helm install l5d-prom linkerd-prometheus

# install linkerd viz
helm install l5d-viz linkerd-viz --set prometheus-url "prometheus.monitoring:3000"
```

Through Helm, There is no direct way to have checks, and make the add-on install
wait. We can have a pre-install hook, but it seems too complicated as the job
runs on the cluster, etc. Instead we can have documentation that mentions that
care should be taken to wait for the minimal install is completed and proxy injector
is working before installing add-ons, another thing to note is even if the packages
are not injected it should not change the functionality but its highly advised
to have them injected.

- Upgrades

Each induvidual chart has to be upgraded induvidually. Overriding and other features
available through Helm can be used here.

```bash
# get latest version of charts
helm repo update

# upgrade the minimal control-plane
helm upgrade minimal --reuse-values --set namespace l5d

# upgrade the prometheus
helm upgrade prometheus --reuse-values

# install linkerd viz
helm upgrade viz  --reuse-values
```

Both through CLI and Helm, All the charts default values will be configured to work
out of the box with each other. If any of these values are changed/configured,
Its for the user to make sure the relevant values are configured in other charts
(if needed).

Except for the minimal package, everything else will not have the proxy injected
by default and will go through the proxy injector.

### Proxy Injection

Proxy Injector will be updated to also inject control-plane components even if
they are in the `controlPlaneNamespace` or others.

There will also be a new CRD which makes the proxy injector more extensible.
This is used for cases like tap, tracing, etc which would want the proxy injector
to be injected with some specific additional configuration

The CRD contains a `jsonPatch` field which follows the widely used [jsonPatch](http://jsonpatch.com/),
These jsonPatches are applied to the Pod by the Proxy Injector based on the request
and label selectors. `JsonPatch` is used instead of having things like `env`, etc
so that this use-case is future proof, and can act on all fields of a `podSpec`.

Having plain jsonPatches may not be enough as we then might have to have a new patch
even for the slightest variation i.e to set a env or not.
Templated jsonPatches can be used instead. The values generated during injection
(which is an override on top of `linkerd-config` based on annotations) along with
the labels can be used to render the final JsonPatch and apply it.

The proxy injector will be updated to watch on the below CR, and maintain this
state internally, and perform injections with

#### CRD

```go
type InjectorConfigSpec struct {
    JsonPatch       String
    Selector        metav1.LabelSelector
}
```

If there is any problem during the applying of a jsonPatch, relevant events are
sent to the pod, along with logs.

**Note:** It's important to note that as the install of CR, can go along with the
component (ex: tap), It might be the case that it might not inject the component
in the add-on with that configuration.

### Full Installation

Though Incremental installation seems like a better way to install Linkerd, The
current full install of Linkerd will have to be supported both for backward
compatability and also simplicty reasons.

The problem to make this work, is that if all the add-on charts depend on the proxy
injector to do the injection, how can we package them all in a single package?

One easy way to do this is to have a template field based on which we toggle the
usage of partials sub-chart dependency (where the injection logic is.). The `partials`
will be there to serve injection for minimal control plane components. This way,
injection during install can be enabled for Full Installs, and thus not dpeending
on proxy injector. This would solve the problem both with CLI and Helm.

The below are some other ways through which this can be done.

#### CLI

Through CLI, The full installation can be performed by running the usual
`linkerd install` command that we know and love. This should be possible by passing
the package resources through the manual inject function internally,
and get all injected resources which are submitted as output.

Though we are installing all the charts at a time (except tracing), This will not
add or remove any extra resources i.e configmaps, etc, and acts just like a
grouping mechanism.

#### Helm

For helm, We can have the current full chart i.e `linkerd2` chart changed into a
*grouping* chart with the above mentioned charts as dependencies without any
templates, but a `values.yaml` file where users can override default configuration.

### High Availability

The high availability story does not change much, as each chart gets its own
`--ha` (through CLI) and `values-ha.yaml` (through Helm), to override default
configuration for HA.

This also provides an extra benefit of mixing and maching HA and Non-HA packages
based on their usage and requirements.

### Upgrade workflow

Using the current upgrade workflow means that each chart gets its own configMap,
which stores the state to be used for upgrades. Rather than using a new protobuf
type to store these configuration, We can instead just marshall out the configuration
directly for most packages.

As of 2.9, We do the same for the isntall and this can be replicated to all the charts
i.e each chart having its own `-config` configMap.

### Uninstall

Just like `linkerd install <package>`, the same sub-commands can be supported
under `uninstall` to render out components.

This is very straight forward through Helm as releases can be deleted induvidually.

### What changes for existing users

For the current Linkerd CLI users, the install should not change much, and
backward compatibility can be guranteed for users on supported flags in the
default installation.

For current Helm users, Good amount of configuration fields will move away from the
minimal package and into other packages. The documentation will be updated mentioning
the same, and caution will have to be taken for users to update older configuration
under new fields.

### Implementation Plan

- Move away from `linkerd-config-overrides`, and make each store its state in their
own cm or secret and make CLI upgrades feasible.
- Update CLI to load add-on charts i.e prometheus, grafana, etc induvidually
(rather than as a single one), and de-couple their install/upgrade from the
linkerd2 chart.
- Move control plane components into independent packages, while adding induvidual
add-on subcommands to `linkerd install/upgrade`. (Linkerd2 chart will still be
using the current `linkerd-config`). These independent packages would follow the
current release,build mechanisms.
- Move commands that depends on a specific package as subcommands as chart
specific commands. Ex: `linkerd viz routes`.
- Update Proxy Injector with new CRD based model.
- Remove dependency on partials i.e injection at install to all add-on packages
and create Injector CR's for the same.
- Add Checks between installs for the CLI.
- Update Documentation and Chart README.

### Open Questions

- Should the jsonPatches in `InjectorConfig` be exhaustive i.e full injection config?
or they should build on the main one which performs the default injection.
- Should the patch be the one to decide on what pod requests does it take or should
the pod have some label to denote that?
