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
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v4.1.1
      - uses: schubergphilis/mcvs-docker-action@v0.1.0
        with:
          dockle-accept-key: libcrypto3,libssl3
          token: ${{ secrets.GITHUB_TOKEN }}
```

<!-- markdownlint-disable MD013 -->

| Option               | Default                              | Required | Description                                                                                                      |
| :------------------- | :----------------------------------- | -------- | :--------------------------------------------------------------------------------------------------------------- |
| dockle-accept-key    | 80                                   |          | Suppress certain environment variables in a docker image that are seen as secrets, but are not                   |
| token                | ' '                                  | x        | GitHub token that is required to push an image to the registry of the project and to pull cached Trivy DB images |
| trivy-action-db      | ghcr.io/aquasecurity/trivy-db:2      |          | Replace this with a cached image to prevent bump into pull rate limiting issues                                  |
| trivy-action-java-db | ghcr.io/aquasecurity/trivy-java-db:1 |          | Replace this with a cached image to prevent bump into pull rate limiting issues                                  |

<!-- markdownlint-enable MD013 -->
