# Vaultwarden RISC-V 64 Build

![Platform](https://img.shields.io/badge/platform-linux%2Friscv64-blue)
![Base](https://img.shields.io/badge/base-debian%20trixie-a80030)
![License](https://img.shields.io/badge/license-MIT-green)

Automated cross-compilation of [Vaultwarden](https://github.com/dani-garcia/vaultwarden) for **linux/riscv64**.

Vaultwarden is an unofficial Bitwarden-compatible server written in Rust. The official
`vaultwarden/server` image is published for `amd64`, `arm64`, `arm/v7` and `arm/v6` only —
this repository fills the `riscv64` gap (OrangePi RV2, StarFive VisionFive 2, Milk-V, QEMU riscv64).

**No patches are applied to Vaultwarden.** The upstream source is cloned at the release tag and
built directly; the resulting image behaves exactly like the official Debian image
(same `/start.sh`, `/healthcheck.sh`, `/data` volume, port 80, same web vault).

New Vaultwarden releases are detected daily and built automatically.

## Quick Start

### Docker image

```bash
docker pull 12345qwert123456/vaultwarden-riscv64:latest

docker run -d --name vaultwarden \
  --restart unless-stopped \
  -v /vw-data/:/data/ \
  -p 80:80 \
  12345qwert123456/vaultwarden-riscv64:latest
```

All [upstream configuration options](https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview)
(`DOMAIN`, `ADMIN_TOKEN`, `SIGNUPS_ALLOWED`, `DATABASE_URL`, …) work as environment variables.

### Docker Compose

```yaml
services:
  vaultwarden:
    image: 12345qwert123456/vaultwarden-riscv64:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vw.domain.tld"
    volumes:
      - ./vw-data/:/data/
    ports:
      - 80:80
```

> Bitwarden clients require HTTPS — put Vaultwarden behind a reverse proxy
> (Caddy, nginx, Traefik) with a valid certificate.

### Download binaries

Pre-built binaries are published as GitHub Releases. Go to the [Releases](../../releases) page and
download `vaultwarden-<version>-linux-riscv64-bin.tar.gz` (the `vaultwarden` binary + `web-vault/`):

```bash
# Debian trixie / Ubuntu 24.04+ (glibc) runtime libraries
sudo apt install libpq5 libmariadb3 libssl3t64 ca-certificates
tar xzf vaultwarden-*-linux-riscv64-bin.tar.gz
WEB_VAULT_FOLDER=./web-vault DATA_FOLDER=./data ./vaultwarden
```

### Build locally

```bash
# Default version (VW_VERSION in the Dockerfile)
docker buildx build --platform linux/riscv64 -t vaultwarden:riscv64 \
  -f Dockerfile.vaultwarden --load .

# Specific version (also pass the web-vault digest pinned in upstream docker/DockerSettings.yaml)
docker buildx build --platform linux/riscv64 \
  --build-arg VW_VERSION=1.37.3 \
  --build-arg WEB_VAULT_DIGEST=sha256:ba8bab66d4330ab9dbafa8f245bcbe99cf6ee3f2c8ce9b5fbb10e9c49658451c \
  -t vaultwarden:1.37.3-riscv64 -f Dockerfile.vaultwarden --load .
```

> The Rust build runs natively on the build host and cross-compiles with `gcc-riscv64-linux-gnu`.
> Only the final stage's `apt-get install` executes riscv64 code, so QEMU binfmt must be registered
> (Docker Desktop ships it; on Linux: `docker run --privileged --rm tonistiigi/binfmt --install riscv64`).

## How It Works

Vaultwarden is pure Rust, but links against several C libraries: OpenSSL (`openssl-sys`),
libpq (`pq-sys`), MariaDB Connector/C (`mysqlclient-sys`), and bundled SQLite / zstd / `ring`
compiled by `cc`. Upstream cross-compiles with `tonistiigi/xx`, but its build matrix
(`docker/DockerSettings.yaml`) simply does not include `riscv64`.

Since Debian trixie has `riscv64` as an official release architecture, the riscv64 `-dev`
packages are installed via multiarch (`dpkg --add-architecture riscv64`) next to the
`gcc-riscv64-linux-gnu` cross toolchain, and Cargo builds for `riscv64gc-unknown-linux-gnu` —
with no patches to the Vaultwarden source code.

### Build stages

| Stage | Base image | Purpose |
|-------|------------|---------|
| `source` | `debian:trixie-slim` (build host) | Clones Vaultwarden once, resolves the Rust version from `rust-toolchain.toml` |
| `vault` | `vaultwarden/web-vault@sha256:…` (amd64, no RUN) | Static web vault files, same digest as upstream |
| `build` | `debian:trixie` (build host) | Cross-compiles `vaultwarden` (`sqlite,mysql,postgresql`) for riscv64 |
| final | `debian:trixie-slim` (riscv64) | Same runtime as upstream `Dockerfile.debian` |

### Key environment variables

```
CARGO_BUILD_TARGET=riscv64gc-unknown-linux-gnu
CARGO_TARGET_RISCV64GC_UNKNOWN_LINUX_GNU_LINKER=riscv64-linux-gnu-gcc
CC_riscv64gc_unknown_linux_gnu=riscv64-linux-gnu-gcc
PKG_CONFIG_ALLOW_CROSS=1
PKG_CONFIG_LIBDIR=/usr/lib/riscv64-linux-gnu/pkgconfig:/usr/share/pkgconfig
RUSTFLAGS=-C link-arg=-Wl,-rpath-link,/usr/lib/riscv64-linux-gnu   # transitive libs of libpq/libmariadb
```

## CI/CD

The GitHub Actions workflow (`.github/workflows/build-riscv64.yml`):
- Runs daily at 05:00 UTC (and on manual trigger)
- Fetches the latest Vaultwarden release and the web-vault digest pinned for it in `docker/DockerSettings.yaml`
- Checks if a riscv64 release already exists — skips if it does
- Builds the `linux/riscv64` image with Docker BuildKit
- Verifies the extracted binary is a genuine RISC-V ELF and runs `vaultwarden --version` under QEMU
- Publishes the binary + web vault (`.tar.gz` + SHA256) as a GitHub Release
- Pushes the Docker image to Docker Hub

Force a rebuild or build a specific version via `workflow_dispatch`.

### Required repository settings

| Type | Name | Description |
|------|------|-------------|
| Variable | `DOCKERHUB_USERNAME` | Docker Hub username |
| Secret | `DOCKERHUB_TOKEN` | Docker Hub access token |

## Requirements

- Docker with BuildKit (Docker Desktop or `docker buildx`) and QEMU binfmt for riscv64
- ~4 GB RAM
- ~10 minutes for a full build without cache (Rust cross-compile ~7.5 min, runtime `apt-get` under QEMU ~1.5 min)

## Versions

| Component | Version |
|-----------|---------|
| Vaultwarden | latest (auto-detected) |
| Web vault | from upstream `docker/DockerSettings.yaml` |
| Rust | from upstream `rust-toolchain.toml` (target `riscv64gc-unknown-linux-gnu`) |
| GCC cross | 14.2 (riscv64-linux-gnu, Debian trixie) |
| Runtime | `debian:trixie-slim` (riscv64) |

## Tested

- Vaultwarden 1.37.3 + web vault 2026.7.0, built on Windows 11 / Docker Desktop 29.7 (amd64 host)
- Image: `linux/riscv64`, ~86 MB; binary: `ELF 64-bit LSB pie executable, UCB RISC-V, RVC, double-float ABI`
- Run under QEMU riscv64: `vaultwarden --version`, `/alive`, `/api/config`, `/identity/accounts/prelogin`,
  web vault `index.html`, SQLite database creation in `/data` and `/healthcheck.sh` all OK

> Like the official image, the container refuses to start without a persistent `/data` volume
> (set `I_REALLY_WANT_VOLATILE_STORAGE=true` to override).

## License

Vaultwarden is licensed under [GNU AGPL v3](https://github.com/dani-garcia/vaultwarden/blob/main/LICENSE.txt).
Build files in this repository are MIT licensed.
