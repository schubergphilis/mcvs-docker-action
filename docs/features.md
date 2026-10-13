# Features

This action provides a complete CI/CD pipeline for Docker images with the following capabilities:

- **Dockerfile Linting**: Static analysis using [hadolint](https://github.com/hadolint/hadolint) to enforce best practices
- **Automated Tagging**: Smart image tagging based on Git context (branches, tags, commits)
- **Multi-Architecture Builds**: `linux/amd64` and `linux/arm64` images are built with [Buildx](https://docs.docker.com/build/) and published as a single manifest list
- **Image Building**: Efficient Docker image building with support for build arguments
- **Image Linting**: CIS Docker benchmark compliance checking using [Dockle](https://github.com/goodwithtech/dockle)
- **Efficiency Analysis**: Layer-by-layer waste detection using [Dive](https://github.com/wagoodman/dive)
- **Multi-Scanner Security**: Triple-layer security scanning with:
  - **Grype**: Scans both source code and built images for vulnerabilities
- **Automated Registry Push**: Conditional push to GitHub Container Registry or Docker Hub on tagged releases
