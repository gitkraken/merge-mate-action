# Merge Mate - Action

A GitHub Action that syncs pull requests with their target branches and optionally uses AI to resolve conflicts.

## Prerequisites

1. **Install the GitHub App** — [Install Merge Mate](https://github.com/apps/gitkraken-services) on your repository
2. **Add workflow files** — Create the YAML files below manually (the app does not generate them automatically)

## Quick Start

Create two workflow files in your repository:

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
      - uses: gitkraken/merge-mate-action/sync@v0.2
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
    if: >-
      github.event_name == 'workflow_dispatch' ||
      (github.event.issue.pull_request && github.event.sender.type != 'Bot')
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/review@v0.2
        with:
          pr-number: ${{ inputs.pr-number }}
          action: ${{ inputs.action }}
```

When the target branch is updated, sync runs automatically. PR comments appear with diff preview and checkboxes. You can also manually trigger apply/undo from the Actions tab using workflow_dispatch or via the Conflict viewer link in the PR message.

## Features

- **Flexible Sync** — Rebase (linear history) or Merge (merge commits)
- **AI Conflict Resolution** — Automatic conflict resolution with AI
- **Safe by Default** — AI resolutions stored in hidden refs until approved; clean rebases are applied directly
- **Parallel Processing** — Configurable concurrency
- **Detailed Reports** — PR comments with diffs, GitHub Summary

## Sync Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `github-token` | `${{ github.token }}` | GitHub token for authentication |
| `mode` | `rebase` | `rebase` or `merge` |
| `pr-filter` | — | YAML filter for selecting PRs: `ids`, `target-branches`, `created`, `updated`, `authors` (supports `@org/team-slug` tokens) |
| `concurrency` | `3` | Maximum number of PRs to process in parallel |
| `apply-policy` | `auto` | `auto` — apply above threshold. `resolved-only` — same but skip clean rebases. `review` — push to hidden ref for review. `dry-run` — no push |
| `confidence-threshold` | `100` | Minimum AI confidence (0–100) to auto-apply. `100` = only when fully confident |
| `ai-provider` | `none` | AI provider: `none` \| `gitkraken` |
| `ai-model` | — | AI model identifier (provider-specific) |
| `ai-api-key` | — | API key or token for the AI provider |
| `ai-api-base` | — | Custom API base URL |
| `exclude-files` | see below | Newline-separated glob patterns for files to exclude from AI resolution |
| `diff-viewer-base-url` | `https://gitkraken.dev` | Base URL for the diff viewer |
| `gk-api-base` | — | GitKraken API base URL for OIDC token exchange |
| `telemetry` | `true` | Enable telemetry and error tracking |
| `log-level` | `info` | `error` \| `warn` \| `info` \| `debug` |

## Review Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `github-token` | `${{ github.token }}` | GitHub token for authentication |
| `pr-number` | — | PR number to process (required for `workflow_dispatch`) |
| `action` | — | `apply` or `undo` (required for `workflow_dispatch`) |
| `gk-api-base` | — | GitKraken API base URL for OIDC token exchange |
| `telemetry` | `true` | Enable telemetry and error tracking |
| `log-level` | `info` | `error` \| `warn` \| `info` \| `debug` |

## Excluding Files from AI Resolution

Lock files are **always** excluded from AI resolution — conflicted lock files are resolved by taking the target branch version.

When `exclude-files` is not set, the following defaults apply:

| Pattern | Description |
|---------|-------------|
| `**/package-lock.json`, `**/pnpm-lock.yaml`, `**/yarn.lock`, `**/cargo.lock` | JS/Rust lock files |
| `**/gemfile.lock`, `**/poetry.lock`, `**/go.sum`, `**/composer.lock` | Ruby/Python/Go/PHP lock files |
| `**/gradle.lock`, `**/maven.lock`, `**/*.lockfile` | JVM/generic lock files |
| `**/*.min.js`, `**/*.min.css` | Minified files |
| `**/*.bundle.js`, `**/*.bundle.css` | Bundled files |
| `**/*.generated.*`, `**/*.auto.*` | Generated code markers |
| `**/*.g.dart`, `**/*.g.ts` | Dart/TS code generation |
| `**/*.pb.go`, `**/*.pb.ts` | Protobuf generated code |
| `**/generated/**`, `**/dist/**`, `**/build/**` | Generated/build directories |

Custom patterns are **appended** to the defaults:

```yaml
- uses: gitkraken/merge-mate-action/sync@v0.2
  with:
    exclude-files: |
      **/vendor/**
      **/fixtures/**
```

## More Examples

See [EXAMPLES.md](./EXAMPLES.md) for ready-to-use workflow presets: apply policies, dry run, manual trigger, PR filtering, and more.

## Versioning

**For v0.x.y (pre-release):**

- Pin to `@v0.2` — patches within the same minor version
- Breaking changes may occur between minors

**For v1+ (stable):**

- Pin to `@v1` — all compatible updates
