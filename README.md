# Merge Mate

GitHub Action that syncs pull requests with target branches and optionally uses AI to resolve conflicts.

## Quick Start

Create two workflow files:

**`.github/workflows/merge-mate.yml`** — syncs all PRs when target branch is updated:

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gitkraken/merge-mate-action/sync@v0.1
        with:
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
```

**`.github/workflows/merge-mate-review.yml`** — handles apply/undo via PR checkbox:

```yaml
name: Merge Mate Review
on:
  issue_comment:
    types: [edited]
permissions:
  contents: write
  pull-requests: write
  id-token: write
concurrency:
  group: merge-mate-review-${{ github.event.issue.number }}
  cancel-in-progress: false
jobs:
  review:
    if: github.event.issue.pull_request
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/review@v0.1
```

When target branch is updated, sync runs automatically. PR comments appear with diff preview and checkboxes.

## Features

- **Flexible Sync** — Rebase (linear history) or Merge (merge commits)
- **AI Conflict Resolution** — Automatic conflict resolution with AI
- **Safe by Default** — Changes stored in hidden refs until approved
- **Parallel Processing** — Configurable concurrency
- **Detailed Reports** — PR comments with diffs, GitHub Summary

## Versioning

**For v0.x.y (pre-release):**

- Pin to `@v0.1`, `@v0.2` — patches within minor
- Breaking changes may occur between minors

**For v1+ (stable):**

- Pin to `@v1` — all compatible updates

## Documentation

See action.yml files for all available inputs and outputs.
