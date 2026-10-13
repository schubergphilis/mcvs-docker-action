# Usage Examples

## Basic Usage

For a simple single Dockerfile project:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```

## Custom Build Arguments

Pass build arguments to your Docker build:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    build-args: my-application
    token: ${{ secrets.GITHUB_TOKEN }}
```

For multiple build arguments, use multiline format:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    build-args: |
      APPLICATION=my-app
      VERSION=${{ github.ref_name }}
      BUILD_DATE=${{ github.event.head_commit.timestamp }}
    token: ${{ secrets.GITHUB_TOKEN }}
```

## Matrix Builds

Build multiple images from the same repository:

```yaml
jobs:
  build:
    strategy:
      matrix:
        config:
          - name: main-app
            build-args: main-application
            context: ./app
            suffix: ""
          - name: cli-tool
            build-args: cli-application
            context: ./cli
            suffix: /cli
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: schubergphilis/mcvs-docker-action@v0.1.0
        with:
          build-args: ${{ matrix.config.build-args }}
          context: ${{ matrix.config.context }}
          images: ghcr.io/${{ github.repository }}${{ matrix.config.suffix }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

## Multi-Architecture Builds

By default the action builds and pushes both `linux/amd64` and `linux/arm64`.
No configuration is required:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```

Every platform is built on every run, so that an architecture specific build
failure surfaces on a pull request rather than on a tag. Only the push is
limited to tags. The `linux/arm64` image is built through QEMU emulation, which
makes every run slower. To build a single architecture instead:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    platforms: linux/amd64
    token: ${{ secrets.GITHUB_TOKEN }}
```

That also skips the QEMU setup, as no emulation is needed.

Dockle, Dive and Grype scan a single platform per job: the platform that the
runner provides natively when it is in the list, otherwise the first entry. On
an `ubuntu-24.04-arm` runner that is `linux/arm64`. To scan every architecture,
give each one its own job as described below.

Additional platforms, e.g. `linux/arm/v7`, can be added to the list and are
built and pushed like any other.

Do not hardcode a platform in the Dockerfile, e.g. `FROM --platform=linux/amd64
alpine`, as hadolint rejects that with
[DL3029](https://github.com/hadolint/hadolint/wiki/DL3029). The action passes
`--platform` to Buildx, which provides `BUILDPLATFORM`, `TARGETPLATFORM`,
`TARGETOS` and `TARGETARCH` to the build.

### Scanning Every Architecture

Only one image can be loaded into the local Docker image store, so a job scans a
single platform. To scan every architecture, run the action in a matrix with one
platform per job, which GitHub executes in parallel:

```yaml
jobs:
  scan:
    strategy:
      matrix:
        platform:
          - linux/amd64
          - linux/arm64
    runs-on: ${{ matrix.platform == 'linux/arm64' && 'ubuntu-24.04-arm' || 'ubuntu-24.04' }}
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: schubergphilis/mcvs-docker-action@v0.1.0
        with:
          platforms: ${{ matrix.platform }}
          push-to-container-registry: ""
```

Each job runs the complete pipeline for its own platform, i.e. hadolint, the
build, Dockle, Dive and Grype. Picking a runner that matches the architecture
also avoids QEMU emulation entirely.

Note that:

- This matrix does not push. `push-to-container-registry: ""` disables it,
  because jobs that each push the same tag would overwrite one another instead of
  producing a manifest list. Keep the ordinary single job shown above for the
  release, which builds every platform and pushes one manifest list.
- This is optional. The single job example above keeps working unchanged and
  needs no matrix.

## Custom Dockerfile Location

If your Dockerfile is not in the repository root:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    context: ./docker/production
    token: ${{ secrets.GITHUB_TOKEN }}
```

## Handling Dockle False Positives

When Dockle incorrectly flags specific package versions as secrets:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    dockle-accept-key: libcrypto3,libssl3
    token: ${{ secrets.GITHUB_TOKEN }}
```

When a third-party package ships a file whose name Dockle always treats as a
credential, e.g. `azure-core`'s `azure/core/settings.py`:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    dockle-accept-file: settings.py
    token: ${{ secrets.GITHUB_TOKEN }}
```

## Push to Docker Hub

Push images to Docker Hub instead of GHCR:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    images: my-org/my-app
    push-to-container-registry: dockerhub
    dockerhub-username: ${{ secrets.DOCKERHUB_USERNAME }}
    dockerhub-token: ${{ secrets.DOCKERHUB_TOKEN }}
    token: ${{ secrets.GITHUB_TOKEN }}
```

## Disable Registry Push

Build and scan without pushing to any registry:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    push-to-container-registry: ""
    token: ${{ secrets.GITHUB_TOKEN }}
```
