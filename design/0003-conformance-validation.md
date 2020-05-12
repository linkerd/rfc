# Conformance Validation

- Contribution Name: Conformance validation
- Implementation Owner: [@mayankshah1607](https://github.com/mayankshah1607)
- Start Date: 2020-05-12
- Target Date: 2020-08-15
- RFC PR: [linkerd/rfc#0000](https://github.com/linkerd/rfc/pull/24)
- Linkerd Issue: https://github.com/linkerd/linkerd2/issues/1096
- Reviewers: [@Pothulapati](https://github.com/Pothulapati), [@ihcsim](https://github.com/ihcsim)

## Summary

This project proposes a new test suite that shall be used for conformance validation. The new test suite shall be used to validate non-trivial network communication (HTTP, gRPC, websocket) among stateless and stateful workloads in the Linkerd data plane. The correctness of the interaction between the Linkerd control plane and the Kubernetes API Server will also be tested. This shall be done by carrying out extensive e2e tests of Linkerd features using a sample distributed application (data plane).

## Problem Statement (Step 1)


Linkerd has an extensive check suite that validates a cluster is ready to install Linkerd and that the installation was successful. These checks are, unfortunately, static checks. Because of the wide number of ways a Kubernetes cluster can be configured, users want a way to validate their specific install for conformance over time. The proposed project tackles this problem by allowing users to deploy sample workloads to their cluster and carry out extensive E2E tests for conformance.

## Goals and Deliverables

### Goals
The goal of this project is to :

1. develop a sample application that can exercise various features of Linkerd, mainly that involve the data plane.
   
2. provide an extensive e2e test suite that can make use of this application and validate for conformance.

### Deliverables

#### Must have (shall be completed by end of GSoC):
- An e2e test suite that can perform conformance validation for the following features:
    1. Automatic proxy injection
    2. `linkerd tap`, `stat`, `routes`, `edges` cmd
    3. Verifying if `tap` extenstion API server is functional
    4. Retries and Timeouts
    5. Verifying if Data Plane proxies are healthy
    6. Ingress
- A Sonobuoy plugin for our test suite

- A sample emojivoto app with feature flags that can enable/disable features required for conformance validation.

#### Good-to-have (if extra time remains, these shall be worked on):
- The conformance validation testing suite shall also cover the following features :
    1. Telemetry and metrics
    2. Cluster security and introspection
    3. Data plane PSP
    4. Distributed tracing
    5. MySQL, Redis
   
- A new sample application - MovieChat - that has all the features required from conformance validation perspective

### Non-goals
It is a non-goal for this project to provide an application that is expected to be a part of Linkerd application architecture (control / data plane). This project in no way shall directly affect the way Linkerd functions. Unlike the existing integration tests, the e2e tests for conformance in this project are not expected to be a part of the Linkerd CI workflow either. These tests shall run as a stand-alone component such that it can interact with a k8s cluster.

## Design proposal (Step 2)

[design-proposal]: #design-proposal

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear
- It is reasonably clear how the contribution would be implemented
- Corner cases are dissected by example
- Dependencies on libraries, tools, projects or work that isn't yet complete
- Use Cases
- Goals
- Non-Goals
- Deliverables

## Interaction with other features
The e2e conformance tests proposed would be carried out outside the Linkerd CI. These tests would make use of the `kubectl` and `linkerd` binaries to interact with the cluster and with various features Linkerd can perform.

We initially start off by modifying the existing emojivoto application to have feature flags to enable/disable features such as MySQL, Redis, gRPC, websockets, etc. This is important because emojivoto is heavily used in the getting started process.

### 1. Running the tests vanilla style (default mode)
First let us look at how the test framework can run without having to depend on Sonobuoy. To do so, we provide 2 essential items as a part of the project :

1. A `run.sh` file that accepts a list of flags (see use case and implementation details below) and calls `go test` accordingly.

2. The test suite (the `_test.go` files)

To run the test suite, users will be required to execute `./run.sh` with a group of flags (more details below). Further, the output of the tests shall be written to a file path passed as a flag to `./run.sh`. By default, `stdout` shall be used to show output of the tests.

> As a part of this default mode, users shall be allowed to run these tests in a container as well. We make use of the Dockerfile being introduced in the next section (Read below).

### 2. Using Sonobuoy

Sonobuoy is a great tool when it comes to conformance validation. Its [plugin model](https://sonobuoy.io/docs/master/plugins/) can allow us to easily extend functionality beyond K8s conformance validation. Essentially, Sonobuoy runs your test suite in a pod in the cluster, hence, also saves developers the overhead of configuring the desired objects such as Namespaces, ServiceAccounts, ClusterRoleBindings, Jobs, RBAC, etc.

Now, for the ease of management of the project, we simply extend our default approach as described above, to provide a [sonobuoy plugin](https://sonobuoy.io/docs/master/plugins/). Hence, we require 2 new items :

**1. A Docker image to house our test suite**

As per Sonobuoy's requirements, the test suite and its dependencies shall be packaged as a Docker container. This container shall run as a job in the cluster. In the Dockerfile, we define that the following must happen

- Setup and install all the binaries/tools required by the testing environment - `linkerd`, `kubectl`, `mysql`, etc.
- Copy _run.sh_ file that initiates the tests. (Note that we are reusing the same `run.sh` file from the default approach).
- Copy the test suite
- ENTRYPOINT `./run.sh`

**Good-to-have:**

Furthermore, having a Dockerfile would not only give us the benefit of being able to use Sonobuoy, but also that users may be able to run
the tests in a Docker container as a part of the default mode (See previous section), instead of running it on the local environment.

To do so, users have to execute the following commands:
```bash
# build the container
$ docker build <user>/l5d-conformance:latest

# run the container
$ docker run -it -v ~/.kube/config:~/.kube/config <user>/l5d-conformance [run.sh args]

# Optional
$ docker push <user>/l5d-conformance
```
Additionally, we may provide a Makefile that runs these commands for the user. However, we may have to rethink the way they can pass `./run.sh` flags (One option could be to have them edit the Makefile according to their needs).

We may also host this Docker image on a public repository, for e.g - _gcr.io/linkerd.io/conformance:v1.0.0_. The Sonobuoy plugin file (see next step) may benefit from this.

**2. A sonobuoy plugin file (YAML)**

The main function of Sonobuoy is running plugins; each plugin may run tests or gather data in the cluster. A plugin is defined using a YAML file that holds information such as 

- Name of plugin
- Format of result
- Type of a plugin - Job or DaemonSet
- Command to execute
- args/flags for the the command
- The Docker image to use
- volumeMounts - for storing results

We shall provide a plugin file, _l5d_conformance.yaml_ for running our test framework as a Sonobuoy plugin. It may look something like :
```yaml
sonobuoy-config:
  driver: Job
  plugin-name: l5d-conformance
  result-format: raw
spec:
  args:
  - --enableTest="TestAutoProxyInjection/*" \
  - --failFast \
  - --ha \
  - --addon-config=addon-config.yaml
  # flags should be added here. More information regarding these is described below
  command:
  - ./run.sh
  image: gcr.io/linkerd.io/conformance:v1.0.0
  name: plugin
  resources: {}
  volumeMounts:
  - mountPath: /tmp/results
    name: results
``` 

In order to run the Sonobuoy plugin, users/testers shall be required to carry out the following steps:

1. Edit the above mentioned plugin file as required. This involves modifying the list of args.

2. Run the plugin
```bash
$ sonobuoy run --plugin l5d-conformance.yaml
```
3. Collect output
```
$ outfile=$(sonobuoy retrieve) && \
mkdir results && tar -xf $outfile -C results
```
---

### The role of `run.sh`
The `run.sh` file performs the following functionalities :

#### 1. Execute `go test` command
The executable `run.sh` file shall accept flags and build a `go test` cmd to make the tests configurable as well as set `linkerd install` options.

Usage : 
```bash
./run.sh [flags]
```
Flags :

**(Specific to test configuration)**

``` 
--wait String
    duration for which the tests must wait for pods to come to a running state

--enableTests String
    Comma separated list of test names that must be executed. This is later converted into regexp and passed as a flag value for `-run` for `go test` command.

--skipTests String
    Comma separated list of test names that must be skipped. The value of this is later used in the go code with the t.Skip() method.

--failFast
    Do not start new tests after the first test failure.

--parallel int
    Number of tests to run in parallel

--buildTags String
    Comma separated list of tags that must be included in the test

--o String
    Filepath to store the result. Ignored in case of Sonobuoy 

--skipInstall
    Skips the `linkerd install` process. This is useful for testers who already have linkerd installed prior to running the tests

--addon-config String
    Filepath to the addon config file. This flag must be used only in "default mode"

--clean
    Always destroy resources created for testing, regardless of failure or success. If not set, resources are destroyed only if all the tests pass.

--workloadNamespace String
    This could be a future possibility. The value of this flag identifies the namespace of the workloads against which the conformance tests must run.
```
Internally, the script executes a `go test` command by simply appending these values to it. The test configuration specific flags (such as `--wait`,`--enableTests`,`--workloadNamespace`, etc) are passed down to the test suite using the _flags_ go module. These values may be stored in variables (instance of objects, for e.g `TestOptions`) so that tests may run accordingly. A universal `TestHelper` object may also be used to make the flag management process easier.

**(Essential `linkerd install` flags to accept)**

- `--identity-trust-anchors-file` 
-   `--identity-issuer-certificate-file`
-  `--identity-issuer-key-file`
-  `--identity-issuance-lifetime`
-  `--proxy-cpu-limit`
-  `--proxy-cpu-request`
-  `--proxy-memory-limit`
-  `--proxy-memory-request`
-  `--ha`
-  `--addon-config`

The script also executes a `linkerd install` command by simply appending these flags to it. The built command is then executed to install Linkerd on the cluster.

Sample Usage :

```bash
./run.sh --wait=30s \
    --enableTest="TestAutoProxyInjection/*" \
    --failFast \
    --identity-trust-anchors-file=ca.crt \
    --identity-issuer-certificate-file=issuer.crt \
    --identity-issuer-key-file=issuer.key \
    --identity-issuance-lifetime=1m \
    --proxy-cpu-limit=1 \
    --proxy-cpu-request=100m \
    --proxy-memory-limit=250Mi \
    --proxy-memory-request=20Mi \
    --ha \
    --addon-config=addon-config.yaml
```

#### 2. Pre-flight checks

Before the test suite is run, the following actions shall be carried out in sequence:

1. Check if `kubectl` binary is available

2. Check if `linkerd` binary is available

3. Check if connection to cluster can be established via `kubectl`

4. Check if Linkerd control plane is installed

5. Install the sample application

---

### Testing Methodologies for various features

The conformance test suite is intended to be run against an installation of Linkerd. Passing this test suite would give the user the assurance that a given configuration of Linkerd works (as expected) with a given version of Kubernetes and its cluster configuration.

This section includes write ups regarding testing methodologies for some primary features:

**1. Automatic Proxy Injection**

Proxy injection process works by adding a `linkerd.io/inject: enabled` annotation to pod template / namespace. This triggers an admission webhook that injects the `linkerd-proxy` container to the targeted deployments. This process shall be tested in 7 steps:

- Check if `inject` command (with and without `--manual` flag) outputs the desired configuration with the appended annotation. We shall provide an injected YAML file of the deployments which can be compared against the YAML blob output of inject.

Alternatively, as users may want to test for conformance on their own sample applications (check "Future possibilities" section), we may have to make use of [kubernetes/client-go](https://github.com/kubernetes/client-go) to only check for the existence of the `linkerd.io/inject: enabled` annotation under the pod metadata.     

- Wait for the pods to come to a _Running_ state for a certain duration, after which the test shall fail.

- Check if the pods actually have the `linkerd-proxy` and the `linkerd-init` container

- Check if the `linkerd-proxy` container is reachable by sending a `GET` request to liveness/readiness probe - to ensure service discovery is functional.

- Check for any `SkipInject` errors in events

- Check if uninject command updates the annotation to `linkerd.io/inject: disabled`

- Wait for the pods to come to a _Running_ state for a certain duration, after which the test shall fail.

- Ensure that the pods no longer have the `linkerd-init` and `linkerd-proxy` containers.
  
**2. `tap` extension API server**

Some custom Kubernetes clusters don't always have the aggregation layer configured, causing the tap service which is an extension API server to fail.
- Issue a `check` command - `linkerd -n <ns> check --pre -o json`
- Validate the returned JSON by ensuring that the array the check with description `"can read extension-apiserver-authentication configmap"` under category `"pre-kubernetes-setup"` says `"success"`

**3. Essential linkerd commands**

**linkerd tap cmd**

This test will work by issuing the `linkerd tap` cmd on various resources of the sample application. The output returned shall be checked for any errors. For e.g - GKE may require extra RBAC configurations and permissions. Tap shall be tested by issuing a tap command using the `-o json` flag and performing checks on the obtained JSON :
- Check for gRPC status codes
- Check authority
- Check for tls settings
- Check httpStatus

**linkerd stat**

- Check if Prometheus pod(s) is/are available
- Check if controller pods are up and running
- Check if the application pods are up and running
- Issue stat cmd and validate output by checking for success rate and various latencies 
> Note: This test may have to be put on hold due to https://github.com/linkerd/linkerd2/pull/3693

**linkerd edges**

- Check if application pods are up and running
- Check if application deployment has desired replicas
- Issue `linkerd edge` cmd using `-o json` flag, on various resources
- Validate the `no_tls_reason` field of JSON output to ensure that the edges are mTLS'ed.

**linkerd routes**
- Check if application pods are up and running
- Issue `linkerd routes` cmd
- Validate the returned output by counting instances of desired route substrings using `strings.Count`
  
**4. Ingress**

It is essential to test ingress to ensure that the ingress controller re-writes incoming headers to the internal service name. This process ensures that service discovery is working. This test shall include deploying various types of ingress controllers (Nginx, Traefik and Ambassador) :

- Deploy and verify if the ingress controller is up and running. The YAML for various ingress controllers shall be provided along with the test tool. 
- Deploy the ingress resource. An example from the docs:
  
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  namespace: emojivoto
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;
      grpc_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;

spec:
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: web-svc
          servicePort: 8080
```

- Obtain external IP of the controller. For e.g :

```bash
$ kubectl get svc --all-namespaces \
  -l app=nginx-ingress,component=controller \
  -o=custom-columns=EXTERNAL-IP:.status.loadBalancer.ingress[0].ip
```

> Note: The ingress resource file and the command to obtain the external IP may change according to the ingress controller being tested.

- Issue a curl cmd on the external IP to check if desired response is returned. This shall be done by wrapping the curl bash cmd as a method in Golang using `command.exec()`.
- On failure of this test suite, useful debugging information shall be displayed - such as the output of attempting to fetch an external IP, or the output of `cURL`. Further, users shall be allowed to not have these resources destroyed on test failure.

**5. Data plane proxy health checks**

- Issue a `check` command - `linkerd -n <ns> check --proxy -o json`
- From the output JSON, under `"categoryName" : "linkerd-data-plane"`, verify if the `result` of each check under the `checks` array shows `success`.
- Make a `GET` request to the `linkerd-proxy` containers (of each of the pods) at the `/metrics` (Liveness probe) and `/ready` (Readiness probes) to ensure that they are reachable.
- From the `linkerd-proxy` container of each of the pods, check for 503 errors.

**6. Retries and Timeouts**

- Some of the code from the existing integration tests may be reused for this section. In particular, we're looking for `test/serviceprofiles/serviceprofiles_test.go`
- After verifying if our _emojivoto_ deployments and services are up and running, a `linkerd routes` cmd shall be issued, and the returned routes are compared to the expected routes. We shall pick deployments that have an edge, For e.g - `web` and `voting`
- Once the expected routes match the returned routes, a `linkerd profile` cmd must be issued to ensure that a `ServiceProfile` is generated for `web` and `voting`. To do this, we may use the `--tap` flag to generate a `ServiceProfile` based off live traffic data using `tap`. The output of this command shall be piped to `kubectl apply`, much like the `TestHelper.KubectlApply()` method in the integration tests.

**Retries:**

- Now, the tests execute `linkerd routes -n emojivoto deploy/web --to svc/voting-svc -o json` and note the value of `"effective_success"` for any of the routes. We may choose any route (or all routes) for this depending on the edge selected.
- The code then proceeds to add an `isRetryable: true` field under the selected route(s) for `deploy/voting`. To do this, the `ServiceProfile` YAML is unmarshalled into a `ServiceProfile` object (from `controller/gen/apis/serviceprofile/v1alpha2/types.go`), and an `IsRetryable: true` field is appended to the `RouteSpec`. The object is then marshalled into a YAML and piped to `kubectl apply`.
- Finally, from the previous `routes` cmd, the value for `"effective_success"` is obtained and verified if the present value is greater than before.

**Timeouts:**

- Testing for Timeouts shall work similar to Retries. The tests execute `linkerd routes -n emojivoto deploy/web --to svc/voting-svc -o json` and note the value of `"effective_success"` for any of the routes depending on the edge selected.
- The `ServiceProfile` YAML for `deploy/voting` is unmarshalled into a `ServiceProfile` object and a `Timeout` value is set under `RouteSpec` (say, "25s"). The object is then marshalled and piped to `kubectl apply`
- Finally, from the previous `routes` cmd, the value for `"effective_success"` is verified , i.e, if the present value is lesser than before.

Additionally, we could show the `EFFECTIVE_RPS`  and `ACTUAL_RPS` metrics from the output of `tap` as a way to observe the difference in the requests being sent by the client and the requests being received by the Linkerd proxy.

Further, following up with [this comment](https://github.com/linkerd/linkerd2/issues/2179#issuecomment-471643537) it would be nice to have `emojivoto` configured to monitor the occurenced of retries and timeouts. For example, a service or a set of services may be configured to have routes dedicated for testing retries and timeouts; a service that accepts a request with 3 parameters - `succeed-after-retries`, `id`, and `delay`. The service returns `200 OK` only after `succeed-after-retries`, and also keeps track of how many times the service was called before that. The service may also sleep for `delay` before servicing a request, as a way of validating timeouts.

This way, users shall be allowed to monitor the occurences of retries and timeouts.  


**7. Distributed Tracing (optional)**

Currently there exist some integration tests for _Distributed tracing_. Applications meshed with Linkerd can be easily tested for this by making a `GET` request on the Jaeger backend at `/api/traces` endpoint on port 16686. (lookback and service parameters shall be made configurable via CLI flags)

**8. Canary Release (optional)**

Once a Canary Release is configured, an updated may be triggered. The metrics from the `stat` cmd may be verified to ensure that this feature is configured correctly.


### Protocol related tests
It is important to note that these tests could fail if the `workloadNamespace` property of the test config is set to anything other than the default (moviechat/emojivoto) application.

**1. gRPC Streaming**

The movie chat application / emojivoto shall leverage gRPC streaming. It is essential to ensure that streaming services are not affected by Linkerd - under normal conditions as well as during stress tests (if any).

- To ensure that there is constant traffic between services that use gRPC, the tests shall issue a `linkerd stat -n <ns> <resource> -o json` cmd. 
- To test this gRPC traffic, the tests shall validate the `success` field of the JSON output, which ideally should be 1.00 (or 100%).

**Good to have:**  
The logs gathered by Sonobuoy may also show metrics such as `rps` and the different latencies.

**2. Websockets**

Similar to gRPC streaming, it is essential to check if Linkerd does not break services that utilise websocket connections. In our sample application, we create a chat server that handles socket connections to and fro from the web application. To test this, we can ping the websocket endpoints using curl. For example:

```bash
$ curl --include --no-buffer \
    --header "Connection: close" \
    --header "Upgrade: websocket" \
    http://localhost:3000/socket
```

This command would return a response from the websocket server from the /socket endpoint where websockets shall be served.

Further, similar to gRPC testing, `linkerd stat` shall be used to check if there is constant traffic to and from the websocket endpoints.

**3. MySQL and Redis**

- The emojivoto application shall be modified to have persistent storage. The `voting` deployment may be configured to work with MySQL and Redis (for cache). 
- The `vote-bot` deployment may send constant traffic to `voting` which can be setup to simulate :
  - **cache hit** : If `voting` finds an emoji in Redis which has already been voted for earlier
  - **cache miss** : If `voting` cannot find the emoji in Redis, it fetches the record from MySQL and performs necessary updates.

## Corner Cases

It is possible to have prior knowledge regarding which use cases may need to be configured differently depending on the environment. For E.g, tap requires extra RBAC configurations on GKE. Instead of having the corresponding tests for tap failing, users may be allowed to explicitly mention the environment these tests shall run on. The testing tool may show a warning, providing a link to the docs which provide instructions on how to setup GKE for Linkerd. This would have a positive impact on UX.

Similarly, any known corner cases related to the infrastructure of the cluster can be documented.

## Dependencies on tools / libraries

- Go
- shell scripting
- Docker
- Sonobuoy
- Kubectl
- Linkerd
- MySQL
- Redis
- curl
- [Ginkgo](https://github.com/onsi/ginkgo) (good-to-have)


## Use Cases

The primary use cases of this project include the tests listed under the "Testing methodologies for various features" section.

## Prior art

- Sample Sonobuoy plugins - https://github.com/vmware-tanzu/sonobuoy/tree/master/examples/plugins
- K8s e2e tests - https://github.com/kubernetes/kubernetes/tree/master/test/e2e
- K8s Sonobuoy image - https://github.com/kubernetes/kubernetes/tree/master/cluster/images/conformance
- Some sample applications developed by Linkerd for testing:
    - https://github.com/BuoyantIO/emojivoto
    - https://github.com/BuoyantIO/booksapp
    - https://github.com/linkerd/linkerd-examples
    - https://github.com/BuoyantIO/slow_cooker/

## Unresolved questions

[unresolved-questions]: #unresolved-questions

-

## Future possibilities

[future-possibilities]: #future-possibilities

As a Linkerd user, it is important to know that Linkerd works with my services, not just emojivoto. Setting the `workloadNamespace` flag while executing `./run.sh` gives the users this flexibility.
- `config.linkerd.io/e2e: conformance` label is appended to the namespace config
- The tests then hit the k8s API with a label selector to annotate all objects (deployments) under this namespace with the `linkerd.io/inject: enabled` annotation to trigger auto-injection.
- It is crucial for Linkerd commands like `stat`, `route` and `edge` to support label selectors, hence, a patch for this must be made.

Related - https://github.com/linkerd/linkerd2/pull/4120 , https://github.com/linkerd/linkerd2/pull/4040
