# Release Action

Automated release workflow for multi-package repositories.

## Overview

This action analyzes commits since the last release tag, automatically determines the version bump type (major/minor/patch), and:
- **PR mode**: Creates a release PR
- **Direct mode**: Commits and creates tags immediately
- **Merge mode**: Creates tags + GitHub releases (after PR merge)

## Features

- Analyzes commits since last release tag
- Auto-detects bump type from commit messages (`[feat]`, `[fix]`, breaking changes)
- Supports manual version bump override
- Three release modes: PR, direct, or merge
- Skips release if no new commits since last release
- Creates git tags with configurable prefix
- Syncs `Cargo.lock` when Rust package versions are bumped

## Usage

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: libnudget/release@v1.0.0
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
| release_mode | `pr`, `direct`, or `merge` | No | `pr` |

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

Creates a PR with version bumps. When a Rust package is bumped, the action also runs `cargo metadata` against the first changed package manifest so `Cargo.lock` is included in the same PR. Tag creation happens when PR is merged (via webhook or manual trigger).

```yaml
- uses: libnudget/release@v1.0.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    release_mode: pr
```

## Mode: Direct

Commits version bumps to main, syncs `Cargo.lock` for Rust packages, and creates git tags immediately.

```yaml
- uses: libnudget/release@v1.0.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    release_mode: direct
```

## Mode: Merge

Creates git tags and GitHub releases from merged PR versions. Use after PR merge to create tags and releases.

```yaml
- uses: libnudget/release@v1.0.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    release_mode: merge
```

## Trigger Commands

Each mode can be triggered via GitHub CLI or workflow_dispatch:

### PR Mode (create release PR)
```bash
gh workflow run release.yml -f version_bump=patch -f release_mode=pr
gh workflow run release.yml -f version_bump=minor -f release_mode=pr
gh workflow run release.yml -f version_bump=major -f release_mode=pr
```

### Direct Mode (commit + tag immediately)
```bash
gh workflow run release.yml -f release_mode=direct
```

### Merge Mode (create tags + releases after PR merge)
```bash
gh workflow run release.yml -f release_mode=merge
```

## Full Example

```yaml
name: Release

on:
  push:
    branches: [main]
    paths: ['lib/**']
  pull_request:
    types: [closed]
    branches: [main]
  workflow_dispatch:
    inputs:
      version_bump:
        type: choice
        options: [patch, minor, major]
      release_mode:
        type: choice
        options: [pr, direct, merge]

jobs:
  release-pr:
    if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: libnudget/release@v1.0.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          packages: '[{"path": "lib/harper-core", "name": "harper-core"}, {"path": "lib/harper-ui", "name": "harper-ui"}]'
          version_bump: ${{ github.event.inputs.version_bump || 'patch' }}
          release_mode: ${{ github.event.inputs.release_mode || 'pr' }}

  create-tags:
    if: github.event_name == 'pull_request' && github.event.action == 'closed' && github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - uses: libnudget/release@v1.0.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          packages: '[{"path": "lib/harper-core", "name": "harper-core"}, {"path": "lib/harper-ui", "name": "harper-ui"}]'
          release_mode: merge
```

## Version Bump Logic

1. **Major**: If any commit contains `!:` or `BREAKING`
2. **Minor**: If any commit contains `[feat]` or `feat:`
3. **Patch**: Default (or manual override)

## License

MIT
