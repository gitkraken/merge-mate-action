# Merge Mate

A GitHub Action that syncs pull requests with their target branches and optionally uses AI to resolve conflicts.

## Prerequisites

1. **Install the GitHub App** — [Install Merge Mate](https://github.com/apps/gitkraken-services) on your repository
2. **Add workflow files** — Create the YAML files below manually (the app does not generate them automatically)

## Quick Start

Manually create two workflow files in your repository:

**`.github/workflows/merge-mate.yml`** — syncs all PRs when the target branch is updated:

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
      - uses: gitkraken/merge-mate-action/sync@v0.1
        with:
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
```

**`.github/workflows/merge-mate-review.yml`** — handles apply/undo via PR checkbox or manual trigger:

```yaml
name: Merge Mate Review
on:
  issue_comment:
    types: [edited]
  workflow_dispatch:
    inputs:
      pr-number:
        description: "PR number to process"
        required: true
        type: number
      action:
        description: "Action to perform"
        required: true
        type: choice
        options:
          - apply
          - undo
permissions:
  contents: write
  pull-requests: write
  id-token: write
concurrency:
  group: merge-mate-review-${{ github.event.issue.number || inputs.pr-number }}
  cancel-in-progress: false
jobs:
  review:
    if: github.event_name == 'workflow_dispatch' || github.event.issue.pull_request
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/review@v0.1
        with:
          pr-number: ${{ inputs.pr-number }}
          action: ${{ inputs.action }}
```

When the target branch is updated, sync runs automatically. PR comments appear with diff preview and checkboxes. You can
also manually trigger apply/undo from the Actions tab using workflow_dispatch or via the Conflict viewer link in the PR
message.

## Features

- **Flexible Sync** — Rebase (linear history) or Merge (merge commits)
- **AI Conflict Resolution** — Automatic conflict resolution with AI
- **Safe by Default** — Changes stored in hidden refs until approved
- **Parallel Processing** — Configurable concurrency
- **Detailed Reports** — PR comments with diffs, GitHub Summary

## Versioning

**For v0.x.y (pre-release):**

- Pin to `@v0.1`, `@v0.2` — patches within the same minor version
- Breaking changes may occur between minors

**For v1+ (stable):**

- Pin to `@v1` — all compatible updates

## Excluding Files from AI Resolution

Lock files (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, etc.) are **always** excluded from AI resolution — conflicted lock files are resolved by taking the target branch version.

All other files can be excluded via the `exclude-from-ai` input using newline-separated glob patterns. When this input is empty, the following defaults apply:

| Pattern | Description |
|---------|-------------|
| `**/*.min.js`, `**/*.min.css` | Minified files |
| `**/*.bundle.js`, `**/*.bundle.css` | Bundled files |
| `**/*.generated.*`, `**/*.auto.*` | Generated code markers |
| `**/*.g.dart`, `**/*.g.ts` | Dart/TS code generation |
| `**/*.pb.go`, `**/*.pb.ts` | Protobuf generated code |
| `**/generated/**` | Generated directories |
| `**/dist/**`, `**/build/**` | Build output directories |

Providing **any** value replaces the defaults entirely:

```yaml
- uses: gitkraken/merge-mate-action/sync@v0.1
  with:
    exclude-from-ai: |
      **/dist/**
      **/vendor/**
      **/*.auto.*
```

## Documentation

See action.yml files for all available inputs and outputs.
