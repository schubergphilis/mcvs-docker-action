# Troubleshooting

## Dockle Reports False Positives for Package Versions

**Problem**: Dockle incorrectly identifies package names containing version numbers as leaked secrets.

**Solution**: Use the `dockle-accept-key` input to exclude specific packages:

```yaml
dockle-accept-key: libcrypto3,libssl3,python3.11
```

See [goodwithtech/dockle#250](https://github.com/goodwithtech/dockle/issues/250) for more details.

## Image Not Pushed to Registry

**Problem**: The image builds successfully but doesn't appear in the registry.

**Possible causes**:

1. Not using a tag: Push only occurs on tag events (e.g., `v1.0.0`)
2. Missing permissions: Ensure `packages: write` permission is set (GHCR)
3. Push disabled: Check if `push-to-container-registry` is set to `""`
4. Wrong registry: Ensure `push-to-container-registry` matches your intended registry (`ghcr` or `dockerhub`)
5. Docker Hub credentials: When using `dockerhub`, ensure both `dockerhub-username` and `dockerhub-token` are provided
6. Image name mismatch: When using Docker Hub, override `images` to match your Docker Hub repository (e.g., `my-org/my-app`)

## hadolint DL3029: Do Not Use --platform Flag with FROM

**Problem**: hadolint fails with [DL3029](https://github.com/hadolint/hadolint/wiki/DL3029) because a `FROM` instruction hardcodes a platform, e.g. `FROM --platform=linux/amd64 alpine`.

**Solution**: Do not hardcode the platform. The action passes the target platforms to Buildx, which sets `BUILDPLATFORM`, `TARGETPLATFORM`, `TARGETOS` and `TARGETARCH` for you. Only these variables are accepted by DL3029:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.24 AS build
ARG TARGETOS TARGETARCH
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app .
```

## Build Arguments Not Working

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

Do not pass credentials as build arguments: they are stored in the image history of the pushed image. Use a BuildKit secret mount instead.
