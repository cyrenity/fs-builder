# fs-mod-builder

Multi-arch Debian 12 builder image for out-of-tree FreeSWITCH modules.

The image bundles a stripped-down FreeSWITCH install (headers, libs,
`freeswitch.pc` — no modules) at `/opt/freeswitch`, plus the usual build
toolchain (cmake, gcc, pkg-config) and the common `-dev` packages module
authors lean on (libcurl, libssl, libsqlite3, libspeex, libopus, libsndfile1,
libedit, libldns, libtiff).

## Tags

Tags follow `<fs-version>-debian<debian-version>`, with `latest` tracking the
most recent build.

| Tag                       | FS source         | Base            | Platforms              |
| ------------------------- | ----------------- | --------------- | ---------------------- |
| `1.10.12-debian12`        | `v1.10.12`        | `debian:12-slim`| `linux/amd64`, `linux/arm64` |
| `latest`                  | same as above     | same            | multi-arch manifest    |

Per-arch tags (`*-arm64`, `*-amd64`) also exist but consumers should prefer
the manifest tag so Docker picks the right one.

## Build & push

CI does this on push to `main` or via `workflow_dispatch`; locally:

```bash
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  --build-arg FS_VERSION=v1.10.12 \
  --build-arg DEBIAN_VERSION=12 \
  -t ghcr.io/cyrenity/fs-mod-builder:1.10.12-debian12 \
  -f Dockerfile.fs-builder \
  --push .
```

## Use from a module repo

```yaml
# .github/workflows/build.yml in your module repo
jobs:
  build:
    strategy:
      matrix:
        include:
          - { runner: ubuntu-latest,    suffix: amd64 }
          - { runner: ubuntu-24.04-arm, suffix: arm64 }
    runs-on: ${{ matrix.runner }}
    container:
      image: ghcr.io/cyrenity/fs-mod-builder:1.10.12-debian12
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v4
      - run: cmake -B build && cmake --build build -j
      - uses: actions/upload-artifact@v4
        with:
          name: mod-${{ matrix.suffix }}
          path: build/*.so
```

Because `pkg-config` already knows about FreeSWITCH (via `PKG_CONFIG_PATH`),
`pkg_check_modules(FREESWITCH REQUIRED freeswitch)` in a module's
`CMakeLists.txt` just works — no extra env vars needed.

## Bumping the FreeSWITCH version

Run the workflow from the Actions tab with `fs_version` set to the new tag
(e.g. `v1.10.13`). The Dockerfile clones the chosen tag from
`signalwire/freeswitch` and the multi-arch manifest is republished.

If consumer modules pin to a specific tag, update their workflow `image:` to
match. Modules pinned to `latest` pick up the new image on next CI run — make
sure the ABI is still compatible.
