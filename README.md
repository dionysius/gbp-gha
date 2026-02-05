# Git-Buildpackage GitHub Actions Workflows

Reusable GitHub Actions workflows for building Debian packages using git-buildpackage with native builds on multiple distributions and architectures.

## Workflows

### `gbp-native.yml` - Build Debian Packages

Builds Debian packages natively on specified distributions and architectures.

#### Build Features

- **Multi-distribution support**: Build on Debian, Ubuntu, or any Debian-based distro
- **Multi-architecture support**: Native builds on x64 (amd64) and ARM (arm64) using GitHub's runners
- **Configurable matrix**: Callers can specify custom combinations of distros and architectures
- **GPG signing**: Automatically signs packages with provided GPG key
- **Artifact upload**: Uploads built packages as workflow artifacts

#### Build Usage

```yaml
jobs:
  build:
    uses: dionysius/gbp-gha/.github/workflows/gbp-native.yml@main
    secrets:
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
    with:
      DEBFULLNAME: "Your Name"
      DEBEMAIL: "your.email@example.com"
```

#### Build Inputs

- **`DEBFULLNAME`** (required): Full name for Debian changelog
- **`DEBEMAIL`** (required): Email for Debian changelog
- **`before_build_deps_install`** (optional): Shell commands to run before installing build dependencies
- **`distros`** (optional): JSON array of container images to build for
  - Default: `["ubuntu:latest", "debian:stable"]`
  - Can be simple strings (container names) or objects with `container` and optional `distro` keys
  - Example: `["ubuntu:24.04", "debian:12"]`
- **`architectures`** (optional): JSON array of architecture names to build for
  - Default: `["amd64"]`
  - ARM architectures (`arm64`, `armhf`, `armv7`) use GitHub's ARM runners (`ubuntu-24.04-arm`)
  - All other architectures use x64 runners (`ubuntu-latest`)
  - Example: `["amd64", "arm64"]`

#### Build Secrets

- **`GPG_PRIVATE_KEY`** (required): GPG private key for signing packages

#### Example: Custom Matrix

Build on multiple distros and architectures:

```yaml
jobs:
  build:
    uses: dionysius/gbp-gha/.github/workflows/gbp-native.yml@main
    secrets:
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
    with:
      DEBFULLNAME: "Your Name"
      DEBEMAIL: "your.email@example.com"
      distros: |
        [
          "ubuntu:24.04",
          "ubuntu:22.04",
          "debian:12",
          "debian:11"
        ]
      architectures: |
        [
          "amd64",
          "arm64"
        ]
```

This will create 8 build jobs (4 distros × 2 architectures).

### `release.yml` - Create GitHub Release

Creates a GitHub release with all built artifacts from the build workflow.

#### Release Features

- **Draft releases**: Create releases as drafts first (default)
- **Immutable releases**: Upload all artifacts to a single release
- **Conditional publishing**: Choose whether to publish immediately or keep as draft
- **Pre-release support**: Mark releases as pre-releases
- **Automatic changelog**: Extracts changelog from debian/changelog

#### Release Usage

```yaml
jobs:
  build:
    uses: dionysius/gbp-gha/.github/workflows/gbp-native.yml@main
    secrets:
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
    with:
      DEBFULLNAME: "Your Name"
      DEBEMAIL: "your.email@example.com"

  release:
    needs: build
    uses: dionysius/gbp-gha/.github/workflows/release.yml@main
    with:
      draft: true
      prerelease: false
      publish: false
```

#### Release Inputs

- **`draft`** (optional): Create release as draft (default: `true`)
- **`prerelease`** (optional): Mark release as pre-release (default: `false`)
- **`publish`** (optional): Immediately publish the release (default: `false`)
  - When `true`, overrides `draft` to publish immediately
  - When `false`, respects the `draft` setting

#### Release Strategy

The workflow supports different release strategies:

1. **Draft for review** (default):

   ```yaml
   with:
     draft: true
     publish: false
   ```

   Creates a draft release that you can review and manually publish later.

2. **Immediate release**:

   ```yaml
   with:
     publish: true
   ```

   Publishes the release immediately.

3. **Pre-release**:

   ```yaml
   with:
     prerelease: true
     publish: true
   ```

   Publishes as a pre-release immediately.

## Complete Example

```yaml
name: Build and Release

on:
  push:
    tags:
      - 'v*'

# Optional: Only needed if default workflow permissions are restricted
# permissions:
#   contents: write

jobs:
  build:
    uses: dionysius/gbp-gha/.github/workflows/gbp-native.yml@main
    secrets:
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
    with:
      DEBFULLNAME: "Your Name"
      DEBEMAIL: "your.email@example.com"
      distros: ["ubuntu:latest", "debian:stable"]
      architectures: ["amd64", "arm64"]

  release:
    needs: build
    uses: dionysius/gbp-gha/.github/workflows/release.yml@main
    with:
      draft: true
      prerelease: false
      publish: false
```

## Notes

- Artifacts are named as `results_{distro}_{arch}` (e.g., `results_ubuntu-latest_amd64`)
- Distro names use dashes within the name (e.g., `ubuntu-latest`, `debian-stable`)
- **Build strategy per distribution**: The first architecture is primary and builds everything (source + arch-dependent + arch-independent packages). Other architectures build only arch-dependent packages.
- The release workflow automatically handles multiple artifacts from different architectures
