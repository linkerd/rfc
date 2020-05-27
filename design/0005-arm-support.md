# Title

- Contribution Name: ARM Support
- Implementation Owner: [@aliariff](https://github.com/aliariff)
- Start Date: 2020-02-23
- Target Date:
- RFC PR: [linkerd/rfc#29](https://github.com/linkerd/rfc/pull/29)
- Linkerd Issue:
  [linkerd/linkerd2#1165](https://github.com/linkerd/linkerd2/issues/1165),
  [linkerd/linkerd2#3114](https://github.com/linkerd/linkerd2/issues/3114)
- Reviewers: [@cpretzer](https://github.com/cpretzer)

## Summary

[summary]: #summary

This project goal is to make Linkerd officially
support ARM architecture (`arm` & `arm64`).

## Problem Statement

[problem-statement]: #problem-statement

Currently, Linkerd is only supporting `amd64` architecture officially.
Supporting other architectures will make Linkerd more powerful and
attractive for the users.

After the project implemented, releasing process will have more steps:

- Build docker images in ARM architecture
- Integration test for the resulting images
- Publish the images

## Design proposal

[design-proposal]: #design-proposal

There are 3 repositories need to change to support `arm`:

1. [linkerd2-proxy-init](https://github.com/linkerd/linkerd2-proxy-init)
2. [linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy)
3. [linkerd2](https://github.com/linkerd/linkerd2)

### Build & Publish Strategy

#### Strategy 1: Introduce ENV key

Adding a new key to define what architecture the image will be build,
and use that key for indicator when performing cross compilation.

- `linkerd2-proxy-init`
  - Add Makefile
  - Define `build` instruction in Makefile to build & apply appropriate tag

    ```makefile
    docker build -t gcr.io/linkerd-io/proxy-init-${DOCKER_BUILD_ARCH:-amd64}:... \
      --build-arg "GOARCH=${DOCKER_BUILD_ARCH:-amd64} .
    ```

  - Define `push-all` instruction in Makefile to push the image & multi-arch manifest

    ```makefile
    docker push gcr.io/linkerd-io/proxy-init:...-amd64
    docker push gcr.io/linkerd-io/proxy-init:...-arm64
    docker push gcr.io/linkerd-io/proxy-init:...-arm
    docker manifest create gcr.io/linkerd-io/proxy-init:... \
      gcr.io/linkerd-io/proxy-init:...-amd64 \
      gcr.io/linkerd-io/proxy-init:...-arm64 \
      gcr.io/linkerd-io/proxy-init:...-arm
    docker manifest push gcr.io/linkerd-io/proxy-init:...
    ```

  - Use the `GOARCH` argument to perform cross compile in `Dockerfile`

    ```Dockerfile
    ARG GOARCH
    RUN CGO_ENABLED=0 GOOS=linux GOARCH=$GOARCH go build ...
    ```

  - Change the runtime image in `Dockerfile`

    ```Dockerfile
    FROM --platform=$GOARCH debian:stretch-20190812-slim
    ```

  - Add Github actions
  - Call the build inside github workflow

     ```yml
     make build # this will build the amd64 image
     DOCKER_BUILD_ARCH=arm64 make build # building arm64 image
     DOCKER_BUILD_ARCH=arm make build # building arm image
     ```

  - Push image & push multi arch manifest

    ```yml
    make push-all
    ```

- `linkerd2-proxy`
  - Add new step in `.github/workflows/release.yml`
  - Install requirement to perform cross compilation in Rust

    ```yml
    sudo apt-get install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
    ```

  - Set the appropriate environment

    ```yml
    env:
      CARGO_TARGET_ARCH: armv7-unknown-linux-gnueabihf # for arm
      CARGO_RELEASE: "1"
    run: make package
    or
    env:
      CARGO_TARGET_ARCH: aarch64-unknown-linux-gnu # for arm64
      CARGO_RELEASE: "1"
    run: make package
    ```

  - Change the Makefile to check for `CARGO_TARGET_ARCH` to perform cross compile

    ```makefile
    .
    .
    CARGO_BUILD = $(CARGO) build --frozen $(RELEASE) --target=$(CARGO_TARGET_ARCH)
    .
    .
    ```

  - Publish the artifact together with other architecture

- `linkerd2`

  There are 7 images to build: `controller`, `cli`, `cni-plugin`,
  `web`, `proxy`, `debug`, `grafana`

  - Export `DOCKER_BUILD_ARCH` in `bin/_docker.sh` &
  append the architecture in the repo name

    ```sh
    export DOCKER_BUILD_ARCH="${DOCKER_BUILD_ARCH:-}"
    ```

  - Update `bin/_tag.sh` to append the architecture

    ```sh
    .
    .
    .
    head_root_tag() {
      if clean_head ; then
        tag=clean_head_root_tag
      else
        name=$(echo $USER | sed 's/[^[:alnum:].-]//g')
        tag="dev-$(git_sha_head)-$name"
      fi
      echo "$tag-$DOCKER_BUILD_ARCH"
    }
    .
    .
    .
    ```

  - Pass the architecture argument to docker build in every `docker-build-*` file

    ```sh
    docker_build .... --build-arg "GOARCH=${DOCKER_BUILD_ARCH:-amd64}"
    ```

  - Use the `GOARCH` argument to perform cross compile in all `Dockerfile`

    ```Dockerfile
    ARG GOARCH
    RUN CGO_ENABLED=0 GOOS=linux GOARCH=$GOARCH go build ...
    ```

  - Change the runtime image in `Dockerfile`

    ```Dockerfile
    FROM --platform=$GOARCH ...
    ```

  - Adjust `bin/fetch-proxy` file to retrieve the correct architecture version
  - Add `bin/docker-manifest` file to create and push multi-arch manifest

    ```sh
    for img in cli-bin cni-plugin controller debug grafana proxy web ; do
      docker manifest create gcr.io/linkerd-io/$img:$tag \
        gcr.io/linkerd-io/$img:$tag-amd64 \
        gcr.io/linkerd-io/$img:$tag-arm64 \
        gcr.io/linkerd-io/$img:$tag-arm
      docker manifest push gcr.io/linkerd-io/$img:$tag
    done
    ```

  - Update `.github/workflows/release.yml`

    ```yml
    bin/docker-build
    DOCKER_BUILD_ARCH=arm64 bin/docker-build
    DOCKER_BUILD_ARCH=arm bin/docker-build
    .
    .
    .
    bin/docker-push $TAG
    DOCKER_BUILD_ARCH=arm64 bin/docker-push $TAG
    DOCKER_BUILD_ARCH=arm bin/docker-push $TAG
    bin/docker-manifest $TAG
    ```

With all of the steps are explained, there is some drawback
that we can see here as we need to create manifest for multi-arch our self.

#### Strategy 2: Docker Buildx

Using the latest experimental feature of docker buildx will make things easier
because it will build multi architecture, create manifest and also annotate
automatically, but as the nature of experimental features it may have problem
in the future, and the DSL might also change.
Another disadvantage is when performing multiple architecture build the
resulting image is push automatically to registry, so it is not loaded
on the local registry (`docker images`), it is not supported currently
([ref](https://github.com/docker/buildx/blob/master/README.md#docker)).

There are 3 strategies that we can do to build the image using Buildx:

1. QEMU

    - Advantage: There is no need to configure or provision machine, it is just works.
    - Disadvantage: It may take long time to build.

1. Remote Docker Machine with `arm` architecture

    - Advantage: Quick to build because it is native
    - Disadvantage: Provision & configure an ARM machine and
      setup as docker remote machine and maintain 2 docker machines
      as we already have 1 remote docker machine.

1. Cross-compile

    - Advantage: We can use what we have now, no need setup a machine.
    - Disadvantage: none

From this we can take a step to choose `cross-compile` options.

As most of the step will be the same as in the previous strategy using ENV key,
the part that different is how to run `docker build`.

```sh
# no need to use ENV key, list of build arguments will automatically available inside Dockerfile
docker buildx build --platform linux/amd64,linux/arm64,linux/arm .
```

Inside Dockerfile

```Dockerfile
# builder image
ARG TARGETARCH # automatically available
ARG TARGETOS # automatically available
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build ...
# runtime image
ARG TARGETPLATFORM # automatically available
FROM --platform=$TARGETPLATFORM ...
```

### Test Strategy

For integration testing using `kind`, unfortunately still not supporting `arm`
architecture ([ref](https://github.com/kubernetes-sigs/kind/issues/166)).
So the only way to test the integration of `arm` build is on the cloud.

- `linkerd2-proxy-init`
  - Setup `kind` in the Github Actions Workflow
  - Run test `integration_test/run_tests.sh` (only test `amd64`)

- `linkerd2-proxy`

  The only way to test `arm` build is to setup a remote docker machine
  with `arm` architecture.
  - Setup & Connect to an `arm` architecture remote docker machine in `.github/workflows/rust.yml`
  - Run build & test

  ```yml
  run: docker build -qf .github/actions/remote/Dockerfile . >image.ref
  .
  .
  .
  run: docker run --rm $(cat image.ref) test-integration
  ```

- `linkerd2`

  Cloud provider that supporting `arm` architecture:
  - [AWS](https://aws.amazon.com/ec2/instance-types/a1/)
  - [Packet](https://www.packet.com/cloud/servers/c2-large-arm/)

  That mentioned providers supporting `arm64`, no `arm` cloud provider found so far.

  Steps:
  - Setup the cluster in `.github/workflows/cloud_integration.yml`

    AWS had EKS that we can setup more straight forward,
    but with Packet we need to setup the cluster ourself.

  - Run the integration tests

### Prior art

[prior-art]: #prior-art

This project is already done in many CNCF projects like:

- [kubernetes/kubernetes#17981](https://github.com/kubernetes/kubernetes/issues/17981)
- [grafana/grafana#13186](https://github.com/grafana/grafana/issues/13186)
- [containous/maesh#206](https://github.com/containous/maesh/issues/206)

And also proof that Linkerd can be run on ARM [linkerd/linkerd2#1165](https://github.com/linkerd/linkerd2/issues/1165#issuecomment-515470739)

### Unresolved questions

[unresolved-questions]: #unresolved-questions

1. Should we consider to test `arm` on merge request (`fork`)?

### Future possibilities

[future-possibilities]: #future-possibilities

- Add more architectures like `ppc64le`, `s390x`
  to match with what Kubernetes already supported.
