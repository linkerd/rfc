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

## Summary

Once implemented, Linkerd's integration test suite will:
- Have a more discoverable UX for running all or one test
- Less side effects when running the tests (and less footguns)
- Easier to extend for more complex integration tests

The two main components that the following subsections talk about are the code
component and the script component.

The code component is comprised of the Go files in the `test/` directory and
have direct interaction with `go test`. The Go test files located here are
responsible for testing specific features of Linkerd such as service profiles,
tap, edges, and more--they are not the actual integration tests. It is
important to note that these tests assume the cluster state is already set,
and it should stay that way. This allows developers to run `go test` and
specify the cluster context, namespace, and other knobs that may be important
for properly testing certain cases.

The script component is comprised of the `test-run` and `_test-run.sh`
scripts. These are the actual integration tests. Each integration test is
comprised of cluster setup and some combination of the test files from the
code component.

The separation here is important to note for this change: the integration
tests are contained within scripts and responsible for managing the cluster
state and running some combination of Go test files.

Therefore, these changes will be mostly script based. There is some possible
tooling that will allow managing cluster state through code (discussed in `#
Future  possibilities`) but the ecosystem is currently not in a state where we
can avoid breaking out to the shell at some point for configuring something
about the environment.

### UX

Running the integration tests will be done through new scripts that are
responsible for the setup and teardown of their testing environments.

This is different from the current situation in a few ways. First, there is
only one self-contained script that runs the integration test suit; it runs
all the tests and does not provide a way to select specific ones. If one does
wish to run a specific test, they must export the internal test script into
their shell environment.

Second, the environment setup check must be run before each integration test.
This is automatic in the self-contained script, but is a required manual step
for running a specific one.

It will be important to document how developers can run all or one of the
tests, so providing clear instructions in the Test markdown file, as well as a
`--help` flag for the scripts will be included.

Before: Running all tests:

```
$ bin/test-run $PWD/target/cli/darwin/linkerd
```

Before: Running specific test:

```
$ . bin/_test-run.sh
$ init_test_run $PWD/target/cli/darwin/linkerd
$ deep_integration_tests
```

After: Running all tests:

```
$ bin/tests $PWD/target/cli/darwin/linkerd
```

After: Running specific test:

```
$ bin/deep-tests $PWD/target/cli/darwin/linkerd
```

### Constraints

Tests will no longer be isolated by namespace. Each will require its own
cluster so that it can require specific initial state.

This will be the default behavior, but can be configured with a
`--skip-cluster` flag that changes the behavior back to creating namespaces
for each test.

Currently, the existing tests all assume the same initial state. With each
test running within its own cluster, deleting the cluster is all that is
required for the cleanup step. This guarantees no previous test leaves behind
resources for future tests.

If there are Linkerd resources in the namespace or cluster, the test will
abort and not attempt to delete those resources. This is different from the
current behavior where resources are immediately deleted; this can be a
surprising effect if ran on in a context where there is an existing install.

Only the scripting component of the integration test suite will need to change
to support this. The scripting component is located in the `_test-run.sh` file
and contains all the existing integration tests. Each test is a combination of
environment setup and composed of tests in the `test/` directory.

Each test will now have an initial step for creating the cluster that it
requires. For example, the current `deep_integration_tests` is below:

```
deep_integration_tests() {  
    run_test "$test_directory/install_test.go" --linkerd-namespace=$linkerd_namespace
    exit_on_err 'error during install'

    run_test "$(go list $test_directory/.../...)" --linkerd-namespace=$linkerd_namespace
    exit_on_err 'error during deep tests'
    cleanup
}
```

This works by running the `install_test.go` test which installs the given
Linkerd binary into the current cluster and specified namespace. It then runs
every test file in the `test/` directory. When complete, it cleans up the
namespace.

After this change, the deep tests will look like the below:

```
deep_integration_tests() {
    "$bindir"/kind create cluster --config $test_directory/configs/deep_integration_tests.yaml --name deep-integration-tests

    run_test "$test_directory/install_test.go" --context kind-deep-integration-tests
    exit_on_err 'error during install'

    run_test "$(go list $test_directory/.../...)" --context kind-deep-integration-tests
    exit_on_err 'error during deep tests'
    
    "$bindir/kind delete cluster --name deep-integration-tests
}
```

There is now a new beginning and ending step that handles cluster creation and
deletion. The cluster created is defined in a new `configs/` directory within
`test/`. Below is an example configuration:

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

This will create a 3 node cluster and run all the deep subtests within that
cluster environment.

### Extensibility

Extending the tests for more advanced cluster setups will be possible the
namespace constraint no longer present.

Using the supported configurations in [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#configuring-your-kind-cluster) or `k3d` (primarily the `--workers`
flag) will allow for our existing tests to specify a single node
configuration, but a new test to require a 3 node configuration.

Another way that tests will be extendable with isolation happening at the
cluster level is tests that require multiple clusters. Similar to the example
shown above, an integration test like this will just create two clusters at
the start:

```
multi_cluster_test() {
    "$bindir"/kind create cluster --config $test_directory/configs/multi-cluster-test.yaml --name cluster-1

    "$bindir"/kind create cluster --config $test_directory/configs/multi-cluster-test.yaml --name cluster-2

    run_test ..

    ..
}
```

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

### Cluster management from Go modules

The proposed design change is mainly about the scripting component of the
integration test suite.

It would be nice to be able to use cluster management commands from the test
code. The scripts and cluster setups will both grow in complexity as more
tests as added. It would be ideal if this was contained in the Go code.

Currently, k3d is working towards a `v3` where cluster management commands
will be offered as a Go module. This should allow the test script funcs to
move their `cluster create/delete` steps into setup and teardown steps of
their respective `TestMain`s.

### Parallel execution

With stronger integration test isolation, it could be possible to run
integration tests in parallel. This is currently not possible because only one
control plane can be installed on a cluster at a time, so namespace isolation
is not strong enough.

There are some unresolved questions about how results will be collected, how
many clusters would be run at once, and how a user switches between modes.
