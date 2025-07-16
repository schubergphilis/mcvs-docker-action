# MCVS-docker-action

Mission Critical Vulnerability Scanner (MCVS) Docker Action is a custom
[GitHub Action](https://github.com/features/actions) that consists of the
following steps:

- YAML linting.
- Dockerfile linting.
- Determining image name and tag.
- Docker image building.
- Docker image linting.
- Detecting waste in the docker image.
- Code and docker image security scanning using Grype and Trivy.
- Logging in and pushing the image to GitHub packages.

## Usage

Create a `.github/workflows/docker.yml` file with the following content:

```yaml
---
name: Docker
"on": push
permissions:
  contents: read
  packages: write
jobs:
  mcvs-docker-action:
    strategy:
      matrix:
        args:
          - build-args: some-app
            context: some/path/to/Dockerfile/home
            image-suffix: ""
          - build-args: some-app-cli
            image-suffix: /some-app-cli
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4.2.2
      - uses: schubergphilis/mcvs-docker-action@v0.1.0
        with:
          build-args: ${{ matrix.args.build-args }}
          images: |-
            ghcr.io/${{ github.repository }}${{ matrix.args.image-suffix }}
          dockle-accept-key: libcrypto3,libssl3
          token: ${{ secrets.GITHUB_TOKEN }}
```

| Option                     | Default | Required |
| :------------------------- | :------ | -------- |
| build-args                 |         |          |
| context                    | x       |          |
| dockle-accept-key          | x       |          |
| grype-version              |         |          |
| images                     | x       |          |
| push-to-container-registry | x       |          |
| token                      | x       |          |
| trivy-action-db            | x       |          |
| trivy-action-java-db       | x       |          |

Note: If an **x** is registered in the Default column, refer to the
[action.yml](action.yml) for the corresponding value.
