# A minimal image to test the mcvs-docker-action against. A distroless base is
# used as it contains virtually no packages, which keeps this test independent
# of the CVE churn of a distribution. 65532 is the unprivileged `nonroot` user
# that is part of the base image.
FROM gcr.io/distroless/static-debian12:nonroot@sha256:afa5c872c891853ca7fcf1f12c3edb23f7eeef36189728842dd51042ff57f7ab

COPY test.txt /test.txt

USER 65532

CMD ["/test.txt"]
