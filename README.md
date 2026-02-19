# MCVS-docker-action

[![GitHub release](https://img.shields.io/github/v/release/schubergphilis/mcvs-docker-action)](https://github.com/schubergphilis/mcvs-docker-action/releases)
[![License](https://img.shields.io/github/license/schubergphilis/mcvs-docker-action)](LICENSE)

Mission Critical Vulnerability Scanner (MCVS) Docker Action is a comprehensive GitHub Action that provides a complete Docker image security and quality validation pipeline. This action combines multiple industry-standard tools to ensure your Docker images meet security and quality standards before deployment.

## Features

This action provides a complete CI/CD pipeline for Docker images with the following capabilities:

- **Dockerfile Linting**: Static analysis using [hadolint](https://github.com/hadolint/hadolint) to enforce best practices
- **Automated Tagging**: Smart image tagging based on Git context (branches, tags, commits)
- **Image Building**: Efficient Docker image building with support for build arguments
- **Image Linting**: CIS Docker benchmark compliance checking using [Dockle](https://github.com/goodwithtech/dockle)
- **Efficiency Analysis**: Layer-by-layer waste detection using [Dive](https://github.com/wagoodman/dive)
- **Multi-Scanner Security**: Triple-layer security scanning with:
  - **Grype**: Scans both source code and built images for vulnerabilities
  - **Trivy**: Additional image scanning with configurable vulnerability databases
- **Automated Registry Push**: Conditional push to GitHub Container Registry on tagged releases

## Quick Start

Create a `.github/workflows/docker.yml` file in your repository:

```yaml
---
name: Docker
on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: schubergphilis/mcvs-docker-action@v0.1.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

## Usage Examples

### Basic Usage

For a simple single Dockerfile project:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```

### Custom Build Arguments

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

### Matrix Builds

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

### Custom Dockerfile Location

If your Dockerfile is not in the repository root:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    context: ./docker/production
    token: ${{ secrets.GITHUB_TOKEN }}
```

### Handling Dockle False Positives

When Dockle incorrectly flags specific package versions as secrets:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    dockle-accept-key: libcrypto3,libssl3
    token: ${{ secrets.GITHUB_TOKEN }}
```

### Using Alternative Trivy Databases

Use custom OCI repositories for Trivy vulnerability databases:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    trivy-action-db: ghcr.io/my-org/trivy-db:2
    trivy-action-java-db: ghcr.io/my-org/trivy-java-db:1
    token: ${{ secrets.GITHUB_TOKEN }}
```

### Disable Registry Push

Build and scan without pushing to any registry:

```yaml
- uses: schubergphilis/mcvs-docker-action@v0.1.0
  with:
    push-to-container-registry: ""
    token: ${{ secrets.GITHUB_TOKEN }}
```

## Input Parameters

| Parameter                    | Required | Default                                       | Description                                                                                                                                                                                     |
| ---------------------------- | -------- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token`                      | No       | -                                             | GitHub token for pushing images to GHCR. Use `${{ secrets.GITHUB_TOKEN }}`                                                                                                                      |
| `build-args`                 | No       | -                                             | Docker build arguments. Single-line values are formatted as `APPLICATION=value`. Multiline values are passed as-is                                                                              |
| `context`                    | No       | `.`                                           | Directory containing the Dockerfile and build context                                                                                                                                           |
| `images`                     | No       | `ghcr.io/${{ github.repository }}`            | Image name(s) for tagging. Supports multiline for multiple registries                                                                                                                           |
| `push-to-container-registry` | No       | `ghcr`                                        | Registry to push to. Set to `""` to disable pushing                                                                                                                                             |
| `dockle-accept-key`          | No       | -                                             | Comma-separated list of package names to exclude from Dockle secret detection. Use for known false positives (see [goodwithtech/dockle#250](https://github.com/goodwithtech/dockle/issues/250)) |
| `grype-version`              | No       | latest                                        | Specific version of Grype to use for vulnerability scanning                                                                                                                                     |
| `trivy-action-db`            | No       | `public.ecr.aws/aquasecurity/trivy-db:2`      | OCI repository for Trivy vulnerability database                                                                                                                                                 |
| `trivy-action-java-db`       | No       | `public.ecr.aws/aquasecurity/trivy-java-db:1` | OCI repository for Trivy Java vulnerability database                                                                                                                                            |

## Security Scanning

This action employs a defense-in-depth approach with multiple security scanners:

### Grype Scanning

- **Code Scan**: Scans source code in the build context for vulnerabilities
  - Severity cutoff: HIGH
  - Reports unfixed vulnerabilities
- **Image Scan**: Scans the built Docker image for vulnerabilities
  - Severity cutoff: HIGH
  - Reports unfixed vulnerabilities

### Trivy Scanning

- Scans the built image with focus on critical vulnerabilities
- Severity levels: CRITICAL and HIGH
- Only reports vulnerabilities with available fixes (`ignore-unfixed: true`)
- Supports custom vulnerability database sources
- Respects `.trivyignore` file for known exceptions

### Dockle Linting

- Validates CIS Docker benchmarks
- Permanently ignores:
  - `CIS-DI-0005`: Content trust (not achievable on public runners)
  - `CIS-DI-0006`: HEALTHCHECK (left to consumer discretion)

## Image Push Behavior

Images are automatically pushed to GitHub Container Registry **only when all conditions are met**:

1. The workflow is triggered by a `push` event (not pull requests)
2. The push is to a Git tag (matches `refs/tags/*`)
3. The `push-to-container-registry` input is set to `ghcr` (default)

This ensures images are only published for tagged releases, keeping your registry clean and organized.

## Required Permissions

Your workflow needs these permissions to function correctly:

```yaml
permissions:
  contents: read # Required to checkout code
  packages: write # Required to push images to GHCR
```

## Troubleshooting

### Dockle Reports False Positives for Package Versions

**Problem**: Dockle incorrectly identifies package names containing version numbers as leaked secrets.

**Solution**: Use the `dockle-accept-key` input to exclude specific packages:

```yaml
dockle-accept-key: libcrypto3,libssl3,python3.11
```

See [goodwithtech/dockle#250](https://github.com/goodwithtech/dockle/issues/250) for more details.

### Image Not Pushed to Registry

**Problem**: The image builds successfully but doesn't appear in GHCR.

**Possible causes**:

1. Not using a tag: Push only occurs on tag events (e.g., `v1.0.0`)
2. Missing permissions: Ensure `packages: write` permission is set
3. Push disabled: Check if `push-to-container-registry` is set to `""`

### Build Arguments Not Working

**Problem**: Build arguments are not being passed correctly to Docker build.

**Solution**: For single values, use plain text (automatically formatted as `APPLICATION=value`):

```yaml
build-args: my-app
```

For multiple arguments, use multiline format:

```yaml
build-args: |
  ARG1=value1
  ARG2=value2
```

### Trivy Database Connection Issues

**Problem**: Trivy fails to download vulnerability databases.

**Solution**: Use custom database locations with `trivy-action-db` and `trivy-action-java-db` inputs, pointing to accessible OCI repositories.

## Contributing

Contributions are welcome. Please open an issue or pull request for bugs, features, or improvements.

## License

See [LICENSE](LICENSE) file for details.
