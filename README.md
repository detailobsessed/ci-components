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

jobs:
  release:
    uses: detailobsessed/ci-components/.github/workflows/semantic-release-bun.yml@main
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**With custom options:**

```yaml
jobs:
  release:
    uses: detailobsessed/ci-components/.github/workflows/semantic-release-bun.yml@main
    with:
      node-version: "22"
      bun-version: "latest"
      build-command: "bun run build"
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
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

## Setup for New Projects

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

3. Create workflow file calling this reusable workflow.
