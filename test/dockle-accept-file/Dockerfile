# Exercises the dockle-accept-file input: settings.py is on Dockle's hard-coded
# list of suspicious file names, so CIS-DI-0010 fails this image unless the
# input reaches dockle-action. Same distroless base as test/Dockerfile.
FROM gcr.io/distroless/static-debian13:nonroot@sha256:e2e927ec666bae08560abb3c55d0659eceabb657f56b6782ab500a9fc7f555e3

COPY settings.py /settings.py

USER 65532
