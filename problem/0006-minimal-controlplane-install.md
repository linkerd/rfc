# Support Minimal Control Plane Installation

- @pothulapati
- Start Date: 2020-06-24
- Target Date:
- RFC PR: 
- Linkerd Issue:
  [linkerd/linkerd2#4612](https://github.com/linkerd/linkerd2/issues/4612)
- Reviewers: @alpeb @ihcsim

## Problem Statement

Currently, Linkerd Control-Plane components are tightly coupled in the installation,
even though they can functionally run indepedently.
Components like tap, web, controller, prometheus & grafana
can be optional.
Currently, There are efforts happening to make Grafana & Prometheus Optional.
But the same can be done with more control-plane components like web, tap, etc.
This allows user to have a minimal control-plane install which can only include
proxy-injector, sp-validator, destination and identity services.

This RFC aims to provide flexibility for users to pick and choose what components
they want with their install, while also providing packages like `linkerd-viz`,etc.

### Goals

- Remove tight coupling between control-plane components with installation.
- Provide an incremental installation approach of the control-plane along with a
upgrade story.
- Update Dashboards and CLI to work as per the installed components.

### Non-goals

- A way to install external components as add-ons.

## Open Questions

- How do you deal with Upgrades from 2.8.1?
- Even though we plan to provide ways to install components like web, etc induvidually,
 In most cases we still need to configure other components that these exist which
 implies we will need to do a upgrade like thing rather than a induvidual component
 install. WDYT?
