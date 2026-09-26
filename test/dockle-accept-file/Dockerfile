# Exercises the dockle-accept-file input: settings.py is on Dockle's hard-coded
# list of suspicious file names, so CIS-DI-0010 fails this image unless the
# input reaches dockle-action. Same distroless base as test/Dockerfile.
FROM gcr.io/distroless/static-debian12:nonroot@sha256:afa5c872c891853ca7fcf1f12c3edb23f7eeef36189728842dd51042ff57f7ab

COPY settings.py /settings.py

USER 65532
