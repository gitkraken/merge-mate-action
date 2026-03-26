# Merge Mate - Action

A GitHub Action that syncs pull requests with their target branches and uses AI to resolve conflicts.

## Prerequisites

1. **Install the GitHub App** — [Install Merge Mate](https://github.com/apps/gitkraken-services) on your repository. Select "Only select repositories" for least-privilege access. The app appears as **GitKraken Services** on all GitHub UI surfaces.

   > After installing, GitHub redirects to your account settings. Return here to continue setup.

2. **Get an API key** — Go to [Settings](https://gitkraken.dev/mergemate/settings), sign in, and create an API key.

3. **Add the key to GitHub Secrets** — In your repository, go to **Settings → Secrets and variables → Actions → New repository secret**. Name it `MERGE_MATE_API_KEY` and paste the key.

4. **Add workflow files** — Create the YAML files below manually (the app does not generate them automatically).

## Quick Start

Create two workflow files in your repository:

**`.github/workflows/merge-mate.yml`** — syncs all PRs when the target branch is updated:

```yaml
name: Merge Mate Sync
on:
  push:
    branches: [main] # ← replace with your default branch
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

## Features

- **Flexible Sync** — Rebase (linear history) or Merge (merge commits)
- **AI Conflict Resolution** — Automatic conflict resolution with AI
- **Safe by Default** — AI resolutions stored in hidden refs until approved; clean rebases are applied directly
- **Parallel Processing** — Configurable concurrency
- **Detailed Reports** — PR comments with diffs, GitHub Summary

## How It Works

When the target branch updates, Merge Mate rebases or merges all open pull requests that target that branch.

The outcome depends on the **apply-policy** and **confidence-threshold** settings:

| Apply Policy     | Clean Rebase (no conflicts) | AI-Resolved Conflicts                                                      |
| ---------------- | --------------------------- | -------------------------------------------------------------------------- |
| `auto` (default) | Pushed to PR branch         | Pushed if confidence ≥ threshold, otherwise saved to hidden ref for review |
| `resolved-only`  | Skipped (no push)           | Same as `auto`                                                             |
| `review`         | Skipped (no push)           | Saved to hidden ref for review                                             |
| `dry-run`        | No push, report only        | No push, report only                                                       |

The default **confidence-threshold** is `100`. Merge Mate does not automatically apply changes unless the AI is fully confident.

To allow automatic application of high-confidence resolutions, lower the value (for example, `80`).

When Merge Mate saves a resolution to a hidden ref, it posts a pull request comment with:
- A diff preview
- An **Apply** checkbox

Select **Apply** to trigger the review workflow and push the resolution to the pull request branch.

Select **Undo** to revert the change.

Both actions force-push to the pull request branch. If you have a local checkout, run:

```bash
git fetch && git reset --hard origin/<branch>
```

## Sync Inputs

| Input                  | Default                         | Description                                                                                                                                                        |
| ---------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `github-token`         | `${{ github.token }}`           | GitHub token for authentication                                                                                                                                    |
| `mode`                 | `rebase`                        | `rebase` or `merge`                                                                                                                                                |
| `pr-filter`            | —                               | YAML filter for selecting PRs: `ids`, `target-branches`, `created`, `updated`, `authors` (supports `@org/team-slug` tokens)                                        |
| `concurrency`          | `3`                             | Maximum number of PRs to process in parallel                                                                                                                       |
| `apply-policy`         | `auto`                          | `auto` — apply above threshold. `resolved-only` — same but skip clean rebases. `review` — skip clean, save conflicts to hidden ref for review. `dry-run` — no push |
| `confidence-threshold` | `100`                           | Minimum AI confidence (0–100) to auto-apply. `100` = only when fully confident                                                                                     |
| `ai-model`             | `google/gemini-3-flash-preview` | AI model identifier                                                                                                                                                |
| `ai-api-key`           | —                               | **Required.** API key for AI conflict resolution                                                                                                                   |
| `ai-api-base`          | —                               | Custom API base URL                                                                                                                                                |
| `exclude-files`        | see below                       | Newline-separated glob patterns for files to exclude from AI resolution                                                                                            |
| `diff-viewer-base-url` | `https://gitkraken.dev`         | Base URL for the diff viewer                                                                                                                                       |
| `telemetry`            | `true`                          | Enable telemetry and error tracking                                                                                                                                |
| `log-level`            | `info`                          | `error` \| `warn` \| `info` \| `debug`                                                                                                                             |

## Review Inputs

| Input          | Default               | Description                                             |
| -------------- | --------------------- | ------------------------------------------------------- |
| `github-token` | `${{ github.token }}` | GitHub token for authentication                         |
| `pr-number`    | —                     | PR number to process (required for `workflow_dispatch`) |
| `action`       | —                     | `apply` or `undo` (required for `workflow_dispatch`)    |
| `telemetry`    | `true`                | Enable telemetry and error tracking                     |
| `log-level`    | `info`                | `error` \| `warn` \| `info` \| `debug`                  |

## Excluding Files from AI Resolution

By default, Merge Mate excludes lock files, minified bundles, generated code, and build artifacts from AI resolution.

For conflicted lock files, Merge Mate uses the target branch version.

Custom patterns are **appended** to the defaults:

```yaml
- uses: gitkraken/merge-mate-action/sync@v0.2
  with:
    exclude-files: |
      **/vendor/**
      **/fixtures/**
```

## Permissions

| Permission             | Why                                              |
| ---------------------- | ------------------------------------------------ |
| `contents: write`      | Push rebased branches and hidden refs            |
| `pull-requests: write` | Post and update PR comments                      |
| `id-token: write`      | Authenticate with GitKraken AI provider via OIDC |

`id-token: write` is only needed when using `ai-provider: gitkraken`. If you run without AI (or with a different provider), you can omit it.

## More Examples

See [EXAMPLES.md](./EXAMPLES.md) for ready-to-use workflow presets: apply policies, dry run, manual trigger, PR filtering, and more.

## Known Limitations

- **Committer identity** — All commits pushed by Merge Mate show `gitkraken-services[bot]` as the committer. The original author is preserved.
- **Commit signatures** — Rewritten commits lose their original signatures. If your branch protection requires signed commits, you may need to re-sign after apply.
- **Force push** — Apply, undo, and direct sync all force-push to the PR branch. Coordinate with co-authors before applying if they have local commits not yet pushed on the same branch.
- **Private forks** — The GitHub App must be installed on each fork individually. Installing it only on the base repository is not enough to push to fork branches.

## Troubleshooting

**"APIKey is required"** — The secret is missing or misconfigured. Verify that `MERGE_MATE_API_KEY` exists in Settings → Secrets and variables → Actions, the name matches exactly (case-sensitive), and the token is valid.

**"No PRs processed"** — Check that the `branches` trigger in your workflow matches the branch that was actually pushed. If you use `pr-filter`, verify the filter does not exclude all open PRs.

**Permission errors on push** — Ensure `contents: write` is set in the workflow `permissions` block. Repository-level settings (Settings → Actions → General → Workflow permissions) must also allow write access.

**Apply/Undo checkbox does nothing** — The review workflow must be set up separately (see Quick Start). Check that `.github/workflows/merge-mate-review.yml` exists and the `issue_comment` trigger is enabled.

## Cleanup

Over time, hidden refs (`refs/merge-mate/*`) accumulate in your repository. To clean them up, add a scheduled workflow:

> [!WARNING]
> This deletes all merge-mate refs including pending resolutions awaiting review.
> Any unapplied AI conflict resolutions will be lost.

```yaml
name: Merge Mate Cleanup
on:
  schedule:
    - cron: "0 3 * * 0" # weekly on Sunday at 3am UTC
  workflow_dispatch:
permissions:
  contents: write
jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Delete stale merge-mate refs
        run: |
          git ls-remote origin 'refs/merge-mate/*' | awk '{print $2}' | while read ref; do
            git push origin --delete "$ref" 2>/dev/null || true
          done
```

This deletes all hidden refs. Applied or undone resolutions are already on the PR branch, so removing the refs is safe.

## Versioning

**For v0.x.y (pre-release):**

- Pin to `@v0.2` — patches within the same minor version
- Breaking changes may occur between minors

**For v1+ (stable):**

- Pin to `@v1` — all compatible updates
