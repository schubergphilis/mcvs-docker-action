# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

MCVS-docker-action is a GitHub composite action for Mission Critical Vulnerability Scanner (MCVS). It provides a comprehensive Docker image security and quality validation pipeline. This is not a traditional application with source code - it's a GitHub Action defined entirely in `action.yml`.

## Action Architecture

The action executes a sequential pipeline defined in `action.yml` (lines 46-159):

1. **Dockerfile Linting** (hadolint) - Static analysis of Dockerfile syntax and best practices
2. **Metadata Extraction** (docker/metadata-action) - Generates image tags and labels from Git context
3. **Build Arguments Parsing** - Handles both single-line and multiline build-args inputs
4. **Image Building** (docker/build-push-action) - Builds Docker image without pushing
5. **Image Linting** (dockle) - Dynamic analysis of built image for CIS benchmarks
6. **Waste Detection** (dive) - Analyzes image layers for efficiency
7. **Code Scanning** (anchore/scan-action with Grype) - Scans source code context for vulnerabilities
8. **Image Scanning** (anchore/scan-action with Grype) - Scans built image for vulnerabilities
9. **Trivy Scanning** (aquasecurity/trivy-action) - Additional image vulnerability scanning
10. **Registry Push** (docker/login-action + docker push) - Pushes to GHCR only on tag push events

## Key Design Patterns

### Build Arguments Handling
The action uses a special parsing step (action.yml:68-83) to support two input formats:
- Single-line: automatically formatted as `APPLICATION=value`
- Multiline: passed through as-is to support multiple build arguments

### Security Tool Integration
Three vulnerability scanners are used with different focuses:
- **Grype**: Used twice - once for code scanning (action.yml:114-120), once for image scanning (action.yml:121-128)
- **Trivy**: Additional image scanning with configurable DB sources (action.yml:129-142)
- **Dockle**: CIS Docker benchmark compliance with known ignores (action.yml:95-104)

### Conditional Push Logic
Images are only pushed when all conditions are met (action.yml:153-156):
- Event is a push (not PR)
- Reference contains `refs/tags/` (tagged release)
- Input `push-to-container-registry` equals `ghcr`

## Testing This Action

Since this is a GitHub Action, testing means:

1. Create a test workflow in `.github/workflows/` that uses the action
2. The action can reference itself using the current branch or commit SHA
3. Example test workflow pattern from README.md (lines 18-47)

To test local changes before pushing:
- Reference the action using the current branch: `schubergphilis/mcvs-docker-action@feature-branch`
- Or use a local path in a workflow: `uses: ./` (when the workflow is in the same repo)

## Important Inputs

- `images`: Default is `ghcr.io/${{ github.repository }}`. Override when using custom image names or matrix builds with suffixes.
- `dockle-accept-key`: Workaround for false positives when specific package versions trigger Dockle's secret detection (see goodwithtech/dockle#250).
- `trivy-action-db` and `trivy-action-java-db`: Allow using alternative OCI repositories for vulnerability databases.
- `push-to-container-registry`: Set to empty string `""` to disable pushing entirely.

## Dependabot Configuration

Dependabot is configured to update all GitHub Actions weekly in a single grouped PR (see `.github/dependabot.yml`). This means action version pins in `action.yml` are automatically maintained.

## Ignored Security Checks

Two Dockle CIS checks are permanently ignored (action.yml:98-102):
- **CIS-DI-0005**: Content trust - not achievable on public GitHub runners
- **CIS-DI-0006**: HEALTHCHECK - intentionally left to action consumers to implement

## Workflow Validation

The repository uses `schubergphilis/mcvs-pr-validation-action` on all PRs (`.github/workflows/mcvs-pr-validation.yml`) to enforce PR standards.
