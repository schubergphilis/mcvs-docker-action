# docker-action

Create a `.github/workflows/docker.yml` file with the following content:

```bash
---
name: Docker
"on": push
jobs:
  docker:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v4.1.1
      - uses: schubergphilis.com/docker-action@v0.1.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```
