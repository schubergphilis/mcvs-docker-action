# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

MCVS-docker-action is a GitHub composite action for Mission Critical Vulnerability Scanner (MCVS). It provides a comprehensive Docker image security and quality validation pipeline. This is not a traditional application with source code - it's a GitHub Action defined entirely in `action.yml`.

**Key files**:

- `action.yml`: The complete action definition (inputs, steps, logic)
- `README.md`: Comprehensive user documentation with examples
- `CLAUDE.md`: This file - guidance for Claude Code
- `.github/workflows/mcvs-pr-validation.yml`: PR validation workflow
- `.github/dependabot.yml`: Automated dependency updates

## Action Architecture

The action executes a sequential pipeline defined in `action.yml` (lines 55-253):

1. **Dockerfile Linting** (hadolint) - Static analysis of Dockerfile syntax and best practices
2. **Builder Setup** (docker/setup-qemu-action, docker/setup-buildx-action) - QEMU is only installed when `platforms != 'linux/amd64'`; buildx is always set up so all build steps share one builder and layer cache
3. **Platform Validation** - Emits a `::warning::` for platforms that are built but not scanned (anything other than `linux/amd64` and `linux/arm64`)
4. **Metadata Extraction** (docker/metadata-action) - Generates image tags and labels from Git context
5. **Build Arguments Parsing** - Handles both single-line and multiline build-args inputs (uses env var for injection safety)
6. **Per-architecture Build + Scan** - Repeated for `linux/amd64` (action.yml:127-163) and `linux/arm64` (action.yml:167-203), each guarded by `contains(inputs.platforms, ...)`:
   - **Image Building** (docker/build-push-action) - Single-platform build with `load: true` and a fixed local tag `mcvs-docker-action:scan-<platform>`
   - **Image Linting** (dockle) - Dynamic analysis of built image for CIS benchmarks
   - **Waste Detection** (dive) - Analyzes image layers for efficiency
   - **Image Scanning** (anchore/scan-action with Grype) - Scans built image for vulnerabilities
7. **Code Scanning** (anchore/scan-action with Grype) - Scans source code context for vulnerabilities; architecture independent, runs once
8. **Registry Login** (docker/login-action) - Conditional login to GHCR or Docker Hub
9. **Registry Push** (docker/build-push-action) - Builds all platforms and pushes a manifest list on tag push events

## Key Design Patterns

### Build Arguments Handling

The action uses a special parsing step (action.yml:103-120) to support two input formats:

- Single-line: automatically formatted as `APPLICATION=value`
- Multiline: passed through as-is to support multiple build arguments

### Multi-Platform Builds

The `platforms` input (default `linux/amd64,linux/arm64`) drives three things:

- **Why buildx**: a multi-architecture image is a manifest list, which cannot be stored in the local Docker image store. That is why the push step is now `docker/build-push-action` with `push: true` instead of `docker push --all-tags`.
- **Why per-architecture builds with `load: true`**: dockle, dive and the Grype image scan all read an image from the local Docker daemon, so each architecture is built separately with a distinct local tag (`mcvs-docker-action:scan-linux-amd64` / `-arm64`). Loading a non-native image is safe because these tools inspect layers and config, they never execute the image. `steps.meta.outputs.tags` is not used for these builds since it can be multiline and would collide between architectures.
- **Why the scan steps are duplicated instead of looped**: composite actions cannot loop over `uses:` steps. Duplicating the dockle/dive/Grype block per architecture keeps the marketplace actions (and therefore their Dependabot-managed pins) in place. The consequence is that only `linux/amd64` and `linux/arm64` are scanned; other platforms are built and pushed with a warning.

The push build re-runs every platform, but against the same buildx builder used by the scan builds, so it is a cache hit. `provenance: false` is set to keep registry contents equivalent to the previous `docker push` behaviour - without it buildx adds attestation manifests that surface as `unknown/unknown` entries in GHCR.

### Security Tool Integration

Three vulnerability scanners are used with different focuses:

- **Grype**: Used once per built architecture for image scanning (action.yml:155-163 and 195-203) and once for code scanning (action.yml:207-214)
- **Dockle**: CIS Docker benchmark compliance with known ignores, once per built architecture (action.yml:138-149 and 178-189)

### Conditional Push Logic

Login steps are conditional on the registry selection (action.yml:218-230):

- GHCR login: runs when `push-to-container-registry == 'ghcr'`
- Docker Hub login: runs when `push-to-container-registry == 'dockerhub'`

Images are only pushed when all conditions are met (action.yml:240-253):

- Event is a push (not PR)
- Reference contains `refs/tags/` (tagged release)
- Input `push-to-container-registry` is not empty (supports both `ghcr` and `dockerhub`)

## Testing This Action

Since this is a GitHub Action, testing means:

1. Create a test workflow in `.github/workflows/` that uses the action
2. The action can reference itself using the current branch or commit SHA
3. Example test workflow pattern from README.md (lines 18-47)

To test local changes before pushing:

- Reference the action using the current branch: `schubergphilis/mcvs-docker-action@feature-branch`
- Or use a local path in a workflow: `uses: ./` (when the workflow is in the same repo)

## Important Inputs

- `images`: Default is `ghcr.io/${{ github.repository }}`. Override when using Docker Hub (e.g., `my-org/my-app`), custom image names, or matrix builds with suffixes.
- `build-args`: Supports both single-line (auto-formatted as `APPLICATION=value`) and multiline (passed as-is) formats.
- `dockle-accept-key`: Workaround for false positives when specific package versions trigger Dockle's secret detection (see goodwithtech/dockle#250).
- `platforms`: Comma separated target platforms, default `linux/amd64,linux/arm64`. `linux/arm64` is emulated with QEMU on the amd64 runner, which makes tagged releases noticeably slower; consumers opt out with `platforms: linux/amd64`. Only `linux/amd64` and `linux/arm64` are scanned.
- `push-to-container-registry`: Set to `ghcr` (default), `dockerhub`, or empty string `""` to disable pushing entirely.
- `dockerhub-username` and `dockerhub-token`: Required when `push-to-container-registry` is `dockerhub`. Typically sourced from `${{ secrets.DOCKERHUB_USERNAME }}` and `${{ secrets.DOCKERHUB_TOKEN }}`.
- `token`: Required for pushing to GHCR authentication. Typically `${{ secrets.GITHUB_TOKEN }}`.

## Common Modification Patterns

### Adding a New Input

1. Add input definition to `action.yml` inputs section with description and optional default
2. Use the input in the appropriate step with `${{ inputs.input-name }}`
3. Update README.md input parameters table
4. Add usage example to README.md if the input enables a new use case
5. Update CLAUDE.md Important Inputs section if there are special considerations

### Adding a New Security Tool

1. Add new step in the appropriate section of action.yml:55-253
2. Consider placement in the pipeline (static analysis before build, dynamic after)
3. Add description to README.md Features section
4. Add configuration details to README.md Security Scanning section
5. Update CLAUDE.md Action Architecture section with step number and purpose

### Modifying Scanner Configuration

1. Update the relevant step in action.yml
2. Document the change in README.md Security Scanning section
3. If behavior changes significantly, add to README.md Troubleshooting section
4. Update CLAUDE.md if the design pattern or rationale changes

### Changing Push Behavior

1. Modify the conditional in action.yml:241-244
2. Update README.md Image Push Behavior section with new conditions
3. Add troubleshooting entry if the change might confuse users
4. Update CLAUDE.md Conditional Push Logic section

## Dependabot Configuration

Dependabot is configured to update all GitHub Actions weekly in a single grouped PR (see `.github/dependabot.yml`). This means action version pins in `action.yml` are automatically maintained.

## Ignored Security Checks

Two Dockle CIS checks are permanently ignored, in both per-architecture dockle steps (action.yml:143-148 and 183-188):

- **CIS-DI-0005**: Content trust - not achievable on public GitHub runners
- **CIS-DI-0006**: HEALTHCHECK - intentionally left to action consumers to implement

## Workflow Validation

The repository uses `schubergphilis/mcvs-pr-validation-action` on all PRs (`.github/workflows/mcvs-pr-validation.yml`) to enforce PR standards.

## Documentation Conventions

### README.md

- User-facing documentation with comprehensive examples
- Should include: quick start, usage examples, input parameters table, troubleshooting
- Examples should be copy-paste ready and demonstrate common use cases
- Keep technical details balanced - enough to understand but not overwhelming

### CLAUDE.md

- Technical reference for Claude Code
- Should include: architecture details, design patterns, line number references to code
- Focus on "why" decisions were made, not just "what" the code does
- Update when adding new features or changing architectural patterns

### Keeping Docs in Sync

- When adding new inputs to action.yml, update both README.md (user table) and CLAUDE.md (technical notes if needed)
- When changing behavior, update README.md examples and CLAUDE.md design patterns section
- README.md is the source of truth for user documentation
- CLAUDE.md is the source of truth for implementation guidance
