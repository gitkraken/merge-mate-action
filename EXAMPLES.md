# Merge Mate — Workflow Examples

Ready-to-use workflow configurations. Copy, adjust branch names and secrets, commit to `.github/workflows/`.

All examples assume the [review workflow](./README.md#quick-start) is already set up.

## Rebase without AI

Keeps PRs up-to-date with `main`. Conflicts are reported but not resolved.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
```

## Rebase with GitKraken AI

Resolves conflicts with AI. Clean rebases are pushed directly to the PR branch. AI-resolved conflicts are saved to hidden refs for review — the default confidence threshold is 100%, so only perfect resolutions are auto-applied.

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
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
```

## Auto-apply AI resolutions

Lowers the confidence threshold so AI-resolved branches are pushed directly when confidence is at or above 80%.

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
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
          confidence-threshold: "80"
```

## Resolved-only

Only pushes AI-resolved branches. Clean rebases (no conflicts) are skipped — useful when you want AI to handle conflicts but don't want to trigger CI for PRs that had no issues.

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
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
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
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
          apply-policy: hidden-only
```

## Dry run

Preview what would happen without pushing anything. Posts comments for all PRs regardless of outcome.

```yaml
name: Merge Mate Sync (Dry Run)
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
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
          apply-policy: dry-run
          comment-policy: all
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
          - hidden-only
          - dry-run
        default: hidden-only
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
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
```

## Filter specific PRs

Only sync PRs targeting `main` or `release/*` branches, created in the last 7 days.

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main, "release/*"]
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
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
          pr-filter: |
            target-branches:
              - main
              - release/*
            created: 7d
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
          - hidden-only
          - dry-run
        default: hidden-only
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
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
```
