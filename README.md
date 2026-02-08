# CI Components

Reusable GitHub Actions workflows for personal projects.

## Available Workflows

| Workflow | File | Description |
|----------|------|-------------|
| **Semantic Release (Python/UV)** | `semantic-release-uv.yml` | Versioning, changelog, and optional PyPI publishing for Python/uv projects |
| **Semantic Release (Bun)** | `semantic-release-bun.yml` | Versioning, changelog, and GitHub releases for Bun/TypeScript projects |
| **NPM Publish (Bun)** | `npm-publish-bun.yml` | Publish to npm with OIDC provenance (no token needed) |
| **MCP Registry Publish** | `mcp-registry-publish.yml` | Publish MCP servers to the [official MCP Registry](https://registry.modelcontextprotocol.io/) |
| **Auto Merge Dependabot** | `auto-merge-dependabot.yml` | Auto-merge Dependabot PRs for minor/patch updates |

All workflows are called with `detailobsessed/ci-components/.github/workflows/<file>@main`.

## Table of Contents

- [Quick Start — Python/UV Project](#quick-start--pythonuv-project)
- [Quick Start — Bun/TypeScript Project](#quick-start--buntypescript-project)
- [Workflow Reference](#workflow-reference)
  - [Semantic Release (Python/UV)](#semantic-release-pythonuv)
  - [Semantic Release (Bun)](#semantic-release-bun)
  - [NPM Publish (Bun)](#npm-publish-bun)
  - [MCP Registry Publish](#mcp-registry-publish)
  - [Auto Merge Dependabot](#auto-merge-dependabot)
- [Gotchas](#gotchas)

---

## Quick Start — Python/UV Project

End-to-end setup for a Python project with automated releases, PyPI publishing, and optional MCP Registry publishing.

### 1. Add python-semantic-release

```bash
# Add to the "maintain" dependency group (keeps it out of production deps)
uv add --group maintain python-semantic-release
```

### 2. Configure semantic-release in `pyproject.toml`

```toml
[tool.semantic_release]
# Where to read/write the version string
version_toml = ["pyproject.toml:project.version"]
# Only create releases from main
branch = "main"
# Build command runs during release — lock file gets updated, then package is built
build_command = """
uv lock --upgrade-package "$PACKAGE_NAME"
uv build
"""
# Allow 0.x.y versions (don't force 1.0.0)
allow_zero_version = true
# Don't bump to 1.0.0 on breaking changes while in 0.x
major_on_zero = false

# If publishing to MCP Registry, keep server.json version in sync:
# version_variables = ["server.json:version"]

[tool.semantic_release.changelog]
changelog_file = "CHANGELOG.md"
# "update" appends to existing changelog instead of replacing it
mode = "update"

[tool.semantic_release.commit_parser_options]
# All conventional commit types that are recognized
allowed_tags = ["build", "chore", "ci", "deps", "docs", "feat", "fix", "perf", "refactor", "style", "test"]
# These trigger a minor version bump (0.x.0)
minor_tags = ["feat"]
# These trigger a patch version bump (0.0.x)
patch_tags = ["fix", "perf"]
```

### 3. Create `release.yml`

```yaml
# .github/workflows/release.yml
name: release

on:
  # Run after CI passes on main — ensures broken code never gets released
  workflow_run:
    workflows: [ci]
    types: [completed]
    branches: [main]
  # Allow manual trigger for re-runs
  workflow_dispatch:

permissions:
  # semantic-release needs to push tags and create GitHub releases
  contents: write
  # PyPI OIDC trusted publishing needs this
  id-token: write

jobs:
  release:
    # Only run if CI passed (or manual trigger)
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/semantic-release-uv.yml@main
    with:
      # MUST be "false" when using trusted publishing — see pypi-publish job below
      pypi-publish: "false"
    # Pass GITHUB_TOKEN to the reusable workflow
    secrets: inherit

  # PyPI publish is a SEPARATE job because OIDC trusted publishing validates
  # the workflow file path. Reusable workflows point to ci-components' repo,
  # not yours — so PyPI rejects the token. This job runs in YOUR repo's context.
  pypi-publish:
    needs: release
    # Only publish if semantic-release created a new version AND publishing is enabled
    if: needs.release.outputs.released == 'true' && vars.PYPI_PUBLISH == 'true'
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/your-package  # TODO: replace with your package name
    permissions:
      contents: read      # Needed to checkout the code
      id-token: write      # Needed for PyPI OIDC authentication
    steps:
      - uses: actions/checkout@v6
        with:
          ref: main          # Start from main (which has the new tag)
          fetch-depth: 0     # Full history needed for git describe
          fetch-tags: true   # Fetch all tags so we can find the latest release

      # Checkout the exact commit that was tagged by semantic-release
      - name: Checkout latest release tag
        run: git checkout "$(git describe --tags --abbrev=0)"

      - name: Setup uv
        uses: astral-sh/setup-uv@v7

      # Build the sdist and wheel
      - run: uv build
      # Publish to PyPI using OIDC (no token needed — trusted publishing handles auth)
      - run: uv publish

  # Optional: MCP Registry publish (only for MCP servers)
  # Uncomment if your project is an MCP server registered on the MCP Registry.
  # mcp-registry-publish:
  #   needs: release
  #   if: needs.release.outputs.released == 'true'
  #   uses: detailobsessed/ci-components/.github/workflows/mcp-registry-publish.yml@main
```

> **⚠️ Why is `pypi-publish` a separate job?** PyPI OIDC trusted publishing validates the `job_workflow_ref` claim, which points to the *reusable* workflow's repo — not yours. Publishing from the reusable workflow will always fail with "no corresponding publisher". The `pypi-publish` job must live in *your* `release.yml`.

### 4. First publish — TestPyPI

Always validate on TestPyPI before touching production PyPI.

**a) Create a TestPyPI account and token:**

1. Register at <https://test.pypi.org/account/register/> (separate from PyPI)
2. Create an API token at <https://test.pypi.org/manage/account/token/>
3. Store the token securely (e.g. 1Password, not in shell history)

**b) Build and publish to TestPyPI:**

```bash
# Build the package (creates dist/ with .tar.gz and .whl)
uv build

# Publish to TestPyPI — uses 1Password CLI to avoid tokens in shell history
uv publish --publish-url https://test.pypi.org/legacy/ \
  --token "$(op read op://Personal/testpypi-token/credential)"
```

**c) Verify the package installs correctly:**

```bash
# Install from TestPyPI to verify it works
uv pip install --index-url https://test.pypi.org/simple/ your-package
```

**d) Optional: Configure TestPyPI trusted publishing too:**

Go to <https://test.pypi.org/manage/project/your-package/settings/publishing/> and add the same trusted publisher config as production (see step 6). This lets you test the full OIDC flow before going live.

To use TestPyPI from the reusable workflow, set `pypi-publish: "test"` instead of `"false"`.

### 5. First publish — production PyPI

Once TestPyPI looks good:

```bash
# Build (if not already built)
uv build

# Publish to production PyPI
uv publish --token "$(op read op://Personal/pypi-token/credential)"
```

### 6. Configure PyPI trusted publishing (one-time)

Go to <https://pypi.org/manage/project/your-package/settings/publishing/> and add:

| Field | Value |
|-------|-------|
| **Owner** | your GitHub user or org (e.g. `detailobsessed`) |
| **Repository** | your repo name (e.g. `codereviewbuddy`) |
| **Workflow** | `release.yml` (**must be the calling workflow**, not the reusable one) |
| **Environment** | `pypi` |

### 7. Enable automated publishing

Set the `PYPI_PUBLISH` repo variable to `true` in GitHub repo settings → Variables → Actions.

From now on, every semantic-release version bump will automatically publish to PyPI.

### 8. Optional: MCP Registry

If your project is an MCP server, see [MCP Registry Publish](#mcp-registry-publish) below.

---

## Quick Start — Bun/TypeScript Project

End-to-end setup for a Bun project with automated releases, npm publishing, and optional MCP Registry publishing.

### 1. Add semantic-release

```bash
# semantic-release core + plugins for changelog and git commit
bun add -d semantic-release @semantic-release/changelog @semantic-release/git
```

### 2. Create `.releaserc.json`

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    ["@semantic-release/npm", { "npmPublish": false }],
    ["@semantic-release/git", {
      "assets": ["package.json", "CHANGELOG.md"],
      "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
    }],
    "@semantic-release/github"
  ]
}
```

Plugin order matters: `commit-analyzer` → `release-notes-generator` → `changelog` → `npm` → `git` → `github`. If publishing to npm, remove `{ "npmPublish": false }` from the npm plugin.

### 3. Create `release.yml`

```yaml
# .github/workflows/release.yml
name: Release

on:
  # Run after CI passes on main — ensures broken code never gets released
  workflow_run:
    workflows: ["CI"]
    branches: [main]
    types: [completed]
  # Allow manual trigger for re-runs
  workflow_dispatch:

permissions:
  contents: write       # Push tags and create releases
  issues: write         # Comment on issues referenced in commits
  pull-requests: write  # Comment on PRs referenced in commits

jobs:
  release:
    # Only run if CI passed (or manual trigger)
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/semantic-release-bun.yml@main
    # Pass GITHUB_TOKEN to the reusable workflow
    secrets: inherit
```

> **⚠️ Important:** Do NOT use `on: release: types: [published]` as a trigger for downstream workflows. Releases created with `GITHUB_TOKEN` (which semantic-release uses) do NOT emit `release` events. Use `workflow_run` instead.

### 4. Optional: npm publishing

Create a separate workflow triggered after release:

```yaml
# .github/workflows/npm-publish.yml
name: NPM Publish

on:
  # Runs after the Release workflow completes on main
  workflow_run:
    workflows: ["Release"]
    branches: [main]
    types: [completed]
  # Allow manual trigger for re-runs
  workflow_dispatch:

permissions:
  id-token: write   # OIDC token for npm trusted publishing
  contents: read    # Checkout the code

jobs:
  publish:
    # Only run if Release succeeded (or manual trigger)
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/npm-publish-bun.yml@main
```

**Prerequisites:**

- Configure [npm trusted publishing](https://docs.npmjs.com/trusted-publishers/) at npmjs.com (workflow filename must be the **calling** workflow, not the reusable one)
- Set `"publishConfig": { "access": "public" }` in `package.json` for scoped packages
- **Node.js 24+** is required for npm OIDC — the workflow defaults to 24
- **Must use `ubuntu-latest`** — Blacksmith/self-hosted runners fail provenance verification

### 5. Optional: MCP Registry

If your project is an MCP server, see [MCP Registry Publish](#mcp-registry-publish) below.

---

## Workflow Reference

### Semantic Release (Python/UV)

`detailobsessed/ci-components/.github/workflows/semantic-release-uv.yml@main`

| Input | Default | Description |
|-------|---------|-------------|
| `python-version` | `3.13` | Python version |
| `runner` | `blacksmith-4vcpu-ubuntu-2404` | GitHub Actions runner |
| `pypi-publish` | `false` | PyPI publish mode: `true`, `test`, or `false` (use `false` with trusted publishing) |

| Output | Description |
|--------|-------------|
| `released` | `true` if a new release was created, `false` otherwise |

**Required in calling repo:**

- `pyproject.toml` with `[tool.semantic_release]` configuration
- `python-semantic-release` in dev dependencies (`uv add --group maintain python-semantic-release`)

---

### Semantic Release (Bun)

`detailobsessed/ci-components/.github/workflows/semantic-release-bun.yml@main`

| Input | Default | Description |
|-------|---------|-------------|
| `node-version` | `22` | Node.js version (semantic-release v25 requires ^22.14.0) |
| `bun-version` | `latest` | Bun version |
| `build-command` | `bun run build` | Build command to run before release |

**Required in calling repo:**

- `.releaserc.json` with semantic-release configuration
- `package.json` with semantic-release dev dependencies

---

### NPM Publish (Bun)

`detailobsessed/ci-components/.github/workflows/npm-publish-bun.yml@main`

| Input | Default | Description |
|-------|---------|-------------|
| `node-version` | `24` | Node.js version (24+ required for OIDC) |
| `bun-version` | `latest` | Bun version |
| `build-command` | `bun run build` | Build command to run before publish |

Provenance attestations are automatically generated when publishing via trusted publishing from public repos.

---

### MCP Registry Publish

`detailobsessed/ci-components/.github/workflows/mcp-registry-publish.yml@main`

Publishes MCP servers to the [official MCP Registry](https://registry.modelcontextprotocol.io/) using GitHub OIDC authentication.

| Input | Default | Description |
|-------|---------|-------------|
| `runner` | `blacksmith-4vcpu-ubuntu-2404` | Runner to use |
| `default-branch` | `main` | Branch to fetch latest release tag from |

#### Setup

**1. Install the CLI locally:**

```bash
brew install mcp-publisher
```

Or without Homebrew:

```bash
curl -L "https://github.com/modelcontextprotocol/registry/releases/latest/download/mcp-publisher_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/').tar.gz" | tar xz mcp-publisher
sudo mv mcp-publisher /usr/local/bin/
```

**2. Create `server.json`:**

```bash
mcp-publisher init
```

Or create manually — example for a PyPI package:

```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json",
  "name": "io.github.{org}/{server-name}",
  "description": "Your server description",
  "version": "0.1.0",
  "repository": {
    "url": "https://github.com/{org}/{repo}",
    "source": "github"
  },
  "packages": [
    {
      "registryType": "pypi",
      "identifier": "your-package-name",
      "version": "0.1.0",
      "runtimeHint": "uvx",
      "transport": { "type": "stdio" },
      "environmentVariables": []
    }
  ]
}
```

Key fields: `name` uses reverse-DNS format (`io.github.{org}/{server-name}`). `registryType` is `"pypi"` or `"npm"`. `runtimeHint` is how clients run it (`uvx`, `npx`, `docker`, etc.). Both `version` fields are required (despite the schema saying optional) and must match — semantic-release updates both.

**3. Add the README ownership marker:**

For PyPI packages, add this HTML comment anywhere in your `README.md`:

```html
<!-- mcp-name: io.github.{org}/{server-name} -->
```

For npm packages, add to `package.json`:

```json
"mcpName": "io.github.{org}/{server-name}"
```

**4. Keep `server.json` version in sync with releases:**

For Python projects, add to `[tool.semantic_release]` in `pyproject.toml`:

```toml
version_variables = ["server.json:version"]
```

For npm projects, add to `.releaserc.json` plugins:

```json
["@semantic-release/exec", {
  "prepareCmd": "sed -i'' -e 's/\"version\": \"[^\"]*\"/\"version\": \"${nextRelease.version}\"/g' server.json"
}]
```

**5. First publish (manual, one-time):**

```bash
mcp-publisher login github
mcp-publisher publish
```

> For org namespaces (`io.github.{org}/...`), your org membership must be **public** at `https://github.com/orgs/{org}/people`.

**6. Add to `release.yml`:**

For Python/UV projects, add as a job alongside `pypi-publish`:

```yaml
  # Gate on released == 'true' to avoid "version already exists" errors
  # when semantic-release runs but decides there's nothing to release
  mcp-registry-publish:
    needs: release
    if: needs.release.outputs.released == 'true'
    uses: detailobsessed/ci-components/.github/workflows/mcp-registry-publish.yml@main
```

For npm projects, create a standalone workflow:

```yaml
# .github/workflows/mcp-registry-publish.yml
name: MCP Registry Publish

on:
  # Runs after npm publish completes
  workflow_run:
    workflows: ["NPM Publish"]
    types: [completed]
  workflow_dispatch:

permissions:
  id-token: write   # OIDC token for MCP Registry authentication
  contents: read    # Checkout the code to find server.json

jobs:
  publish:
    # Only run if NPM Publish succeeded (or manual trigger)
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/mcp-registry-publish.yml@main
```

---

### Auto Merge Dependabot

`detailobsessed/ci-components/.github/workflows/auto-merge-dependabot.yml@main`

Automatically merge Dependabot PRs for minor and patch updates.

```yaml
# .github/workflows/auto-merge.yml
name: Auto Merge Dependabot PRs

on:
  # Triggered on every PR event from Dependabot
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: write       # Merge the PR
  pull-requests: write  # Enable auto-merge on the PR

jobs:
  auto-merge:
    uses: detailobsessed/ci-components/.github/workflows/auto-merge-dependabot.yml@main
    # Pass GITHUB_TOKEN for merge permissions
    secrets: inherit
```

| Input | Default | Description |
|-------|---------|-------------|
| `merge-method` | `merge` | Merge method (merge, squash, rebase) |
| `update-types` | `minor,patch` | Update types to auto-merge (comma-separated) |

---

## Gotchas

- **`GITHUB_TOKEN` releases don't emit `release` events.** Always use `workflow_run` to chain workflows, never `on: release: types: [published]`.
- **PyPI trusted publishing only works from the calling workflow.** The `pypi-publish` input in `semantic-release-uv.yml` will NOT work with OIDC — you must have a separate job in your own `release.yml`.
- **npm OIDC requires Node.js 24+** and **`ubuntu-latest`** (not Blacksmith). Earlier Node versions fail with "Access token expired or revoked".
- **npm trusted publisher workflow filename** must be the calling workflow (e.g. `npm-publish.yml`), not the reusable workflow.
- **MCP Registry "version already exists"** — if semantic-release runs but there's nothing to release, the MCP publish step would try to re-publish the same version. Gate on `released == 'true'` to avoid this.
