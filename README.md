# mcvs-docker-action

Create a `.github/workflows/docker.yml` file with the following content:

```bash
---
name: Docker
"on": push
jobs:
  mcvs-docker-action:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v4.1.1
      - uses: schubergphilis/mcvs-docker-action@v0.1.0
        with:
          dockle-accept-key: libcrypto3,libssl3
          token: ${{ secrets.GITHUB_TOKEN }}
```
