# Merge Mate — Workflow Examples

Ready-to-use workflow configurations. Copy, adjust branch names and secrets, commit to `.github/workflows/`.

All examples assume the [review workflow](./README.md#quick-start) is already set up.

All sync examples include a `concurrency` block to prevent overlapping runs from pushing to the same refs simultaneously. The default group key `merge-mate-sync-${{ github.ref }}` serializes runs per trigger branch. You can customize the key for your use case — for example, include a PR filter input so that manual runs with disjoint PR sets can execute in parallel.

If your repository receives PRs from forks, read the fork-specific examples below and apply the same fork credential choice to both the sync and review workflows.

`github-token` has no implicit default. Examples that omit it intentionally rely on GitKraken App authentication via OIDC and therefore include `permissions: id-token: write`. For simple runs without OIDC, pass `github-token: ${{ github.token }}` explicitly.

## Rebase without AI

Keeps PRs up-to-date with `main`. Conflicts are reported but not resolved.

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
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          github-token: ${{ github.token }}
```

## Rebase with GitKraken AI

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

## Review resolved conflicts

Resolved conflicts are saved to hidden refs for review. Clean rebases are skipped, so nothing is pushed to the PR branch automatically in this mode.

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

## SSH signing for protected branches

Enables SSH signing when Merge Mate needs it, which preserves signed-commit requirements on protected branches and on PRs that already contain signed commits.

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
    env:
      MERGE_MATE_SSH_SIGNING_KEY: ${{ secrets.MERGE_MATE_SSH_SIGNING_KEY }}
      MERGE_MATE_GIT_COMMITTER_NAME: merge-mate[bot]
      MERGE_MATE_GIT_COMMITTER_EMAIL: merge-mate[bot]@users.noreply.github.com
    steps:
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          github-token: ${{ github.token }}
          signing-mode: ssh
```

If signing is required but the bot key is unavailable or GitHub still rejects the push, Merge Mate falls back to hidden refs and posts manual re-sign instructions instead of the normal apply checkbox.

## Private forks with the GitKraken App installed on the fork owner

Use this when you can install the app on both the base repository and the account or organization that owns the fork repository. This is the cleanest option for private forks because Merge Mate can mint a fork-specific installation token when it needs to push the source branch.

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

Requirements:

- The GitKraken App must be installed on the base repository.
- The same app must also be installed on the owner of the fork.
- The fork repository must be included in that installation.

## Private forks with fork-push-token

Use this when you cannot install the app on the fork owner, or when you want a deterministic fallback path for pushing fork branches.

Sync workflow:

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
          fork-push-token: ${{ secrets.MERGE_MATE_FORK_PUSH_TOKEN }}
```

Review workflow:

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
          fork-push-token: ${{ secrets.MERGE_MATE_FORK_PUSH_TOKEN }}
```

Notes:

- Use the same secret in both workflows.
- The token must be able to update the fork branch.
- If the token can read the fork repository but GitHub still rejects the final push, Merge Mate will leave manual commands in the PR comment and disable the interactive apply retry.

## Using github.token as a best-effort fork fallback

This can work in some repositories, but it is not the recommended primary solution for private forks because GitHub scopes `GITHUB_TOKEN` to the workflow repository.

```yaml
      - uses: gitkraken/merge-mate-action/sync@v0.2
        with:
          fork-push-token: ${{ github.token }}
```

If you use this approach:

- Keep `permissions: contents: write` in the workflow.
- Expect it to be best-effort only for private forks.
- Prefer a dedicated secret or a fork-owner app installation if you need a reliable apply/undo flow.

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
