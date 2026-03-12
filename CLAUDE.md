# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

KasmVNC is a web-based VNC server that provides remote desktop access via modern browsers. It diverges from the RFB specification to support modern technologies (WebRTC, H.264, WebP). It does NOT support legacy VNC viewer applications. Built on top of X.org's X server with custom VNC extensions.

## Build Commands

### Docker-based build (recommended)

```bash
git submodule init && git submodule update --remote --merge
sudo docker build -t kasmvnc:dev -f builder/dockerfile.ubuntu_jammy.dev .
sudo docker run -it --rm -v ./:/src -p 2222:22 -p 6901:6901 -p 8443:8443 \
  --device=/dev/dri/card0 --device=/dev/dri/renderD128 \
  --group-add video --group-add render --name kasmvnc_dev kasmvnc:dev
```

**Host UID must be 1000** (matches container UID).

Inside the container:
```bash
cd kasmweb && npm install && npm run build && cd ..   # frontend (only on changes)
builder/build.sh                                       # build KasmVNC (Xvnc)
```

### Running the dev server (inside container)

```bash
/src/xorg.build/bin/Xvnc -interface 0.0.0.0 -PublicIP 127.0.0.1 -disableBasicAuth \
  -RectThreads 0 -Log *:stdout:100 -httpd /src/kasmweb/dist -sslOnly 0 \
  -SecurityTypes None -websocketPort 6901 -FreeKeyMappings :1 &
/usr/bin/xfce4-session --display :1
```

For live frontend development with hot reload, use `npm run serve` in kasmweb/ with nginx proxying (port 8443).

### Package building

```bash
builder/build-package <os> <codename>         # e.g., ubuntu jammy
builder/build-and-test-deb ubuntu focal       # build + test in one step
builder/build-deb debian bookworm             # just the deb (needs tarball first)
builder/build-tarball debian bookworm         # just the tarball
```

Packages output to `builder/build/`.

### CMake flags

- `-DCMAKE_BUILD_TYPE=Debug` — debug build
- `-DBUILD_STATIC=1` — semi-static portable build
- `-DTESTS=ON` — build C++ performance tests

## Testing

### Python spec tests (BDD with Mamba)

```bash
./run-specs                              # all specs
./run-specs spec/vncserver_spec.py       # single spec file
./run-specs -d                           # with debug output (KASMVNC_SPEC_DEBUG_OUTPUT=1)
./run-specs -v                           # verbose (test descriptions)
```

Dependencies managed via Pipfile (mamba, expects, pexpect). Specs are in `spec/`.

### Package testing

```bash
builder/test-deb ubuntu focal            # install & run in fresh container
builder/test-deb ubuntu focal -s         # shell into test container for debugging
builder/test-deb ubuntu focal -p         # performance tests
builder/test-deb-barebones ubuntu focal  # test deps without XFCE
```

Test users: `foo`/`foobar` (r/w), `foo-ro`/`foobar` (view-only), `foo-owner`/`foobar` (admin).

### vncserver development environment

```bash
builder/devenv-vncserver                 # start dockerized devenv
# Inside devenv:
ty                                       # alias for running python specs
ty -d                                    # specs with debug output
ty -v                                    # verbose spec descriptions
```

Uses `./unix/vncserver` (not system-installed) and local config files.

## Architecture

### Core C++ Libraries (`common/`)

- **`rfb/`** — RFB protocol implementation: encoders (Tight, ZRLE, Hextile, H.264), pixel formats, security handlers, server/client state machines. This is the largest component (~200 files).
- **`network/`** — WebSocket server, WebRTC/UDP transport (`webudp/`), TLS support.
- **`rdr/`** — Low-level byte stream I/O (readers, writers, compression wrappers).
- **`os/`** — OS abstraction (threads, mutexes, timing).
- **`Xregion/`** — Rectangle/region math for screen updates.

### Xvnc Server (`unix/xserver/hw/vnc/`)

The VNC X server extension. Built by patching X.org server source (1.19.x/1.20.x) with KasmVNC's VNC extension. Linked against the `common/` libraries. Build uses autotools (`Makefile.am`) and links to `libvnc.a` built by CMake.

### Server Management (`unix/`)

- **`vncserver`** — Main session manager (Perl, ~3100 lines). Handles starting/stopping Xvnc, user creation, desktop environment selection, YAML config parsing.
- **`KasmVNC/`** — Perl modules used by vncserver.
- **`kasmxproxy/`** — X proxy service.
- **`vncpasswd/`, `kasmvncpasswd/`** — Password management tools.

### Web Frontend (`kasmweb/` submodule)

noVNC-based client (JS/TS). Built with Vite/webpack. `npm run build` for production, `npm run serve` for dev. Output goes to `kasmweb/dist/`.

### Configuration

YAML-based. System config: `/etc/kasmvnc/kasmvnc.yaml`. Per-user: `~/.vnc/kasmvnc.yaml`. User config overrides system config.

## CI/CD

GitLab CI (`.gitlab-ci.yml`). Stages: www → build → functional_test → run_test → test → upload. Builds 17+ distros × 2 architectures. Ubuntu Jammy is required for functional and spec tests.

`BUILD_DISTROS_REGEX` variable filters which distros to build (e.g., `"jammy"` for fast iteration).

## Compiler Requirements

- C standard: gnu99, C++ standard: C++20
- gcc/g++ 11+ required (gcc 12 has known bugs)
- OpenMP support required
- NDEBUG is removed even in Release builds (asserts always active)
