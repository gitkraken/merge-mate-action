# Merge Mate — Workflow Examples

Ready-to-use workflow templates live in [`templates/`](./templates/). Copy a template, replace `{{DEFAULT_BRANCH}}` and `{{VERSION}}`, commit to `.github/workflows/`.

All examples assume the [review workflow](./templates/review.yml) is already set up. Use the [Setup Wizard](https://gitkraken.dev/merge-mate/setup) or copy a template manually.

All sync templates include a `concurrency` block to prevent overlapping runs from pushing to the same refs simultaneously. The default group key `merge-mate-sync-${{ github.ref }}` serializes runs per trigger branch. You can customize the key for your use case — for example, include a PR filter input so that manual runs with disjoint PR sets can execute in parallel.

`github-token` has no implicit default. Templates rely on GitKraken App authentication via OIDC and therefore include `permissions: id-token: write`. For simple runs without OIDC, pass `github-token: ${{ github.token }}` explicitly.

## Templates

| Template | Triggers | Description |
|----------|----------|-------------|
| [sync-push.yml](./templates/sync-push.yml) | `push` | Syncs PRs when the target branch is updated |
| [sync-cron.yml](./templates/sync-cron.yml) | `schedule` + `push` | Syncs on a cron schedule and on push |
| [sync-manual.yml](./templates/sync-manual.yml) | `workflow_dispatch` | On-demand sync from the Actions tab |
| [sync-push-cron.yml](./templates/sync-push-cron.yml) | `push` + `schedule` + `workflow_dispatch` | Combined: auto-sync on push and schedule, plus manual trigger |
| [review.yml](./templates/review.yml) | `issue_comment` + `repository_dispatch` | Apply or undo AI-resolved changes |
| [review-manual.yml](./templates/review-manual.yml) | `issue_comment` + `repository_dispatch` + `workflow_dispatch` | Same as review, plus manual trigger from Actions tab |

## Apply Policies

Control what happens after sync by adding `apply-policy` to the `with:` block of any sync template.

**Auto-apply high-confidence resolutions** — pushes AI-resolved branches directly when confidence is at or above 80%:

```yaml
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          confidence-threshold: "80"
```

**Resolved-only** — only pushes AI-resolved branches, skips clean rebases:

```yaml
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          apply-policy: resolved-only
          confidence-threshold: "80"
```

**Review** — saves all non-clean results to hidden refs for review:

```yaml
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          apply-policy: review
```

**Dry run** — preview what would happen without pushing anything:

```yaml
        with:
          ai-api-key: ${{ secrets.MERGE_MATE_API_KEY }}
          apply-policy: dry-run
```

## SSH Signing for Protected Branches

Enables SSH signing when Merge Mate needs it, which preserves signed-commit requirements on protected branches and on PRs that already contain signed commits.

Add these environment variables and the `signing-mode` input to any sync template:

```yaml
    env:
      MERGE_MATE_SSH_SIGNING_KEY: ${{ secrets.MERGE_MATE_SSH_SIGNING_KEY }}
      MERGE_MATE_GIT_COMMITTER_NAME: merge-mate[bot]
      MERGE_MATE_GIT_COMMITTER_EMAIL: merge-mate[bot]@users.noreply.github.com
    steps:
      - uses: gitkraken/merge-mate-action/sync@{{VERSION}}
        with:
          signing-mode: ssh
```

If signing is required but the bot key is unavailable or GitHub still rejects the push, Merge Mate falls back to hidden refs and posts manual re-sign instructions instead of the normal apply button.

## Fork Pushes

### Private forks with the GitKraken App installed on the fork owner

Use this when you can install the app on both the base repository and the account or organization that owns the fork repository. This is the cleanest option for private forks because Merge Mate can mint a fork-specific installation token when it needs to push the source branch.

No extra inputs needed — OIDC handles authentication for both repositories.

Requirements:

- The GitKraken App must be installed on the base repository.
- The same app must also be installed on the owner of the fork.
- The fork repository must be included in that installation.

### Private forks with fork-push-token

Use this when you cannot install the app on the fork owner, or when you want a deterministic fallback path for pushing fork branches.

Add `fork-push-token` to both the sync and review workflows:

```yaml
        with:
          ai-provider: gitkraken
          ai-api-key: ${{ secrets.GK_AI_PROVISIONER_TOKEN }}
          fork-push-token: ${{ secrets.MERGE_MATE_FORK_PUSH_TOKEN }}
```

Review workflow:

```yaml
        with:

          fork-push-token: ${{ secrets.MERGE_MATE_FORK_PUSH_TOKEN }}
```

Notes:

- Use the same secret in both workflows.
- The token must be able to update the fork branch.
- If the token can read the fork repository but GitHub still rejects the final push, Merge Mate will leave manual commands in the PR comment and disable the interactive apply retry.

### Using github.token as a best-effort fork fallback

This can work in some repositories, but it is not the recommended primary solution for private forks because GitHub scopes `GITHUB_TOKEN` to the workflow repository.

```yaml
        with:
          fork-push-token: ${{ github.token }}
```

If you use this approach:

- Keep `permissions: contents: write` in the workflow.
- Expect it to be best-effort only for private forks.
- Prefer a dedicated secret or a fork-owner app installation if you need a reliable apply/undo flow.

## PR Filtering

### Filter by target branches

Only sync PRs targeting `main` or `release/*` branches, created in the last 7 days:

```yaml
        with:
          pr-filter: |
            target-branches:
              - main
              - release/*
            created: 7d
```

### Filter by team

Only sync PRs authored by members of a GitHub team. Requires `read:org` scope (PAT) or `members: read` permission (GitHub App):

```yaml
        with:
          pr-filter: |
            authors:
              - '@gitkraken/frontend'
              - '!@gitkraken/bots'
```

### Filter by labels

Scope Merge Mate to PRs that carry specific GitHub labels, or explicitly skip labeled PRs. Matching is case-insensitive:

```yaml
        with:
          pr-filter: |
            labels:
              - auto-sync
              - '!do-not-rebase'
```

### Include draft PRs (stacked-PR workflows)

By default, Merge Mate skips draft PRs. When a draft sits in the middle of a stacked-PR dependency chain, exclusion breaks topological processing for the entire stack. Opt in to process drafts alongside ready PRs:

```yaml
        with:
          pr-filter: |
            include-drafts: true
            target-branches:
              - main
              - release/*
```
