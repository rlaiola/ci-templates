# CI Templates

Reusable GitHub Actions workflow templates for building, testing, and maintaining Docker images and code quality.

## Table of Contents

- [Overview](#overview)
- [Available Workflows](#available-workflows)
- [Usage in Other Repositories](#usage-in-other-repositories)
- [Configuration Files](#configuration-files)
- [Contributing](#contributing)
- [Code of Conduct](#code-of-conduct)
- [Security](#security)
- [License](#license)

## Overview

This repository contains a collection of reusable GitHub Actions workflows that can be used across multiple repositories to:

- Build and publish multi-platform Docker images to GitHub Container Registry (ghcr.io)
- Scan Docker images for security vulnerabilities
- Lint Docker images for best practices and CIS benchmarks
- Lint code bases for quality and standards
- Clean up unused Docker images and build caches
- Manage stale issues and pull requests

## Available Workflows

### [build-images.yml](.github/workflows/build-images.yml)

Builds and publishes multi-platform Docker images to ghcr.io with support for multiple base images, platforms, and custom tags.

**Features:**

- Multi-platform builds (linux/amd64, linux/arm64, etc.)
- Support for multiple base images (Ubuntu releases)
- GitHub Actions cache optimization
- Automatic tagging strategies (semver, branch, schedule)
- Push-by-digest for efficient multi-arch builds

```mermaid
sequenceDiagram
    autonumber
    participant W as Workflow Call
    participant B as build job<br>(matrix: image × parent × platform)
    participant M as merge job<br>(matrix: image × parent)
    participant GHCR as ghcr.io
    participant Artifacts as GitHub Artifacts

    W->>B: Trigger workflow_call<br>Inputs: images, parents, platforms, tags, latest, ref, ...

    Note over B: permissions: contents:read, packages:write

    B->>B: Checkout code (optional ref / external repo)
    B->>B: Setup QEMU + Buildx + Login to GHCR

    B->>GHCR: Build & push single-arch image by digest<br>(with layer caching from current + base branch)

    B->>Artifacts: Upload digest file<br>(one artifact per image+release+arch)

    Note over B,M: All platform-specific builds complete<br>(fail-fast: false → full matrix runs)

    M->>Artifacts: Download all digests for this image+release
    M->>M: Generate final tag list using docker/metadata-action<br>• semver, raw, nightly, latest (if enabled + default parent)

    M->>GHCR: Login to GHCR
    M->>GHCR: Create multi-arch manifest list<br>docker buildx imagetools create -t <all-tags> <digests...>

    M->>GHCR: Push final tagged multi-arch image<br>(including :latest when appropriate)

    M->>M: Inspect final image (optional)

    Note over B,GHCR: All images built per-platform<br>→ merged into clean, tagged multi-arch manifests

    M-->>W: Build & publish complete
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

### [scan-images.yml](.github/workflows/scan-images.yml)

Performs security vulnerability scanning on Docker images using Trivy.

**Features:**

- Scans for CRITICAL and HIGH severity vulnerabilities
- Generates SARIF reports
- Multi-platform image support

```mermaid
sequenceDiagram
    autonumber
    participant W as Workflow Call
    participant S as scan job<br>(matrix: image × parent × platform)
    participant GHCR as ghcr.io
    participant Trivy as Trivy Scanner<br>(aquasecurity/trivy-action)

    W->>S: Trigger workflow_call<br>Inputs: images, parents, platforms, tag

    Note over S: permissions: packages:read, security-events:write

    S->>S: Compute per-matrix values<br>• release (from parent)<br>• full image name: ghcr.io/owner/image:tag-release<br>• architecture (from platform)

    S->>S: Setup Docker Buildx

    S->>GHCR: Login to ghcr.io<br>(using GITHUB_TOKEN)

    S->>GHCR: Pull specific platform image<br>docker pull --platform linux/amd64 (etc.)

    S->>Trivy: Scan image with Trivy<br>• Format: SARIF<br>• Ignore unfixed vulnerabilities<br>• Only CRITICAL + HIGH<br>• OS + library packages only

    Trivy-->>S: Generate trivy-<image>-<release>-<arch>-image-results.sarif

    %% Optional upload step (currently commented out in workflow)
    Note right of S: (Optional: Upload SARIF to GitHub Security tab<br>via github/codeql-action/upload-sarif)

    alt No critical/high issues found
        S-->>W: Job passes (all matrix combos)
    else
        S-->>W: Job fails on first vulnerable image<br>(fail-fast: false → all platforms scanned anyway)
    end

    Note over S,Trivy: All built multi-platform images<br>scanned for critical vulnerabilities

    S-->>W: Security scan complete
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

### [lint-images.yml](.github/workflows/lint-images.yml)

Lints Docker images for best practices and CIS benchmarks using Dockle.

**Features:**

- Best practices validation
- CIS benchmark compliance checks
- Multi-platform image support

```mermaid
sequenceDiagram
    autonumber
    participant W as Workflow Call
    participant L as lint job<br>(matrix: image × parent × platform)
    participant GHCR as ghcr.io
    participant Dockle as Dockle Linter<br>(erzz/dockle-action)

    W->>L: Trigger workflow_call<br>Inputs: images, parents, platforms, tag

    Note over L: permissions: packages:read, security-events:write

    L->>L: Compute per-matrix values<br>• release (from parent)<br>• full image: ghcr.io/owner/image:tag-release<br>• architecture (from platform)

    L->>L: Checkout repository<br>(to load .dockleignore if present)

    L->>L: Setup Docker Buildx

    L->>GHCR: Login to ghcr.io<br>(using GITHUB_TOKEN)

    L->>GHCR: Pull specific platform image<br>docker pull --platform linux/arm64 (etc.)

    L->>Dockle: Run Dockle on pulled image<br>• Checks CIS benchmarks + best practices<br>• Output: SARIF report<br>• Failure threshold: fatal only<br>• Respects .dockleignore

    Dockle-->>L: Generate dockle-<image>-<release>-<arch>-image-results.sarif

    %% Upload step is currently commented out
    Note right of L: (Optional: Upload SARIF to GitHub Security tab<br>via codeql-action/upload-sarif)

    alt No fatal issues found
        L-->>W: Job passes (all matrix variants)
    else One or more fatal findings
        L-->>W: Job fails<br>(fail-fast: false → all platforms still checked)
    end

    Note over L,Dockle: All published multi-platform images<br>validated against Docker/CIS best practices

    L-->>W: Image linting complete
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

### [lint-files.yml](.github/workflows/lint-files.yml)

Lints the entire codebase using Super-Linter.

**Features:**

- Multiple language support
- Dockerfile, YAML, JSON, Markdown validation
- Configurable via `.github/super-linter.env`

```mermaid
sequenceDiagram
    autonumber
    participant T as Trigger<br>(push / PR / dispatch / workflow_call)
    participant L as lint job<br>(ubuntu-latest)
    participant Repo as GitHub Repository
    participant SL as Super-Linter<br>(slim image)

    T->>L: Trigger workflow<br>(push to non-main, PR to main, manual, or call)
    Note right of T: Optional input: ref (branch/tag/SHA)

    Note over L: permissions: contents:read, statuses:write

    alt inputs.ref is empty or not provided
        L->>Repo: Checkout current commit + full history<br>(fetch-depth: 0)
    else
        L->>Repo: Checkout specified ref + full history<br>(ref: ${{ inputs.ref }})
    end

    L->>L: Load Super-Linter environment variables<br>(from .github/super-linter.env → $GITHUB_ENV)

    L->>SL: Run super-linter/super-linter/slim@v8
    SL->>Repo: Analyze changed + all relevant files<br>(using full git history)
    SL-->>L: Validation results (pass/fail per linter)

    alt All linters pass
        L-->>T: Job succeeds → status: success
    else
        L-->>T: Job fails → status: failure + annotated errors
    end

    Note over L,SL: Code base has been fully linted<br>with multiple languages/formatters

    L-->>T: Workflow completes
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

### [clean-packages.yml](.github/workflows/clean-packages.yml)

Removes untagged and unsupported Docker image versions from ghcr.io.

**Features:**

- Deletes untagged images
- Removes deprecated version tags
- Preserves supported releases

```mermaid
sequenceDiagram
    autonumber
    participant W as Workflow Call
    participant S as setup job<br>(read-matrix.yml)
    participant C as cleanup job
    participant GHCR as ghcr.io

    W->>S: Get supported releases (from matrix.json)
    S-->>C: outputs.release → list of supported versions

    W->>C: Trigger cleanup (optional: extra tags to keep)

    Note over C: permissions: packages:write

    C->>C: Build final "keep these tags" list<br>(supported releases + input tags)

    %% Phase 1 – Untagged images
    C->>GHCR: crane manifest → collect digests of all kept tags
    C->>GHCR: Delete ONLY untagged versions<br>but ignore digests belonging to kept tags
    GHCR-->>C: Untagged garbage removed

    %% Phase 2 – Fully unsupported (deprecated) tags
    C->>GHCR: crane ls → get all existing tags
    C->>C: Find deprecated tags = all tags ∉ keep-list

    alt There are deprecated tags
        C->>GHCR: Query GitHub API → get version IDs for deprecated tags
        C->>C: Skip any version that also has a supported tag
        C->>GHCR: Delete remaining unsupported package versions by ID
        GHCR-->>C: Deprecated images removed
    else
        Note right of C: Nothing to delete
    end

    Note over W,GHCR: Repository is now clean:<br>• No untagged images<br>• No unsupported/old tags

    C-->>W: Done
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

### [clean-cache.yml](.github/workflows/clean-cache.yml)

Deletes all GitHub Actions cache entries for a repository.

**Features:**

- Batch deletion of cache entries
- Useful for troubleshooting build issues

```mermaid
sequenceDiagram
    autonumber
    participant W as Workflow Call
    participant C as cleanup job<br>(ubuntu-latest)
    participant GHA as GitHub Actions Cache<br>(via gh-actions-cache extension)

    W->>C: Trigger workflow_call<br>(optional: repository + GH_TOKEN)
    Note over C: permissions: actions:write, contents:read

    C->>C: Checkout repository

    C->>GHA: Install gh extension: actions/gh-actions-cache

    loop While caches exist
        C->>GHA: List up to 100 largest cache entries<br>(gh actions-cache list --limit 100 --sort size)
        GHA-->>C: Returns cache keys

        alt Caches found
            C->>GHA: Delete each cache key<br>(gh actions-cache delete <key> --confirm)
            Note right of C: set +e → individual deletions won't fail job
        else
            C->>C: No more caches → done
        end
    end

    Note over C,GHA: All GitHub Actions cache entries<br>for the repository have been deleted

    C-->>W: Workflow completes successfully
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

### [close-stale.yml](.github/workflows/close-stale.yml)

Automatically marks and closes stale issues and pull requests.

**Features:**

- Configurable staleness periods
- Customizable warning and closing messages
- Separate handling for issues and PRs

```mermaid
sequenceDiagram
    autonumber
    participant T as Trigger<br>(schedule / dispatch / workflow_call)
    participant S as stale job<br>(ubuntu-latest)
    participant GH as GitHub Issues & PRs

    T->>S: Run daily at 18:30 UTC<br>or manually / via workflow_call
    Note right of T: Configurable via inputs:<br>• stale labels<br>• days before stale/close<br>• custom messages

    Note over S: permissions: issues:write, pull-requests:write

    S->>GH: actions/stale@v10 analyzes repository

    loop For every open issue
        alt No activity for ≥ ${{ inputs.days_before_stale }} days
            S->>GH: Add label: ${{ inputs.stale_issue_label }}<br>Comment: "This issue is stale..."
        end
        alt Has stale label AND no activity for ≥ ${{ inputs.days_before_close }} days
            S->>GH: Close issue + comment: "Closed due to inactivity"
        end
    end

    loop For every open pull request
        alt No activity for ≥ ${{ inputs.days_before_pr_stale }} days
            S->>GH: Add label: ${{ inputs.stale_pr_label }}<br>Comment: "This PR is stale..."
        end
        alt Has stale label AND no activity<br>AND days_before_pr_close ≠ -1
            S->>GH: Close PR + comment: "Closed due to inactivity"
        else if days_before_pr_close == -1
            Note right of S: PRs are only marked stale<br>→ never auto-closed
        end
    end

    Note over S,GH: Stale issues/PRs automatically warned<br>and optionally closed<br>Keeps repository clean and active

    S-->>T: Workflow completes
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

### [read-matrix.yml](.github/workflows/read-matrix.yml)

Reads build matrix configuration from a JSON file.

**Features:**

- Centralizes build configuration
- Outputs matrix values for use in other workflows

```mermaid
sequenceDiagram
    autonumber
    participant W as Workflow Call
    participant M as get-matrix job<br>(ubuntu-latest)
    participant Repo as Repository
    participant File as matrix.json<br>(via inputs.matrix-path)

    W->>M: Call reusable workflow<br>Input: matrix-path (required)

    M->>Repo: Checkout repository<br>(actions/checkout@v5)

    M->>File: Read JSON file at ${{ inputs.matrix-path }}

    M->>M: Parse matrix with jq<br>→ .release, .parent, .platform, .repository, .ref

    M->>M: Determine defaults<br>• default_release = first item in release list<br>• default_parent   = first item in parent list

    M->>M: Set job outputs<br>• release (array)<br>• default_release (string)<br>• parent (array)<br>• default_parent (string)<br>• platform (array)<br>• repository, ref

    Note over M: Outputs are automatically mapped<br>to workflow-level outputs<br>(release, default_release, parent, ...)

    M-->>W: Matrix data ready for downstream jobs<br>(e.g. multi-platform Docker builds)

    Note over W,M: Reusable source of truth for build matrix<br>Centralized in a single JSON file
```

> **Workflow overview** – _Mermaid diagram generated with Grok (xAI) · Simplified for clarity, reflects current implementation_

## Usage in Other Repositories

### Prerequisites

1. Enable GitHub Actions in your repository
2. For Docker image workflows, ensure GitHub Container Registry is enabled
3. Create a Personal Access Token (PAT) with `packages:write` and `repo` permissions for cleanup workflows

### Basic Example

Create a workflow file in your repository at `.github/workflows/build.yml`:

```yaml
name: Build Docker Images

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Read build matrix configuration
  setup:
    uses: rlaiola/ci-templates/.github/workflows/read-matrix.yml@main
    with:
      matrix-path: matrix.json

  # Build and publish images
  build:
    needs: setup
    uses: rlaiola/ci-templates/.github/workflows/build-images.yml@main
    with:
      images: ${{ needs.setup.outputs.release }}
      parents: ${{ needs.setup.outputs.parent }}
      default_parent: ${{ needs.setup.outputs.default_parent }}
      platforms: ${{ needs.setup.outputs.platform }}
      tags: 'latest'
      ref: ${{ github.ref }}

  # Lint images
  lint:
    needs: [setup, build]
    uses: rlaiola/ci-templates/.github/workflows/lint-images.yml@main
    with:
      images: ${{ needs.setup.outputs.release }}
      parents: ${{ needs.setup.outputs.parent }}
      platforms: ${{ needs.setup.outputs.platform }}
      tag: 'latest'

  # Scan images
  scan:
    needs: [setup, build]
    uses: rlaiola/ci-templates/.github/workflows/scan-images.yml@main
    with:
      images: ${{ needs.setup.outputs.release }}
      parents: ${{ needs.setup.outputs.parent }}
      platforms: ${{ needs.setup.outputs.platform }}
      tag: 'latest'
```

### Matrix Configuration File

Create a `matrix.json` file in your repository root:

```json
{
  "release": ["1.0.0", "1.1.0"],
  "parent": ["ubuntu:22.04", "ubuntu:24.04"],
  "platform": ["linux/amd64", "linux/arm64"],
  "repository": "https://github.com/yourusername/yourrepo.git",
  "ref": "main"
}
```

### Cleanup Workflows

For cleanup workflows, add a secret named `GH_TOKEN` to your repository with a PAT that has the required permissions:

```yaml
name: Cleanup

on:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 0' # Weekly on Sunday at 2 AM

jobs:
  cleanup-packages:
    uses: rlaiola/ci-templates/.github/workflows/clean-packages.yml@main
    with:
      tags: 'dev test' # Additional tags to preserve
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}

  cleanup-cache:
    uses: rlaiola/ci-templates/.github/workflows/clean-cache.yml@main
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
```

### Code Linting

```yaml
name: Lint Code

on:
  push:
    branches-ignore: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    uses: rlaiola/ci-templates/.github/workflows/lint-files.yml@main
```

### Stale Issue Management

```yaml
name: Manage Stale Issues

on:
  schedule:
    - cron: '30 18 * * *'

jobs:
  stale:
    uses: rlaiola/ci-templates/.github/workflows/close-stale.yml@main
    with:
      stale_issue_label: 'stale'
      days_before_stale: 90
      days_before_close: 7
```

## Configuration Files

- `.github/super-linter.env` - Super-Linter configuration
- `.yamllint` - YAML linting rules
- `markdownlint-cli2.jsonc` - Markdown linting configuration
- `.prettierrc` - Code formatting rules

## Contributing

If you would like to help contribute to this project, please see [CONTRIBUTING.md](https://github.com/rlaiola/ci-templates/blob/main/CONTRIBUTING.md).

Before submitting a PR consider building and testing a Docker image locally and checking your code with Super-Linter:

```sh
docker run --rm \
           -e ACTIONS_RUNNER_DEBUG=true \
           -e RUN_LOCAL=true \
           -e DEFAULT_BRANCH=main \
           --env-file ".github/super-linter.env" \
           -v "$PWD":/tmp/lint \
           ghcr.io/super-linter/super-linter:latest
```

## Code of Conduct

See [CODE_OF_CONDUCT.md](https://github.com/rlaiola/ci-templates/blob/main/CODE_OF_CONDUCT.md) for community standards.

## Security

See [SECURITY.md](https://github.com/rlaiola/ci-templates/blob/main/SECURITY.md) for security policy and vulnerability reporting.

## License

Copyright Universidade Federal do Espirito Santo (Ufes)

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program. If not, see <https://www.gnu.org/licenses/>.

This program is released under license GNU GPL v3+ license.
