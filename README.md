# Build Images

Docker-based build infrastructure for C++/Qt and Python projects with multi-platform support, artifact hosting, and
CI/CD integration.

## Overview

This repository provides the tooling and Docker images required to build, package, and distribute C++ and Python
applications across Linux (x86\_64 and aarch64) and Windows (x86\_64). It supports:

- **C++ builds** with native and cross-compilation toolchains (GCC, MinGW, MSVC via Wine).
- **Qt framework** built from source for multiple platforms and architectures.
- **Python executable packaging** with Wine support for Windows targets.
- **Self-hosted artifact storage** using Sonatype Nexus (Docker registry and raw files) and MinIO (S3-compatible object
  storage).
- **CI/CD integration** with GitLab Runner and Docker Model Runner for local AI models.

## Components

### C++ Build Environment

The primary build image (`gnu-cpp`) is based on Ubuntu 24.04 and includes:

| Category      | Tools                                                              |
|---------------|--------------------------------------------------------------------|
| Compilers     | GCC, G++, MinGW (cross-compile to Windows), aarch64 cross-compiler |
| Build systems | CMake, Ninja, GNU Make                                             |
| Debugging     | GDB, GDBserver, Valgrind                                           |
| Code quality  | Clang-Format, Doxygen, Graphviz, Gcovr                             |
| Packaging     | CPack, dpkg, RPM, NSIS                                             |
| Runtime       | Wine (Windows emulation), Python 3, OpenJDK                        |
| Utilities     | Git, SSH, bindfs, fuse-zip, jq, ccache                             |

**Dockerfiles:**

| File                        | Purpose                                       |
|-----------------------------|-----------------------------------------------|
| `builder/cpp.Dockerfile`    | Base C++ build image with toolchains and Wine |
| `builder/qt.Dockerfile`     | Adds Qt libraries to the base image           |
| `builder/python.Dockerfile` | Standalone Python build image with Wine       |
| `sshd/sshd.Dockerfile`      | Minimal SSH server container                  |

**Build script:** [`cpp-builder.sh`](cpp-builder.sh)

```bash
# Build the base image without Qt
./cpp-builder.sh --qt-ver '' build

# Build the image with a specific Qt version
./cpp-builder.sh --qt-ver '6.10.1' build

# Push to Nexus registry
./cpp-builder.sh --qt-ver '6.10.1' push

# Run the container with X11 forwarding
./cpp-builder.sh --project /path/to/project runx
```

### Python Build Environment

Builds Python executables for Linux and Windows (via Wine).

**Build script:** [`py-builder.sh`](py-builder.sh)

```bash
# Build the Python image
./py-builder.sh build

# Run a Python project inside the container
./py-builder.sh --project /path/to/project run -- ./script.py
```

### Qt Framework from Source

Builds the Qt framework for multiple platforms and architectures.

**Build scripts:**

| Script                                 | Language | Purpose                                                                       |
|----------------------------------------|----------|-------------------------------------------------------------------------------|
| [`build-qt-lib.sh`](build-qt-lib.sh)   | Bash     | Step-by-step Qt build workflow (deps, clone, init, conf, build, install, zip) |
| [`build-qt-lib.py`](build-qt-lib.py)   | Python   | INI-configured Qt build with Docker and Wine support                          |
| [`build-qt-lib.ini`](build-qt-lib.ini) | INI      | Configuration for compilers, features, and modules                            |

**Qt library directory structure:**

```
lib/qt
├── lnx-aarch64/6.10.1/gcc_64
├── lnx-x86_64/6.10.1/gcc_64
├── w64-x86_64/6.10.1/
│   ├── mingw_64
│   └── msvc_64
└── win-x86_64/6.10.1/mingw_64
```

**Build steps:**

```bash
# 1. Build the base C++ image without Qt
./cpp-builder.sh --qt-ver '' build

# 2. Start the container and build Qt inside it
./build-qt-lib.sh start
./build-qt-lib.sh attach

# Inside the container:
./build.sh deps
./build.sh clone
./build.sh init
./build.sh conf
./build.sh fix
./build.sh build
./build.sh install
./build.sh zip

# 3. Package and upload the Qt libraries
./cpp-builder.sh qt-lnx
./cpp-builder.sh qt-lnx-up

# 4. Build the final image with Qt
./cpp-builder.sh --qt-ver '6.10.1' build
```

See also: [Building the Qt Library from Source](doc/qt-from-source.md)

### Artifact Storage

#### Sonatype Nexus

Self-hosted registry for Docker images and raw file storage (Qt libraries, toolchains, Wine registry files).

**Script:** [`nexus-docker.sh`](nexus-docker.sh)

```bash
# Login to Nexus Docker registry
./nexus-docker.sh login

# List images on Nexus
./nexus-docker.sh list

# Upload compressed Wine registry
./nexus-docker.sh wine-reg-up
```

**Credentials:** `.nexus-credentials` (not tracked by Git, template: `tpl.nexus-credentials`)

#### MinIO

S3-compatible object storage used for GitLab Runner cache.

**Script:** [`minio.sh`](minio.sh)

```bash
# Start the MinIO server
./minio.sh start

# Use the MinIO CLI
./minio.sh mc -- ls -r vps/my-bucket
```

**Credentials:** `.minio-credentials` (not tracked by Git, template: `tpl.minio-credentials`)

### SSH Server Container

A lightweight container running `sshd` on port 3022 for remote access to build environments.

**Script:** [`sshd.sh`](sshd.sh)

```bash
./sshd.sh start
./sshd.sh stop
```

### Docker Model Runner

Runs local AI/LLM models using the Docker Model Runner plugin.

**Script:** [`dmr.sh`](dmr.sh)

```bash
# Install dependencies and start the model runner
./dmr.sh deps
./dmr.sh start

# Test the API
./dmr.sh test
```

### Utility Scripts

| Script                                                 | Purpose                                                     |
|--------------------------------------------------------|-------------------------------------------------------------|
| [`build-gcc.sh`](build-gcc.sh)                         | Build GCC from source (download, configure, build, install) |
| [`ubuntu.sh`](ubuntu.sh)                               | Run a generic Ubuntu 24.04 container with X11 support       |
| [`test-nexus-docker-reg.sh`](test-nexus-docker-reg.sh) | Inspect and manage Docker images on Nexus                   |
| [`test-sync_nexus.sh`](test-sync_nexus.sh)             | Sync artifacts between Nexus instances                      |

### Supporting Files

| Path                        | Purpose                                                                                |
|-----------------------------|----------------------------------------------------------------------------------------|
| `builder/bin/entrypoint.sh` | Container entrypoint — handles user UID/GID mapping, Wine prefix, and library mounting |
| `builder/bin/versions.sh`   | Prints installed tool versions inside the container                                    |
| `builder/apt-repo-gpg/`     | GPG keys for the Nexus APT repository                                                  |
| `builder/wine-reg/`         | Wine registry files for Windows emulation                                              |
| `builder/debian-pkgs/`      | Custom Debian packages (e.g., clang-format fix)                                        |
| `inc/`                      | Shared shell library functions                                                         |
| `project/`                  | Mount point for project source code                                                    |
| `minio/`                    | MinIO server configuration and data directories                                        |

## Configuration

### Credentials

Two credential files are required (not tracked by Git):

| File                 | Purpose                                      | Template                |
|----------------------|----------------------------------------------|-------------------------|
| `.nexus-credentials` | Nexus server URL, user, and password         | `tpl.nexus-credentials` |
| `.minio-credentials` | MinIO server URL, access key, and secret key | `tpl.minio-credentials` |

Copy the template and fill in the values:

```bash
cp tpl.nexus-credentials .nexus-credentials
cp tpl.minio-credentials .minio-credentials
```

### CLion Integration

The C++ build image can be used with JetBrains CLion for containerized development.

**Docker run arguments:**

```
--rm --privileged --user 0:0 --env USER_LOCAL="<uid>:<gid>"
```

For GUI applications, add X11 forwarding:

```
--env DISPLAY --volume "${HOME}/.Xauthority:/home/user/.Xauthority:ro"
```

> Debugging is not supported due to the entrypoint script. Console applications run normally.

## Architecture Support

| Host    | Native Builds | Cross-Compilation                           |
|---------|---------------|---------------------------------------------|
| x86\_64 | Linux x86\_64 | Linux aarch64, Windows x86\_64 (MinGW/MSVC) |
| aarch64 | Linux aarch64 | Linux aarch64 only                          |

## Documentation

- [Building the Qt Library from Source](doc/qt-from-source.md) — detailed Qt build guide
- [Semantic Versioning](doc/semantic-versioning.md) — commit message format and version bumping rules
- [Build Process Diagram](doc/cpp-builder-process.puml) — PlantUML diagram of the C++ build workflow
