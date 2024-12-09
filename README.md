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

| Option               | Default | Required |
| :------------------- | :------ | -------- |
| build-args           |         |          |
| dockle-accept-key    | x       |          |
| images               | x       |          |
| token                | x       | x        |
| trivy-action-db      | x       |          |
| trivy-action-java-db | x       |          |

Note: If an **x** is registered in the Default column, refer to the
[action.yml](action.yml) for the corresponding value.
