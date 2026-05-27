# fs-mod-builder

Multi-arch Debian builder image for out-of-tree FreeSWITCH modules.

CI publishes two variants in parallel:

- **`1.11.0-trixie`** — FreeSWITCH 1.11.0 on Debian Trixie (13)
- **`1.10.12-bookworm`** — FreeSWITCH 1.10.12 on Debian Bookworm (12)

Both variants are multi-arch (`linux/amd64` + `linux/arm64`) and bundle:

- FreeSWITCH itself at `/opt/freeswitch` (headers, `libfreeswitch`, helper
  scripts, `freeswitch.pc`) — built `--disable-everything` so no bundled
  modules; just the ABI surface module repos link against.
- The four source-only prerequisites FS needs since the bundled tree was
  removed: **spandsp**, **sofia-sip**, **libks**, **signalwire-c**. The
  `1.10.12-bookworm` variant uses a pinned spandsp commit
  (`0d2e6ac…`) that matches FS 1.10.12's ABI expectations; `1.11.0-trixie`
  tracks spandsp `master`.
- A full module-build toolchain: `cmake`, `gcc`, `pkg-config`, `autoconf`,
  `libtool` + `libtool-bin`, plus the usual `-dev` packages (`libcurl`,
  `libssl`, `libsqlite3`, `libpcre2`/`libpcre3`, `libspeex`, `libopus`,
  `libsndfile1`, `libedit`, `libldns`, `libtiff`, `liblua5.2`, `libpq`,
  `libavformat`, `libswscale`, `uuid`).

## Tags

| Tag                  | FS source  | Base                   | Platforms                    |
| -------------------- | ---------- | ---------------------- | ---------------------------- |
| `1.11.0-trixie`      | `v1.11.0`  | `debian:trixie-slim`   | `linux/amd64`, `linux/arm64` |
| `1.10.12-bookworm`   | `v1.10.12` | `debian:bookworm-slim` | `linux/amd64`, `linux/arm64` |

Per-arch tags (`*-arm64`, `*-amd64`) also exist but consumers should prefer
the manifest tag so Docker picks the right architecture automatically.

## Build & push

CI handles this. On push to `main` (with Dockerfile or workflow changes)
both variants are built and published. Manual triggers via `workflow_dispatch`
can scope to a single variant via the `variants` input.

Local equivalent (one variant at a time):

```bash
# Trixie / FS 1.11
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  --build-arg FS_VERSION=v1.11.0 \
  --build-arg DEBIAN_RELEASE=trixie \
  --build-arg SPANDSP_REF=master \
  -t ghcr.io/cyrenity/fs-mod-builder:1.11.0-trixie \
  -f Dockerfile.fs-builder --push .

# Bookworm / FS 1.10.12
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  --build-arg FS_VERSION=v1.10.12 \
  --build-arg DEBIAN_RELEASE=bookworm \
  --build-arg SPANDSP_REF=0d2e6ac65e0e8f53d652665a743015a88bf048d4 \
  --build-arg EXTRA_CFLAGS="-Wno-error -Wno-implicit-function-declaration -Wno-incompatible-pointer-types" \
  --build-arg EXTRA_CXXFLAGS="-Wno-error" \
  -t ghcr.io/cyrenity/fs-mod-builder:1.10.12-bookworm \
  -f Dockerfile.fs-builder --push .
```

First build of each variant is slow (~45–90 min) because four source-only
prereqs plus FS itself compile from scratch per arch. Subsequent builds reuse
the GHA cache and re-run only what changed.

## Use from a module repo

Pin to whichever variant matches your runtime:

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - { runner: ubuntu-latest,    suffix: amd64 }
          - { runner: ubuntu-24.04-arm, suffix: arm64 }
    runs-on: ${{ matrix.runner }}
    container:
      image: ghcr.io/cyrenity/fs-mod-builder:1.11.0-trixie  # or 1.10.12-bookworm
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

`PKG_CONFIG_PATH` is set in the image so
`pkg_check_modules(FREESWITCH REQUIRED freeswitch)` works without extra env.

## Bumping FreeSWITCH

For a new patch release (e.g. `v1.11.1`), edit the matrix entry's
`fs_version` in `.github/workflows/builder-image.yml` and push. The Dockerfile
clones whatever tag is named.

Major upgrades (e.g. `v1.12`) may need different deps or different
warning-suppression flags — check the upstream install guide first and
adjust the matrix entry's `extra_cflags` / `spandsp_ref` accordingly.

## Sources

Build deps and pre-build sequence follow the FreeSWITCH install guides:

- [FreeSWITCH 1.11.0 on Debian Trixie](https://www.iqaai.com/resources/freeswitch-v1-11-0-installation-guide)
- [FreeSWITCH 1.10.12 on Debian 12](https://www.iqaai.com/resources/freeswitch-v1-10-12-installation-guide)
