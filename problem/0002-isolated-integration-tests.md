- Contribution Name: Isolated Integration Tests
- Implementation Owner: @kleimkuhler, @alpeb
- Start Date: 2020-03-18
- Target Date: 2020-04-03
- RFC PR: TODO: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/0000)
- Linkerd Issue: [linkerd/linkerd2#4190](https://github.com/linkerd/linkerd2/issues/0000)
- Reviewers: @grampelberg

# Summary

[summary]: #summary

It should be easier to run integration tests in a more isolated way. Instead
of the default testflow which isolates tests by namespace, tests will be
isolated by cluster. With each test creating it's own cluster, it will ensure
there is no existing Linkerd resources when starting the test. It will also
allow tests to be run in parallel because each test can run within its own
cluster.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

The default testflow for running integration tests isolates tests by namespace
on a single cluster. This is problematic for the following reasons:
- The resources from any previous or existing Linkerd installation must be
  deleted in order to not conflict with the new installation.
- Tests cannot be run in parallel because only a single control plane can be
  installed on a cluster

Once the change is implemented, it will be easier to isolate integration tests
by more than just namespace. This will allow for each integration test to run
independently of any previous run or existing installation.

Additionally, with finer grained integration test isolation, it will be
possible for the default testflow to run integration test in parallel. This
will allow for a faster test runs.

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

The existing process for running Linkerd integration tests provides both good
and bad properties for isolated integration tests.

It should still be possible to run integration tests on a single cluster. In
this scenario, the current process is acceptable. Running tests serially and
in their own namespaces ensures that previous runs are completely removed,
only one control plane is installed on the cluster at a time, and errors are
properly communicated to the user.

There are several bad properties that this current process has. If an existing
installation exists on the cluster, it is deleted. This can be a surprising
effect especially if the user did not intend to delete their current
installation.

Also, if a previous run is not cleaned up properly then the currently running
test has no way to recover. This can be a symptom of tests not being isolated
enough.

Finally, the only way to run tests in parallel is manually create all the
clusters, export the `_test-run.sh` functions into the current environment,
and manually start each integration test. This requires deeper knowledge of
the script and is not clearly documented.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

Should the default testflow for running integration tests assume cluster
creation is available? Running each integration tests on a fresh cluster would
be ideal, but that that may not be feasible.

Without detailing the proposed solution, I imagine KinD will be a valuable
tool and we already have tooling built up around it being available
(`bin/kind`), but what cases would we need to continue isolating only by
namespace?

# Future possibilities

[future-possibilities]: #future-possibilities

- If cluster creation is available, running integration tests in parallel
  would be valuable.
