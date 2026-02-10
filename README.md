# Git-Buildpackage GitHub Actions Workflows

Reusable GitHub Actions workflows for building Debian packages using git-buildpackage with native builds on multiple distributions and architectures.

## Workflows

### `gbp-native.yml` - Build Debian Packages

Builds Debian packages natively on specified distributions and architectures.

#### Build Features

- **Multi-distribution support**: Build on Debian, Ubuntu, or any Debian-based distro
- **Multi-architecture support**: Native builds on x64 (amd64) and ARM (arm64) using GitHub's runners
- **Configurable matrix**: Callers can specify custom combinations of images and architectures
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
- **`images`** (optional): JSON array as string of container images to build for
  - Default: `["ubuntu:latest", "debian:stable"]`
- **`architectures`** (optional): JSON array as string of architecture names to build for
  - Default: `["amd64"]`, available: `["amd64", "arm64"]`

#### Build Secrets

- **`GPG_PRIVATE_KEY`** (required): GPG private key for signing packages

#### Example: Custom Matrix

Build on multiple images and architectures:

```yaml
jobs:
  build:
    uses: dionysius/gbp-gha/.github/workflows/gbp-native.yml@main
    secrets:
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
    with:
      DEBFULLNAME: "Your Name"
      DEBEMAIL: "your.email@example.com"
      images: |
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

This will create 8 build jobs (4 images × 2 architectures).

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
      prerelease: false
      publish: false
```

#### Release Inputs

- **`prerelease`** (optional): Mark release as pre-release (default: `false`)
- **`publish`** (optional): Immediately publish the release (default: `false`)
- if both are false, release is marked as draft

#### Release Strategy

The workflow supports different release strategies:

1. **Draft for review** (default):

   ```yaml
   with:
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

### `promote-release.yml` - Promote Releases After Waiting Period

Promotes prereleases to full releases after a specified waiting period based on configurable grouping patterns.

#### Promotion Logic

The promotion workflow implements a staged release approach:

1. **Pattern-based grouping**: Releases are grouped based on the captured portion of the pattern (e.g., with pattern `debian/{*.*}.*`, releases like `debian/1.2.0` and `debian/1.2.1` both group by `1.2`)
2. **First release starts the clock**: The first release in a version group marks the starting day
3. **Waiting period**: All prereleases in that group wait for the specified number of days
4. **Automatic promotion**: After the waiting period, eligible prereleases are promoted to full releases

**Calendar day calculation**: Days are calculated based on calendar dates (midnight to midnight), ignoring the time of day. This ensures consistent behavior regardless of when the workflow runs during the day.

**Context-aware processing**:

- When called from a **tag context** (e.g., from a release trigger), only that specific tag is checked for promotion
- When called from a **branch context** (e.g., scheduled or manual trigger), all matching releases are processed

This allows time for bug fixes and testing while ensuring all releases eventually become fully published.

#### Promotion Usage

**As a reusable workflow** (call on-demand from your workflow):

```yaml
jobs:
  promote:
    uses: dionysius/gbp-gha/.github/workflows/promote-release.yml@main
    with:
      in_days: 7
      release_pattern: 'debian/{*.*}.*'
```

**With automatic daily scheduling** (create this in your repository):

```yaml
# In your repo: .github/workflows/promote-daily.yml
name: Daily Release Promotion

on:
  schedule:
    - cron: '0 3 * * *'  # Daily at 03:00 UTC
  workflow_dispatch:  # Allow manual trigger

jobs:
  promote:
    uses: dionysius/gbp-gha/.github/workflows/promote-release.yml@main
    with:
      in_days: 7
      release_pattern: 'debian/{*.*}.*'
```

#### Promotion Inputs

- **`in_days`** (optional): Number of days to wait after the first minor release before promotion (default: `7`)
- **`release_pattern`** (optional): Glob-style pattern to match and group releases (default: `'{debian/*.*}.*'`)
  - Use `{...}` to mark the captured portion that defines the version group
  - The captured group determines which releases share the same waiting period
  - Use `*` as a wildcard to match a single segment (excludes `/` and `.` characters)
  - Example patterns and grouping behavior:
    - `debian/{*.*}.*` matches `debian/1.2.0`, `debian/1.2.1`, groups by `1.2`
      - All debian releases with same minor version share one waiting period
    - `{debian/*.*}.*` matches `debian/1.2.0`, groups by `debian/1.2` (equivalent to above)
    - `*/{*.*}.*` matches `debian/1.2.0` and `test/1.2.0`, both group by `1.2`
      - All distros with same version **share one waiting period** (cross-distro grouping)
    - `{*/*.*}.*` matches `debian/1.2.0` and `test/1.2.0`, groups by `debian/1.2` and `test/1.2`
      - Each distro has **independent waiting periods** (separate grouping)
    - `v{*.*}.*` matches `v1.2.0`, `v1.2.1`, groups by `1.2`
    - `{*}.*.*` matches `1.0.0`, `1.1.0`, groups by `1` (major version only)

## Complete Example

```yaml
name: Build and Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    uses: dionysius/gbp-gha/.github/workflows/gbp-native.yml@main
    secrets:
      GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
    with:
      DEBFULLNAME: "Your Name"
      DEBEMAIL: "your.email@example.com"
      images: ["ubuntu:latest", "debian:stable"]
      architectures: ["amd64", "arm64"]

  release:
    needs: build
    uses: dionysius/gbp-gha/.github/workflows/release.yml@main
    with:
      prerelease: true
      publish: true

  # Optional: Promote immediately if enough time has passed
  promote:
    needs: release
    uses: dionysius/gbp-gha/.github/workflows/promote-release.yml@main
    with:
      in_days: 7
```

## Notes

- Artifacts are named as `results_{distro}_{arch}` (e.g., `results_ubuntu-latest_amd64`)
- Distro names use dashes within the name (e.g., `ubuntu-latest`, `debian-stable`)
- **Build strategy per distribution**: The first architecture is primary and builds everything (source + arch-dependent + arch-independent packages). Other architectures build only arch-dependent packages.
- The release workflow automatically handles multiple artifacts from different architectures
