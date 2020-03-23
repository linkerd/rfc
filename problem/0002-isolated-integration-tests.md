- Contribution Name: Isolated Integration Tests
- Implementation Owner: @kleimkuhler, @alpeb
- Start Date: 2020-03-18
- Target Date: 2020-04-03
- RFC PR: [linkerd/rfc#12](https://github.com/linkerd/rfc/pull/12)
- Linkerd Issue:
  [linkerd/linkerd2#4190](https://github.com/linkerd/linkerd2/issues/4190)
- Reviewers: @grampelberg

# Summary

[summary]: #summary

It will be easier to isolate integration tests by more than just namespace.
With stronger isolation, running the tests will more reliable since they will
not be as dependent on a complete cleanup of previous test runs. Additionally,
running individual integration tests will more accessible; currently this
requires knowledge about how the internal scripts work.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

There is currently one way scripted way to run Linkerd integration tests.
There are a few problems:
- It isolates each run by namespace, relying on a complete cleanup of the
  current test in order to run the next one
- As a consequence of isolating by namespace, it deletes any existing Linkerd
  installation if any resources are present
- It runs all the integration tests (upgrade, helm, helm upgrade, deep, and
  external issuer) and does not provide a way to run any one individually

Once the change is implemented, there will be a scripted way to isolate
integration tests by more than namespace, as well as individually run them
without existing knowledge of internal scripts.

Isolation by namespace is valuable since it requires only one cluster to run
all the integration tests. This should still be possible, since it cannot be
assumed that additional cluster creation is always possible.

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

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- Should the default process for running integration tests assume isolation
  stronger than namespace is available?
  - For example, if KinD is always available through `bin/kind`, the default
    process can create a KinD cluster for each integration test

# Future possibilities

[future-possibilities]: #future-possibilities

### Parallel execution

With stronger integration test isolation, it could be possible to run
integration tests in parallel. This is currently not possible because only one
control plane can be installed on a cluster at a time, so namespace isolation
is not strong enough.

There are some unresolved questions about how results will be collected, how
many clusters would be run at once, and how a user switches between modes.