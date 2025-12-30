# CI Components

Reusable GitHub Actions workflows for personal projects.

## Available Workflows

### Semantic Release (Bun)

Automated versioning and changelog generation for Bun/TypeScript projects.

**Usage (basic):**

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    uses: detailobsessed/ci-components/.github/workflows/semantic-release-bun.yml@main
    secrets: inherit
```

**Usage (gated on CI):**

To ensure releases only happen after CI passes:

```yaml
# .github/workflows/release.yml
name: Release

on:
  workflow_run:
    workflows: ["CI"]
    branches: [main]
    types: [completed]
  workflow_dispatch:

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/semantic-release-bun.yml@main
    secrets: inherit
```

This prevents broken commits from being released before CI catches failures.

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `node-version` | `22` | Node.js version (semantic-release v25 requires ^22.14.0) |
| `bun-version` | `latest` | Bun version |
| `build-command` | `bun run build` | Build command to run before release |

**Required in calling repo:**

- `.releaserc.json` - semantic-release configuration
- `package.json` with semantic-release dev dependencies

---

### NPM Publish (Bun)

Publish to npm with OIDC provenance (no `NPM_TOKEN` secret needed).

> **⚠️ Important:** Do NOT use `on: release: types: [published]` as the trigger. Releases created with `GITHUB_TOKEN` (which semantic-release uses) do NOT emit `release` events. Use `workflow_run` instead.

**Usage:**

```yaml
# .github/workflows/npm-publish.yml
name: NPM Publish

on:
  workflow_run:
    workflows: ["Release"]
    branches: [main]
    types: [completed]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  publish:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/npm-publish-bun.yml@main
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `node-version` | `24` | Node.js version (24+ required for OIDC) |
| `bun-version` | `latest` | Bun version |
| `build-command` | `bun run build` | Build command to run before publish |

**Prerequisites:**

1. Configure [npm trusted publishing](https://docs.npmjs.com/trusted-publishers/) for your package at npmjs.com
2. Set `"publishConfig": { "access": "public" }` in `package.json` for scoped packages

**Note:** Provenance attestations are automatically generated when publishing via trusted publishing from public repos.

---

## Setup for New Projects

### GitHub-only releases (no npm)

1. Add `.releaserc.json`:

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

2. Install dependencies:

```bash
bun add -d semantic-release @semantic-release/changelog @semantic-release/git
```

3. Create `release.yml` workflow calling the semantic-release workflow.

### With npm publishing

Use the same setup as above, plus:

4. Create `npm-publish.yml` workflow calling the npm-publish workflow.
5. Configure npm trusted publishing for your GitHub repo.

---

### Semantic Release (Python/UV)

Automated versioning and changelog generation for Python projects using uv.

**Usage:**

```yaml
# .github/workflows/release.yml
name: release

on:
  workflow_run:
    workflows: [ci]
    types: [completed]
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write
  id-token: write

jobs:
  release:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/semantic-release-uv.yml@main
    with:
      pypi-publish: "false"  # See PyPI publishing section below
    secrets: inherit

  # PyPI publish must be in calling workflow for trusted publishing to work
  pypi-publish:
    needs: release
    if: needs.release.outputs.released == 'true' && vars.PYPI_PUBLISH == 'true'
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/your-package
    permissions:
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv build
      - run: uv publish
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `python-version` | `3.13` | Python version |
| `runner` | `blacksmith-4vcpu-ubuntu-2404` | GitHub Actions runner |
| `pypi-publish` | `false` | PyPI publish mode: `true`, `test`, or `false` |

**Outputs:**

| Output | Description |
|--------|-------------|
| `released` | `true` if a new release was created, `false` otherwise |

**Required in calling repo:**

- `pyproject.toml` with `[tool.semantic_release]` configuration
- `python-semantic-release` in dev dependencies (`uv add --group maintain python-semantic-release`)

> **⚠️ PyPI Trusted Publishing:** The `pypi-publish` input in this workflow will NOT work with PyPI trusted publishing because OIDC tokens from reusable workflows contain the reusable workflow's repository in the claims, not the calling repository. You MUST add a separate `pypi-publish` job in your calling workflow as shown above.

---

### MCP Registry Publish (Bun)

Publish MCP servers to the [official MCP Registry](https://registry.modelcontextprotocol.io/) using OIDC authentication.

**Usage:**

```yaml
# .github/workflows/mcp-registry-publish.yml
name: MCP Registry Publish

on:
  workflow_run:
    workflows: ["NPM Publish"]
    types: [completed]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  publish:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: detailobsessed/ci-components/.github/workflows/mcp-registry-publish-bun.yml@main
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `runner` | `blacksmith-4vcpu-ubuntu-2404` | Runner to use (Blacksmith works with OIDC) |

**Prerequisites:**

1. Create `server.json` with MCP server metadata ([schema](https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json))
2. For npm packages: Add `mcpName` to `package.json`: `"mcpName": "io.github.{org}/{server-name}"`
3. For PyPI packages: Add `<!-- mcp-name: io.github.{org}/{server-name} -->` to README.md (ownership validation)
4. For PyPI packages: Include `version` in `packages[]` array (required despite schema saying optional)
5. For org namespaces, ensure your org membership is **public** at `https://github.com/orgs/{org}/people`
6. First publish must be done manually: `mcp-publisher login github && mcp-publisher publish`

**Tip:** Use `@semantic-release/exec` (npm) or `version_variables` (python-semantic-release) to keep `server.json` version in sync:

```json
["@semantic-release/exec", {
  "prepareCmd": "sed -i'' -e 's/\"version\": \"[^\"]*\"/\"version\": \"${nextRelease.version}\"/g' server.json"
}]
```

---

### Auto Merge Dependabot

Automatically merge Dependabot PRs for minor and patch updates.

**Usage:**

```yaml
# .github/workflows/auto-merge.yml
name: Auto Merge Dependabot PRs

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    uses: detailobsessed/ci-components/.github/workflows/auto-merge-dependabot.yml@main
    secrets: inherit
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `merge-method` | `merge` | Merge method (merge, squash, rebase) |
| `update-types` | `minor,patch` | Update types to auto-merge (comma-separated) |
