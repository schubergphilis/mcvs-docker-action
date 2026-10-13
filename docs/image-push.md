# Image Push Behavior

Images are automatically pushed to the configured container registry **only when all conditions are met**:

1. The workflow is triggered by a `push` or a `workflow_dispatch` event (not
   pull requests)
2. The ref is a Git tag (matches `refs/tags/*`)
3. The `push-to-container-registry` input is set to `ghcr` or `dockerhub`

| Registry   | `push-to-container-registry` | `images` override needed?   | Credentials                              |
| ---------- | ---------------------------- | --------------------------- | ---------------------------------------- |
| GHCR       | `ghcr` (default)             | No                          | `token` (GITHUB_TOKEN)                   |
| Docker Hub | `dockerhub`                  | Yes (e.g., `my-org/my-app`) | `dockerhub-username` + `dockerhub-token` |
| None       | `""`                         | -                           | -                                        |

This ensures images are only published for tagged releases, keeping your registry clean and organized.

A manifest list covering every platform in the `platforms` input is pushed, so consumers automatically pull the image that matches their architecture. Note that the image is *built* for every platform on every run, which is what catches an architecture specific build failure on a pull request; only the push waits for a tag.

## Releasing tags created by a workflow

A tag that a workflow creates with `GITHUB_TOKEN`, for example with
[mcvs-general-action's auto-release](https://github.com/schubergphilis/mcvs-general-action/blob/main/docs/auto-release.md),
starts no `push` run, so the [Quickstart](../README.md#quickstart) workflow does
not run for it. Add `workflow_dispatch` to its triggers:

```yaml
"on":
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

and let the workflow that created the tag dispatch it on that tag, which
`GITHUB_TOKEN` may do. Its job needs `actions: write`:

```yaml
- env:
    GH_REPO: ${{ github.repository }}
    GH_TOKEN: ${{ github.token }}
    TAG: ${{ steps.release.outputs.tag }}
  if: steps.release.outputs.tag != ''
  run: gh workflow run docker.yml --ref "${TAG}"
```

The dispatched run has the tag as its ref, so the image gets the same tags as
on a push of that tag.

## Required Permissions

Your workflow needs these permissions to function correctly:

```yaml
permissions:
  contents: read # Required to checkout code
  packages: write # Required to push images to GHCR
```

The job that dispatches the workflow on a tag created by a workflow, see
[Releasing tags created by a workflow](#releasing-tags-created-by-a-workflow),
additionally needs:

```yaml
permissions:
  actions: write # Required to dispatch the workflow with gh workflow run
```
