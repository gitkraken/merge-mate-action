# Merge Mate — Workflow Examples

Ready-to-use workflow configurations. Copy, adjust branch names and secrets, commit to `.github/workflows/`.

All examples assume the [review workflow](./README.md#quick-start) is already set up.

All sync examples include a `concurrency` block to prevent overlapping runs from pushing to the same refs simultaneously. The default group key `merge-mate-sync-${{ github.ref }}` serializes runs per trigger branch. You can customize the key for your use case — for example, include a PR filter input so that manual runs with disjoint PR sets can execute in parallel.

## Rebase with AI

Resolves conflicts with AI. Clean rebases are pushed directly to the PR branch. AI-resolved conflicts are saved to hidden refs for review — the default confidence threshold is 100%, so only perfect resolutions are auto-applied.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main]
concurrency:
  group: merge-mate-sync-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
```

## Auto-apply AI resolutions

Lowers the confidence threshold so AI-resolved branches are pushed directly when confidence is at or above 80%.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main]
concurrency:
  group: merge-mate-sync-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          confidence-threshold: "80"
```

## Resolved-only

Only pushes AI-resolved branches. Clean rebases (no conflicts) are skipped — useful when you want AI to handle conflicts but don't want to trigger CI for PRs that had no issues.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main]
concurrency:
  group: merge-mate-sync-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          apply-policy: resolved-only
          confidence-threshold: "80"
```

## Hidden-only (review everything)

All results — including clean rebases — are saved to hidden refs. Nothing is pushed to the PR branch until approved via checkbox or workflow_dispatch.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main]
concurrency:
  group: merge-mate-sync-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          apply-policy: review
```

## Dry run

Preview what would happen without pushing anything. Comments are posted only on errors.

```yaml
name: Merge Mate Sync (Dry Run)
on:
  push:
    branches: [main]
concurrency:
  group: merge-mate-sync-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          apply-policy: dry-run
```

## Manual trigger

Run sync on demand from the Actions tab. Useful for testing or one-off syncs.

```yaml
name: Merge Mate Sync (Manual)
on:
  workflow_dispatch:
    inputs:
      mode:
        description: "Sync strategy"
        required: true
        type: choice
        options:
          - rebase
          - merge
        default: rebase
      apply-policy:
        description: "Apply policy"
        required: true
        type: choice
        options:
          - auto
          - resolved-only
          - review
          - dry-run
        default: review
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          mode: ${{ inputs.mode }}
          apply-policy: ${{ inputs.apply-policy }}
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
```

## Filter specific PRs

Only sync PRs targeting `main` or `release/*` branches, created in the last 7 days.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main, "release/*"]
concurrency:
  group: merge-mate-sync-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          pr-filter: |
            target-branches:
              - main
              - release/*
            created: 7d
```

## Filter by team

Only sync PRs authored by members of a GitHub team. Requires `read:org` scope (PAT) or `members: read` permission (GitHub App).

```yaml
          pr-filter: |
            authors:
              - @gitkraken/frontend
              - !@gitkraken/bots
```

## Push + manual (combined)

Automatic sync on push to `main`, plus manual trigger for ad-hoc runs.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      apply-policy:
        description: "Apply policy"
        required: true
        type: choice
        options:
          - auto
          - review
          - dry-run
        default: review
concurrency:
  group: merge-mate-sync-${{ github.ref }}
  cancel-in-progress: true
permissions:
  contents: write
  pull-requests: write
  id-token: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          apply-policy: ${{ inputs.apply-policy || 'auto' }}
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
```
