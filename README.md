# fs-mod-builder

Multi-arch Debian builder image for out-of-tree FreeSWITCH modules.

Default target is **FreeSWITCH 1.11.0 on Debian Trixie (13)**. Other versions
and releases work via build-args / workflow inputs.

The image bundles:

- FreeSWITCH itself at `/opt/freeswitch` (headers, `libfreeswitch`, helper
  scripts, `freeswitch.pc`) — built `--disable-everything` so no bundled
  modules; we only need the ABI surface.
- The four source-only prerequisites FS 1.11 requires: **spandsp**,
  **sofia-sip**, **libks**, **signalwire-c** — pre-compiled and installed so
  consumer modules link cleanly.
- A full module-build toolchain: `cmake`, `gcc`, `pkg-config`, `autoconf`,
  `libtool` + `libtool-bin`, plus the usual `-dev` packages (`libcurl`,
  `libssl`, `libsqlite3`, `libpcre2`, `libspeex`, `libopus`, `libsndfile1`,
  `libedit`, `libldns`, `libtiff`, `liblua5.2`, `libpq`, `libavformat`,
  `libswscale`, `uuid`).

## Tags

Tags follow `<fs-version>-<debian-release>`, with `latest` tracking the most
recent build.

| Tag                | FS source  | Base                 | Platforms                    |
| ------------------ | ---------- | -------------------- | ---------------------------- |
| `1.11.0-trixie`    | `v1.11.0`  | `debian:trixie-slim` | `linux/amd64`, `linux/arm64` |
| `latest`           | same       | same                 | multi-arch manifest          |

Per-arch tags (`*-arm64`, `*-amd64`) also exist but consumers should prefer
the manifest tag so Docker picks the right one automatically.

## Build & push

CI does this on push to `main` or via `workflow_dispatch`; locally:

```bash
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  --build-arg FS_VERSION=v1.11.0 \
  --build-arg DEBIAN_RELEASE=trixie \
  -t ghcr.io/cyrenity/fs-mod-builder:1.11.0-trixie \
  -f Dockerfile.fs-builder \
  --push .
```

First build is slow (~45–90 min) because four source-only prereqs plus FS
itself all compile from scratch on each arch. Subsequent builds reuse the
GHA build cache and only re-run layers that changed — usually <10 min.

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
      image: ghcr.io/cyrenity/fs-mod-builder:1.11.0-trixie
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

Because `PKG_CONFIG_PATH` already points at FreeSWITCH and the source-built
prereqs, `pkg_check_modules(FREESWITCH REQUIRED freeswitch)` in a module's
`CMakeLists.txt` just works — no extra env vars needed.

## Bumping the FreeSWITCH version

Run the workflow from the Actions tab with `fs_version` set to the new tag
(e.g. `v1.11.1`). The Dockerfile clones the chosen tag from
`signalwire/freeswitch` and the multi-arch manifest is republished under the
new version tag and `latest`.

If consumer modules pin to a specific tag, update their workflow `image:` to
match. Modules pinned to `latest` pick up the new image on next CI run — make
sure the ABI is still compatible before promoting.

## Targeting a different Debian release

Set `debian_release` (workflow input) or `DEBIAN_RELEASE` (build-arg) to the
codename you want — e.g. `bookworm` for Debian 12. Note that FS 1.11's apt
deps assume modern package names (`libpcre2-dev`, `libsndfile1-dev`); older
releases may need different package names — adjust the Dockerfile if so.

## Source

Build deps and pre-build sequence follow the FreeSWITCH 1.11 Debian Trixie
install guide:
<https://www.iqaai.com/resources/freeswitch-v1-11-0-installation-guide>
