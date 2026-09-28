# A minimal image to test the mcvs-docker-action against. A distroless base is
# used as it contains virtually no packages, which keeps this test independent
# of the CVE churn of a distribution. 65532 is the unprivileged `nonroot` user
# that is part of the base image.
FROM gcr.io/distroless/static-debian13:nonroot@sha256:e2e927ec666bae08560abb3c55d0659eceabb657f56b6782ab500a9fc7f555e3

USER 65532
