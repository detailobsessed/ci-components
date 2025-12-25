# CI Components

Reusable GitHub Actions workflows for personal projects.

## Available Workflows

### Semantic Release (Bun)

Automated versioning and changelog generation for Bun/TypeScript projects.

**Usage:**

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

Publish to npm with OIDC provenance (no `NPM_TOKEN` secret needed). Triggered by GitHub releases.

**Usage:**

```yaml
# .github/workflows/npm-publish.yml
name: NPM Publish

on:
  release:
    types: [published]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  publish:
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
