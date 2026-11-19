# snarkos-docker

Docker images for [snarkOS](https://github.com/aleo-labs-protocol/snarkOS), built for `linux/amd64` and `linux/arm64`.

## Images

The **Build and Push SnarkOS** workflow (`.github/workflows/build-snarkos-features.yml`) builds one of two images from a snarkOS tag or commit:

| Image choice | Dockerfile | Tag | Entrypoint |
|---|---|---|---|
| `snarkos` | `service/snarkos/Dockerfile` | `<tag>` | `/aleo/bin/snarkos` |
| `ledger-distribution` | `service/ledger-distribution/Dockerfile` | `<tag>-ledger-distribution` | `/aleo/bin/snarkos`; the image also has `/aleo/bin/ledger-distribution` |

Both Dockerfiles use two stages:
- **Builder** compiles snarkOS (and ledger-distribution, for the ledger-distribution image) with `cargo`.
- **Runtime** is `ubuntu:24.04` with the runtime dependencies and the binaries.

`ledger-distribution` uploads a node's blocks to the block CDN bucket (see its [README](https://github.com/aleo-labs-protocol/snarkOS/tree/staging/ledger-distribution)). The crate exists in snarkOS from `testnet-v4.13.0`; the build fails for older tags. Run it from the same snarkOS tag as the node whose ledger it reads:

```bash
docker run --rm --entrypoint /aleo/bin/ledger-distribution <image>:<tag>-ledger-distribution <network ID>
```

## Workflow

Run it by hand (`workflow_dispatch`) with:

| Input | Meaning |
|---|---|
| `commit_or_tag` | snarkOS commit SHA or tag to build |
| `tag_name` | image tag; when empty, the first snarkOS tag that contains the commit |
| `features` | `snarkos` or `ledger-distribution` |

The workflow builds each architecture on its own self-hosted runner (`snarkos-amd64-builder`, `snarkos-arm64-builder`) and pushes `<DOCKERHUB_SNARKOS_REPO>:<tag>-amd64` and `-arm64` to Docker Hub, then a multi-arch manifest under the plain tag `<DOCKERHUB_SNARKOS_REPO>:<tag>`.

It needs these repository settings:

| Kind | Name |
|---|---|
| Variable | `DOCKERHUB_SNARKOS_REPO` (for example `aleohq/snarkos`) |
| Secret | `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` |

## Manual build

```bash
git clone git@github.com:aleo-labs-protocol/snarkos-docker.git
cd snarkos-docker

docker build \
  --build-arg COMMIT_OR_TAG=<tag-or-commit> \
  -f service/snarkos/Dockerfile \
  -t snarkos:<tag> .

docker run --rm snarkos:<tag> --version
docker run --rm -it -v "$PWD/data:/aleo/data" snarkos:<tag> start
```

For the ledger-distribution image use `-f service/ledger-distribution/Dockerfile`. To build for another architecture, use [`docker buildx`](https://docs.docker.com/build/building/multi-platform/).
