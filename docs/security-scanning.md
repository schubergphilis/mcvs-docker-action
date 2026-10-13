# Security Scanning

This action employs a defense-in-depth approach with multiple security scanners:

## Grype Scanning

- **Code Scan**: Scans source code in the build context for vulnerabilities
  - Severity cutoff: HIGH
  - Reports unfixed vulnerabilities
- **Image Scan**: Scans the built Docker image for vulnerabilities
  - Severity cutoff: HIGH
  - Reports unfixed vulnerabilities
  - Runs once per job, against the scanned platform

## Dive Efficiency Analysis

- Analyzes image layers for wasted space
- Runs in CI mode (`--ci`) and reads thresholds from a `.dive-ci` file in your
  repository root when present
- Example `.dive-ci` configuration:

```yaml
rules:
  lowestEfficiency: 0.95
  highestWastedBytes: 20MB
  highestUserWastedPercent: 0.20
```

See the [Dive CI documentation](https://github.com/wagoodman/dive#ci-integration)
for all available options.

## Dockle Linting

- Validates CIS Docker benchmarks
- Permanently ignores:
  - `CIS-DI-0005`: Content trust (not achievable on public runners)
  - `CIS-DI-0006`: HEALTHCHECK (left to consumer discretion)
