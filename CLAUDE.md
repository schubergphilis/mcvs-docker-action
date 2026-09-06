# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

MCVS-docker-action is a GitHub composite action for Mission Critical Vulnerability Scanner (MCVS). It provides a comprehensive Docker image security and quality validation pipeline. This is not a traditional application with source code - it's a GitHub Action defined entirely in `action.yml`.

**Key files**:

- `action.yml`: The complete action definition (inputs, steps, logic)
- `merge/action.yml`: Second entry point that merges digests pushed by a matrix of per-platform jobs into one manifest list
- `README.md`: Comprehensive user documentation with examples
- `CLAUDE.md`: This file - guidance for Claude Code
- `.github/workflows/mcvs-pr-validation.yml`: PR validation workflow
- `.github/dependabot.yml`: Automated dependency updates

## Action Architecture

The action executes a sequential pipeline defined in `action.yml` (lines 62-283):

1. **Dockerfile Linting** (hadolint) - Static analysis of Dockerfile syntax and best practices
2. **Builder Setup** (docker/setup-qemu-action, docker/setup-buildx-action) - QEMU is only installed when `platforms != 'linux/amd64'`; buildx is always set up so all build steps share one builder and layer cache
3. **Platform Selection and Validation** (action.yml:88-121) - Picks the platform to scan (`linux/amd64` when present, else the first entry), emits a `::warning::` for every other platform, rejects a `push-by-digest` run that names more than one platform or image, and exports `scan_platform` plus the `platform_slug` used for the digest artifact name
4. **Metadata Extraction** (docker/metadata-action) - Generates image tags and labels from Git context
5. **Build Arguments Parsing** - Handles both single-line and multiline build-args inputs (uses env var for injection safety)
6. **Build + Scan** (action.yml:159-195) - Runs once, against `steps.validate.outputs.scan_platform`:
   - **Image Building** (docker/build-push-action) - Single-platform build with `load: true` and the fixed local tag `mcvs-docker-action:scan`
   - **Image Linting** (dockle) - Dynamic analysis of built image for CIS benchmarks
   - **Waste Detection** (dive) - Analyzes image layers for efficiency
   - **Image Scanning** (anchore/scan-action with Grype) - Scans built image for vulnerabilities
7. **Code Scanning** (anchore/scan-action with Grype) - Scans source code context for vulnerabilities; architecture independent
8. **Registry Login** (docker/login-action) - Conditional login to GHCR or Docker Hub
9. **Registry Push** - On tag push events either builds all platforms and pushes a manifest list (action.yml:232-246), or, when `push-by-digest` is set, pushes without a tag and uploads the digest as an artifact for `merge/action.yml` (action.yml:253-283). The two are mutually exclusive

## Key Design Patterns

### Build Arguments Handling

The action uses a special parsing step (action.yml:135-152) to support two input formats:

- Single-line: automatically formatted as `APPLICATION=value`
- Multiline: passed through as-is to support multiple build arguments

### Multi-Platform Builds

The `platforms` input (default `linux/amd64,linux/arm64`) drives three things:

- **Why buildx**: a multi-architecture image is a manifest list, which cannot be stored in the local Docker image store. That is why the push step is now `docker/build-push-action` with `push: true` instead of `docker push --all-tags`.
- **Why one scan build with `load: true`**: dockle, dive and the Grype image scan all read an image from the local Docker daemon, and only one image can be loaded, so a single platform is built separately under the local tag `mcvs-docker-action:scan`. Loading a non-native image is safe because these tools inspect layers and config, they never execute the image. `steps.meta.outputs.tags` is not used for this build since it can be multiline.
- **Why one platform is scanned rather than all of them**: composite actions cannot loop over `uses:` steps, so scanning N platforms means pasting the dockle/dive/Grype block N times - and for the same Dockerfile those three tools report the same findings per architecture, so the second copy costs a full QEMU build for no signal. `scan_platform` prefers `linux/amd64` because it needs no emulation. A consumer who does want every architecture scanned uses the matrix below, where each job scans the one platform it builds.

The push build re-runs every platform, but against the same buildx builder used by the scan build, so it is a cache hit. `provenance: false` is set to keep registry contents equivalent to the previous `docker push` behaviour - without it buildx adds attestation manifests that surface as `unknown/unknown` entries in GHCR.

### Matrix Builds With a Digest Merge

`platforms` accepting a single value means a consumer can already fan out one job per architecture, which GitHub runs in parallel and which allows `linux/arm64` to run on a native `ubuntu-24.04-arm` runner instead of under QEMU. What does not work in that shape is the push: jobs that each push the same tag overwrite one another and no manifest list results.

- **Why push by digest**: `push-by-digest=true` with `name-canonical=true` pushes an untagged image and returns its digest. Each job writes its digest to `${RUNNER_TEMP}/digests/<digest>` and uploads it as `mcvs-docker-action-digest-<platform-slug>`; the slug comes from the validation step so that matrix jobs cannot collide.
- **Why a second action**: `merge/action.yml` is a separate entry point (`schubergphilis/mcvs-docker-action/merge@<ref>`) rather than a mode of the main action, because it shares almost no steps with it - it only downloads the digests, recomputes the tags with the same `docker/metadata-action` configuration, logs in and runs `docker buildx imagetools create`.
- **Why the merge job is gated by the consumer**: `merge/action.yml` has no push conditions of its own. The consumer puts `if: startsWith(github.ref, 'refs/tags/')` on the merge job, which is one line in their workflow instead of a guard step plus a condition on all six steps here. The create step still fails explicitly when no digest artifacts were downloaded.
- **Why every job scans**: because each matrix job passes a single platform, `scan_platform` resolves to that platform, so the matrix shape scans `linux/amd64` and `linux/arm64` where the single-job shape scans only `linux/amd64`.
- **Constraints**: a job produces exactly one digest, so `push-by-digest` rejects more than one platform or image name. Every build job in a run must target the same `images` value, as the merge downloads every digest artifact of the run.
- **Not the default**: the single-job multi-platform path is unchanged and remains what a consumer gets without any extra configuration.

### Security Tool Integration

Three vulnerability scanners are used with different focuses:

- **Grype**: Used once for image scanning (action.yml:188-195) and once for code scanning (action.yml:199-206)
- **Dockle**: CIS Docker benchmark compliance with known ignores (action.yml:172-181)

### Conditional Push Logic

Login steps are conditional on the registry selection (action.yml:210-222):

- GHCR login: runs when `push-to-container-registry == 'ghcr'`
- Docker Hub login: runs when `push-to-container-registry == 'dockerhub'`

Images are only pushed when all conditions are met (action.yml:232-246 and 253-267):

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
- `platforms`: Comma separated target platforms, default `linux/amd64,linux/arm64`. `linux/arm64` is emulated with QEMU on the amd64 runner, which makes tagged releases noticeably slower; consumers opt out with `platforms: linux/amd64`. Exactly one platform is scanned per job.
- `push-by-digest`: Default `"false"`. Switches the tag push for a digest push plus a digest artifact, which is what makes a matrix of per-platform jobs mergeable. Requires exactly one platform and one image name per job.
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

1. Add new step in the appropriate section of action.yml:62-283
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

1. Modify the conditional in action.yml:233-237
2. Update README.md Image Push Behavior section with new conditions
3. Add troubleshooting entry if the change might confuse users
4. Update CLAUDE.md Conditional Push Logic section

## Dependabot Configuration

Dependabot is configured to update all GitHub Actions weekly in a single grouped PR (see `.github/dependabot.yml`). This means action version pins in `action.yml` are automatically maintained.

## Ignored Security Checks

Two Dockle CIS checks are permanently ignored (action.yml:175-180):

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
