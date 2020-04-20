- Contribution Name: Isolated Integration Tests
- Implementation Owner: @kleimkuhler, @alpeb
- Start Date: 2020-03-18
- Target Date: 2020-04-03
- RFC PR: [linkerd/rfc#12](https://github.com/linkerd/rfc/pull/12)
- Linkerd Issue:
  [linkerd/linkerd2#4190](https://github.com/linkerd/linkerd2/issues/4190)
- Reviewers: @grampelberg @olix0r

# Summary

[summary]: #summary

Linkerd's integration testing process will provide developers a way to test
basic Linkerd installations as well as more complex ones, such as
multi-cluster installations and cluster failure scenarios.

A better UX will be available for running all or one of the integration tests.
Additionally, running the tests will be less reliant on existing cluster
state.

With these changes to Linkerd's integration testing process, developers will
be more equip to run Linkerd tests locally without surprises. Adding new tests
for features that require more complex cluster conditions will be possible.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

The Linkerd integration test suite is extensive and effective at catching
bugs, but it does not provide a very positive UX experience to new or existing
Linkerd developers.

It is also unable to provide many options for testing complex cluster
scenarios such as multi-cluster installations and cluster failure scenarios.

The are several reasons for these existing problems and each should be
addressed by the change this RFC is proposing.

### UX

Running specific tests is not currently possible without knowing how the
internal testing scripts work. This can be better so designed so that
developers are able to test specific Linkerd integrations without running the
entire test suite, or looking at the script internals.

After this change, the process for running specific integration tests will be
more perceptible to developers.

### Constraints

The integration test setup checks require that certain conditions are
satisfied by the given cluster. A surprising condition is that no pre-existing
Linkerd installation resource may exist; if it does then it is deleted. These
conditions exist because the test suite is constrained to a single
cluster--running each test in its own namespace.

After this change, the test suite should isolate tests by cluster. This will
allow for each test to require any pre-existing state that is required without
affecting later runs or existing installations.

Another possibility with less constraints on the cluster state is parallel
execution of integration tests.

### Extensibility

Extending integration tests for more complex installation setups is not
accessible.

After this change, the testing process should be extendable in order to test
new features being added to Linkerd. This can include things like checking the
behavior of multi-cluster installations as well as the handling of errors in
cluster failure scenarios.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

**Note**: This should be completed as part of `Step 2`.

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear
- It is reasonably clear how the contribution would be implemented
- Corner cases are dissected by example
- Dependencies on libraries, tools, projects or work that isn't yet complete
- Use Cases
- Goals
- Non-Goals
- Deliverables

# Prior art

[prior-art]: #prior-art

### KinD

Kubernetes isolates it's testing through
[KinD](https://github.com/kubernetes-sigs/kind) which Linkerd already relies
on in several scripts.

Isolating through KinD clusters will provide stronger isolation than namespace
because creating a new a cluster for each test ensures previous runs do not
affect the current installation.

It may not be possible to always rely on KinD if it is not installed, which is
why namespace isolation should still be available.

### k3d (and k3s)

k3d provides similar lightweight Kubernetes clusters that KinD does. While
Linkerd does not have any existing use of k3s, much faster cluster creation
times were observed and it should be equally evaluated for integration test
isolation.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

Should the default process for running all integration tests assume cluster
creation is available?

- Developers would need to have have KinD or k3d installed in order to run the
  tests
- The tests would be less constrained than they currently are, such as dealing
  with existing Linkerd installations
- Tests that require complex cluster states would be runnable by default


Should the cluster creation tool provide the ability for tests to setup their
own clusters?

- If the tool used for cluster creation provided a programmatic way for tests
  to setup their own clusters instead of depending on scripts to set the
  state, the tests could set finer grained cluster properties.
- Injecting cluster failures could be easier if done as part of the test code

# Future possibilities

[future-possibilities]: #future-possibilities

### Parallel execution

With stronger integration test isolation, it could be possible to run
integration tests in parallel. This is currently not possible because only one
control plane can be installed on a cluster at a time, so namespace isolation
is not strong enough.

There are some unresolved questions about how results will be collected, how
many clusters would be run at once, and how a user switches between modes.
