# nfrastack/gha - Reusable GitHub Actions for Container Builds

Reusable workflows for building multi-architecture Docker container images with native ARM64 support, automatic tagging, and build summaries.

## Workflows

| Workflow                      | Used by                 | Purpose                                              |
| ----------------------------- | ----------------------- | ---------------------------------------------------- |
| `container-build_generic.yml` | 47 repos                | Generic + PHP-dependent images                       |
| `container-build_base.yml`    | container-base          | Base distro images with module selection             |
| `container-build_multi.yml`   | container-nginx         | Images with image_variant (core/faflopza/faflopzaze) |
| `container-build_phpfpm.yml`  | container-nginx-php-fpm | PHP-FPM images with image_variant                    |
| `artifacts-encrypt.yml`       | All repos               | Encrypt sensitive files before build                 |
| `artifacts-remove.yml`        | All repos               | Clean up encrypted artifacts after build             |

## Quick Start

### Simple image (single OS, no PHP)

This is the most common pattern.

```yaml
# .github/workflows/main.yml
on:
  push:
    paths:
      - '**'
      - '!CHANGELOG.md'
      - '!LICENSE'
      - '!README.md'

jobs:
  prepare:
    uses: nfrastack/gha/.github/workflows/artifacts-encrypt.yml@main
    secrets: inherit

  build:
    needs: prepare
    strategy:
      matrix:
        include:
          - { distro: "alpine", distro_variant: "3.23", latest: "true", distro_latest: "true", arch: "linux/amd64,linux/arm64" }
    uses: nfrastack/gha/.github/workflows/container-build_generic.yml@main
    with:
      base_image: "ghcr.io/nfrastack/container-base"
      distro: ${{ matrix.distro }}
      distro_variant: ${{ matrix.distro_variant }}
      latest: ${{ matrix.latest }}
      distro_latest: ${{ matrix.distro_latest }}
      platforms: ${{ matrix.arch }}
      base_image_version: ${{ matrix.base_image_version || '' }}
    secrets: inherit

  cleanup:
    needs: [ build ]
    uses: nfrastack/gha/.github/workflows/artifacts-remove.yml@main
    secrets: inherit
```

**Resulting tags on main push:** `latest`, `alpine`, `alpine_3.23`
**On tag push (e.g. `8-3.0.4`):** `8-3.0.4`, `8-3.0.4-alpine_3.23`, `latest`, `alpine`

### Multiple OS builds

Add more entries to the matrix. Each entry builds independently.

```yaml
strategy:
  matrix:
    include:
      - { distro: "alpine", distro_variant: "3.23", latest: "true",  distro_latest: "true",  arch: "linux/amd64,linux/arm64" }
      - { distro: "alpine", distro_variant: "3.22", latest: "false", distro_latest: "false", arch: "linux/amd64,linux/arm64" }
      - { distro: "debian", distro_variant: "trixie", latest: "false", distro_latest: "true", arch: "linux/amd64,linux/arm64" }
```

**Tags on main push:**
- alpine 3.23: `latest`, `alpine`, `alpine_3.23`
- alpine 3.22: `alpine_3.22`
- debian trixie: `debian`, `debian_trixie`

### PHP-dependent image
Uses `php_version` and `php_version_latest` inputs.

```yaml
# .github/workflows/image_build.yml
name: build
on:
  workflow_call

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        build:
          - { php_version: "8.4", php_version_latest: "true",  distro: "alpine", distro_variant: "3.23", latest: "false", distro_latest: "true",  arch: "linux/amd64,linux/arm64" }
          - { php_version: "8.3", php_version_latest: "true",  distro: "alpine", distro_variant: "3.23", latest: "true",  distro_latest: "true",  arch: "linux/amd64,linux/arm64" }
          - { php_version: "8.2", php_version_latest: "true",  distro: "alpine", distro_variant: "3.22", latest: "false", distro_latest: "false", arch: "linux/amd64,linux/arm64" }
    uses: nfrastack/gha/.github/workflows/container-build_generic.yml@main
    with:
      base_image: "ghcr.io/nfrastack/container-nginx-php-fpm"
      php_version: ${{ matrix.build.php_version }}
      php_version_latest: ${{ matrix.build.php_version_latest }}
      distro: ${{ matrix.build.distro }}
      distro_variant: ${{ matrix.build.distro_variant }}
      latest: ${{ matrix.build.latest || 'false' }}
      distro_latest: ${{ matrix.build.distro_latest || 'false' }}
      platforms: ${{ matrix.build.arch || 'linux/amd64,linux/arm64' }}
      base_image_version: ${{ matrix.build.base_image_version || '' }}
    secrets: inherit
```

## Understanding `latest` and Tag Hierarchy

When a user runs `docker pull nfrastack/phpapplication` (no tag), Docker pulls `:latest`. You control which matrix entry produces `:latest` by setting `latest: "true"` on exactly ONE entry.

### Example: Application with PHP 8.2, 8.3, 8.4

You want:
- `docker pull nfrastack/application` to get PHP **8.3** on alpine 3.23
- Users who want PHP 8.4 can pull `nfrastack/application:php8.4`
- Users who want PHP 8.2 can pull `nfrastack/application:php8.2`

```yaml
matrix:
  build:
    # PHP 8.4 - newest, available but not the default
    - { php_version: "8.4", php_version_latest: "true",  latest: "false", distro: "alpine", distro_variant: "3.23", distro_latest: "true",  arch: "linux/amd64,linux/arm64" }

    # PHP 8.3 - the recommended default, gets :latest
    - { php_version: "8.3", php_version_latest: "true",  latest: "true",  distro: "alpine", distro_variant: "3.23", distro_latest: "true",  arch: "linux/amd64,linux/arm64" }

    # PHP 8.2 - still supported
    - { php_version: "8.2", php_version_latest: "true",  latest: "false", distro: "alpine", distro_variant: "3.22", distro_latest: "true",  arch: "linux/amd64,linux/arm64" }
```

**Tags produced on main push:**

| Matrix entry                       | Tags                                                   |
| ---------------------------------- | ------------------------------------------------------ |
| PHP 8.4, alpine 3.23               | `php8.4-alpine_3.23`, `php8.4`, `alpine`               |
| PHP 8.3, alpine 3.23 (latest=true) | `php8.3-alpine_3.23`, `php8.3`, **`latest`**, `alpine` |
| PHP 8.2, alpine 3.22               | `php8.2-alpine_3.22`, `php8.2`                         |

So `docker pull nfrastack/application` gets PHP 8.3.

### Adding Debian builds alongside Alpine

When you add Debian variants for the same PHP version, use `distro_latest` to control which distro gets the bare distro tag:

```yaml
matrix:
  build:
    # Alpine builds
    - { php_version: "8.4", php_version_latest: "true",  distro: "alpine", distro_variant: "3.23", latest: "false", distro_latest: "true",  arch: "linux/amd64,linux/arm64" }
    - { php_version: "8.3", php_version_latest: "true",  distro: "alpine", distro_variant: "3.23", latest: "true",  distro_latest: "true",  arch: "linux/amd64,linux/arm64" }

    # Debian builds - note php_version_latest is FALSE (Alpine is the default for each PHP version)
    - { php_version: "8.4", php_version_latest: "false", distro: "debian", distro_variant: "trixie", latest: "false", distro_latest: "true",  arch: "linux/amd64,linux/arm64" }
    - { php_version: "8.3", php_version_latest: "false", distro: "debian", distro_variant: "trixie", latest: "false", distro_latest: "false", arch: "linux/amd64,linux/arm64" }
```

Key: `php_version_latest: "false"` on Debian entries means `docker pull nfrastack/application:php8.4` pulls the **Alpine** variant, not Debian. Users who want Debian must be explicit: `nfrastack/application:php8.4-debian_trixie`.

## Input Reference

### container-build_generic.yml

| Input                | Type    | Default                   | Description                                                                 |
| -------------------- | ------- | ------------------------- | --------------------------------------------------------------------------- |
| `base_image`         | string  | -                         | Upstream base image registry path (e.g. `ghcr.io/nfrastack/container-base`) |
| `distro`             | string  | -                         | Linux distribution (e.g. `alpine`, `debian`)                                |
| `distro_variant`     | string  | -                         | Distribution version (e.g. `3.23`, `trixie`)                                |
| `image_variant`      | string  | -                         | Image variant identifier                                                    |
| `tag`                | string  | -                         | Custom tag prefix                                                           |
| `latest`             | string  | `"true"`                  | Add `:latest` tag on main/master                                            |
| `distro_latest`      | string  | `"false"`                 | Add bare distro tag (e.g. `:alpine`)                                        |
| `php_version`        | string  | -                         | PHP version (e.g. `8.3`). Enables PHP-aware tagging                         |
| `php_version_latest` | string  | `"false"`                 | Add bare PHP tag (e.g. `:php8.3` or `:8.3`)                                 |
| `platforms`          | string  | `linux/amd64,linux/arm64` | Target architectures (comma-separated)                                      |
| `push_dockerhub`     | boolean | `true`                    | Push to DockerHub                                                           |
| `push_ghcr`          | boolean | `true`                    | Push to GHCR                                                                |
| `build_args`         | string  | -                         | Extra build-args (newline-separated `KEY=VALUE`)                            |
| `base_image_version` | string  | -                         | Pin base image to a version. See [Pinning](#pinning-base-image-versions)    |
| `base_image_digest`  | string  | -                         | Pin base image to SHA digest. Overrides tag entirely                        |

### How `latest`, `distro_latest`, and `php_version_latest` interact

These three flags control the tag hierarchy. Set each to `"true"` on exactly ONE matrix entry to avoid tag conflicts:

| Flag                 | What it controls | Example tag                  | Rule                                                          |
| -------------------- | ---------------- | ---------------------------- | ------------------------------------------------------------- |
| `latest`             | `:latest`        | `nfrastack/bookstack:latest` | ONE entry total (the overall default)                         |
| `distro_latest`      | `:{distro}`      | `nfrastack/bookstack:alpine` | ONE entry per distro (e.g. alpine 3.23 gets it, 3.22 doesn't) |
| `php_version_latest` | `:{php_version}` | `nfrastack/bookstack:php8.3` | ONE entry per PHP version (typically the Alpine build)        |

### Pinning base image versions

By default, images pull the `:latest` equivalent of their upstream base. To pin to a specific release, add `base_image_version` to your matrix entry:

```yaml
# Simple image (e.g. redis) — pins container-base to release 2026.3.3
# Pulls: ghcr.io/nfrastack/container-base:2026.3.3-alpine_3.23
matrix:
  include:
    - { distro: "alpine", distro_variant: "3.23", latest: "true", arch: "linux/amd64,linux/arm64", base_image_version: "2026.3.3" }

# PHP image (e.g. bookstack) — pins container-nginx-php-fpm to release 8.0.2
# Pulls: ghcr.io/nfrastack/container-nginx-php-fpm:8.4-8.0.2-alpine_3.23
matrix:
  build:
    - { php_version: "8.4", distro: "alpine", distro_variant: "3.23", ..., base_image_version: "8.0.2" }
```

Tag format with `base_image_version`:

- **Simple images:** `{version}-{distro}_{variant}` (e.g. `2026.3.3-alpine_3.23`)
- **PHP images:** `{php}-{version}-{distro}_{variant}` (e.g. `8.4-8.0.2-alpine_3.23`)

Omit `base_image_version` from a matrix entry to pull the latest upstream (default behavior).

```yaml
# Pin to an exact SHA digest (most reproducible, overrides tag entirely)
with:
  base_image: "ghcr.io/nfrastack/container-base"
  base_image_digest: "sha256:a1b2c3d4e5f6..."
  # Pulls: ghcr.io/nfrastack/container-base@sha256:a1b2c3d4e5f6...
```

## Architecture Support

All workflows build natively on matching runners:
- **amd64**: `ubuntu-latest`
- **arm64**: `ubuntu-24.04-arm`

After both builds complete, a manifest job merges them into a multi-arch image. Users don't need to specify architecture when pulling.

To build amd64-only (things that won't compile on ARM):

```yaml
- { php_version: "7.3", distro: "alpine", distro_variant: "3.12", arch: "linux/amd64" }
```

## Build Flow

Each workflow call executes 5 jobs:

```
prepare  -->  build-amd64  -->  manifest  -->  summary
             build-arm64  --|
```

1. **prepare**: Checkout, download encrypted artifacts, compute tags, label Containerfile, upload workspace
2. **build-amd64**: Native amd64 build, push staging image
3. **build-arm64**: Native arm64 build, push staging image (skipped if not in platforms)
4. **manifest**: Merge per-arch images into multi-arch manifests for all final tags
5. **summary**: Write pass/fail table to GitHub Actions summary, emit error annotations

## Error Handling

- **`fail-fast: false`**: All matrix entries run to completion. A failing alpine 3.15 build won't stop alpine 3.23.
- **Build summaries**: Each build writes a markdown summary table to the GitHub Actions UI showing per-architecture pass/fail status.
- **Error annotations**: Failed builds produce `::error::` annotations visible in the PR/commit checks.
- **Manifest resilience**: If only one arch succeeds (e.g. arm64 fails), the manifest is still created with the successful arch. The summary reports which arch failed.

## Containerfile Requirements

All Containerfiles must accept `BASE_IMAGE` as a build arg:

```dockerfile
ARG BASE_IMAGE
FROM ${BASE_IMAGE}
```

The workflow computes the full `BASE_IMAGE` reference (including registry, tag, and optional version) and passes it. No tag construction logic should live in the Containerfile.

Additional build-args vary by workflow:
- **container-build_base.yml**: `IMAGE_BASE_MODULES`, `IMAGE_MODULES`
- **container-build_phpfpm.yml**: `PHP_BASE`
- **container-build_generic.yml**: Whatever you pass via `build_args` input

## Dependency Chain

```
docker.io/alpine:3.23
  |
  v
container-base (container-build_base.yml)
  |
  +--> container-redis, container-postgres, container-mariadb, ... (container-build_generic.yml)
  |
  +--> container-nginx (container-build_multi.yml)
         |
         +--> container-nginx-php-fpm (container-build_phpfpm.yml)
                |
                +--> container-wordpress, container-nextcloud, container-moodle, ... (container-build_generic.yml)
```

## Versioning Schemes

| Repo                    | Format        | Example    | Description                    |
| ----------------------- | ------------- | ---------- | ------------------------------ |
| container-base          | `YYYY.M.N`    | `2026.4.0` | Calendar-based                 |
| container-nginx         | `X.Y.Z`       | `8.0.6`    | Semver                         |
| container-nginx-php-fpm | `X.Y.Z`       | `8.0.2`    | Semver                         |
| container-redis         | `MAJOR-X.Y.Z` | `8-3.0.4`  | Upstream Major + image version |
| All others              | `X.Y.Z`       | `5.9.1`    | Semver                         |

Git tags trigger versioned Docker tags. Push `v8.0.6` in container-nginx and the workflow produces `8.0.6`, `8.0.6-alpine_3.23_faflopza`, etc.

## Trigger Workflow Template

Most repos need 3 trigger workflows that call the image build workflow. These are identical across repos:

```yaml
# .github/workflows/build_push.yml
name: "PUSH - Build on repository Push"
on:
  push:
    paths:
      - '**'
      - '!CHANGELOG.md'
      - '!/examples/*'
      - '!LICENSE'
      - '!README.md'
jobs:
  prepare:
    uses: nfrastack/gha/.github/workflows/artifacts-encrypt.yml@main
    secrets: inherit
  build:
    needs: prepare
    uses: ./.github/workflows/image_build.yml
    secrets: inherit
  cleanup:
    needs: [build]
    uses: nfrastack/gha/.github/workflows/artifacts-remove.yml@main
    secrets: inherit
```

```yaml
# .github/workflows/build_manual.yml
name: "MANUAL - Build all images"
on:
  workflow_dispatch:
    inputs:
      Manual_Build:
        description: 'Manual Build'
        required: false
jobs:
  prepare:
    uses: nfrastack/gha/.github/workflows/artifacts-encrypt.yml@main
    secrets: inherit
  build:
    needs: prepare
    uses: ./.github/workflows/image_build.yml
    secrets: inherit
  cleanup:
    needs: [build]
    uses: nfrastack/gha/.github/workflows/artifacts-remove.yml@main
    secrets: inherit
```

The `image_build.yml` file is the only file that varies per repo -- it defines the matrix and calls the appropriate reusable workflow.
