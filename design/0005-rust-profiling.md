# Title

Contribution Name: (Profiling to Linkerd Rust controllers)

## Problem

Enabling users to dynamically profile the running application, which can aid
significantly in debugging and diagnostics. This will allow users to identify
and address performance bottlenecks or other issues in real-time and they can
act accordingly.

## Desired Outcome

With the new CLI arguments that enable pprof, users will be able to profile
resource consumption of individual controllers and gain insights into how the
policy-controller is operating.

## Potential Solution

Integrate [pprof-rs](https://github.com/tikv/pprof-rs/blob/master/README.md)
into the Linkerd Proxy-controller: pprof-rs is a profiling tool for Rust that
can be used to generate CPU profiles and heap snapshots. It can be integrated
into the Linkerd control plane to provide dynamic profiling capabilities to
users. This will involve adding the pprof-rs library as a dependency and
modifying the controllers to support pprof-rs. Create a new CLI argument that
tells the controller to enable-pprof profiling. Parse the new CLI argument and
start the ProfilerGuard if the argument is present. Implement a route that
exposes the profiling endpoint using the pprof crate.

## Prior Art

Pyroscope itself is a profiler tool which uses pprof-rs for profiling Rust code.
