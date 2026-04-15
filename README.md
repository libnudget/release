# Release Action

Automated release workflow for multi-package repositories.

## Overview

This action analyzes commits since the last release tag, automatically determines the version bump type (major/minor/patch), and either:
- **PR mode**: Creates a release PR for review/merge
- **Direct mode**: Commits and creates git tags immediately

## Features

- Analyzes commits since last release tag
- Auto-detects bump type from commit messages (`[feat]`, `[fix]`, breaking changes)
- Supports manual version bump override
- Two release modes: PR or direct commit+tag
- Skips release if no new commits since last release
- Creates git tags with configurable prefix

## Usage

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: libnudget/release@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          packages: '[{"path": "lib/pkg1", "name": "pkg1"}, {"path": "lib/pkg2", "name": "pkg2"}]'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| token | GitHub token for API access | Yes | - |
| release_token | Separate token for PR creation | No | token |
| packages | JSON array of packages to release | Yes | - |
| version_bump | Manual version bump override | No | `patch` |
| tag_prefix | Prefix for git tags | No | `` |
| release_mode | `pr` or `direct` | No | `pr` |

## Outputs

| Output | Description |
|--------|------------|
| released | Whether a release was created (`true`/`false`) |
| versions | JSON object with package versions |

## Package Format

```json
[
  {"path": "lib/pkg1", "name": "pkg1"},
  {"path": "lib/pkg2", "name": "pkg2"}
]
```

## Mode: PR

Creates a PR with version bumps. Tag creation happens when PR is merged (via webhook or manual trigger).

```yaml
- uses: libnudget/release@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    release_mode: pr
```

## Mode: Direct

Commits version bumps to main and creates git tags immediately.

```yaml
- uses: libnudget/release@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    release_mode: direct
```

## Example

```yaml
name: Release

on:
  push:
    branches: [main]
    paths: ['lib/**']
  workflow_dispatch:
    inputs:
      version_bump:
        type: choice
        options: [patch, minor, major]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: libnudget/release@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          packages: '[{"path": "lib/harper-core", "name": "harper-core"}, {"path": "lib/harper-ui", "name": "harper-ui"}]'
          version_bump: ${{ github.event.inputs.version_bump || 'patch' }}
```

## Version Bump Logic

1. **Major**: If any commit contains `!:` or `BREAKING`
2. **Minor**: If any commit contains `[feat]` or `feat:`
3. **Patch**: Default (or manual override)

## License

MIT