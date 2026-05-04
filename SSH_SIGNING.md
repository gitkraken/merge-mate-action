# SSH Signing Setup

Step-by-step guide for enabling SSH commit signing in Merge Mate.

Use this when:

- your repository requires signed commits on protected branches
- your PRs already contain signed commits and you do not want rebases to strip that signal
- you want Merge Mate to push rebased commits directly when signing is required

## What Merge Mate Does

When `signing-mode: ssh` is enabled, Merge Mate decides per PR whether it should:

- push unsigned commits
- sign rewritten commits with the bot SSH signing key
- fall back to hidden refs and post manual re-sign instructions

Merge Mate preserves the original commit author. Only the committer identity changes.

## Before You Start

You need:

- a GitHub account that will act as the committer
- a verified email on that account
- a passphrase-less SSH private key for signing
- the matching public key registered in GitHub as a signing key

## Step 1: Choose the Committer Identity

Pick the GitHub account that should appear as the committer on rewritten commits.

Example identity:

- name: `merge-mate[bot]`
- email: `merge-mate[bot]@users.noreply.github.com`

If you use a different bot account, use that account's verified email instead.

## Step 2: Generate a Signing Key

Generate a new passphrase-less SSH key:

```bash
ssh-keygen -t ed25519 -C "merge-mate[bot]@users.noreply.github.com" -f merge-mate-signing
```

This creates:

- `merge-mate-signing`
- `merge-mate-signing.pub`

Do not add a passphrase. Passphrase-protected keys are not supported in this phase.

## Step 3: Register the Public Key as a GitHub Signing Key

Sign in as the bot account and add `merge-mate-signing.pub` in GitHub under SSH signing keys.

Important:

- add it as a signing key, not as an authentication key
- use the same GitHub account that owns the committer email

If the key is added in the wrong place, GitHub may report `not_signing_key` or `unknown_key`.

## Step 4: Verify the Committer Email

Make sure the email you plan to use in `MERGE_MATE_GIT_COMMITTER_EMAIL` is verified on the same GitHub account.

If it is not, GitHub can mark the signature as unverified with reasons such as:

- `unverified_email`
- `no_user`

## Step 5: Save the Private Key as a Secret

Store the private key contents from `merge-mate-signing` as a repository or organization secret:

- secret name: `MERGE_MATE_SSH_SIGNING_KEY`

Copy the full private key exactly as exported.

## Step 6: Configure the Workflow

Add the signing env vars and enable `signing-mode: ssh` in your sync workflow.

Example:

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
      - uses: gitkraken/merge-mate-action/sync@v0.4
        with:
          signing-mode: ssh
```

Required values:

- `MERGE_MATE_SSH_SIGNING_KEY`: private key secret
- `MERGE_MATE_GIT_COMMITTER_NAME`: displayed committer name
- `MERGE_MATE_GIT_COMMITTER_EMAIL`: verified committer email on the bot account

## Step 7: Run a Test Sync

Trigger the sync workflow on a PR that matches one of these cases:

- the PR already contains signed commits
- the target branch requires signed commits

Expected result:

- Merge Mate rebases the PR
- rewritten commits are signed by the configured bot
- GitHub shows the rewritten commits as verified

## Step 8: Validate the Result

Check the PR after the workflow runs:

1. open the PR commits tab
2. inspect one of the rewritten commits
3. confirm GitHub marks the signature as verified
4. confirm the author is still the original author
5. confirm the committer is the configured bot identity

## If Merge Mate Falls Back to Hidden Refs

If signing is required but Merge Mate cannot push safely, it will:

- publish the rebased result to hidden refs
- post manual re-sign instructions in the PR comment
- avoid the normal apply checkbox flow

This is expected behavior for cases such as:

- missing signing configuration
- signing setup failure on the runner
- GitHub rejecting the push with `GH006`

## Troubleshooting

### `unknown_key`

The public key is not registered as a signing key on the bot account.

Fix:

- upload the public key to the same GitHub account as a signing key

### `not_signing_key`

The key was added as an SSH authentication key instead of a signing key.

Fix:

- remove it and add it again as a signing key

### `unverified_email`

The committer email is not verified on the bot account.

Fix:

- update `MERGE_MATE_GIT_COMMITTER_EMAIL` to a verified email
- or verify the email on the bot account

### `no_user`

GitHub cannot associate the committer email with an account.

Fix:

- use an email that belongs to the bot GitHub account

### `malformed_signature`

The private key secret was likely saved incorrectly.

Fix:

- re-save `MERGE_MATE_SSH_SIGNING_KEY` from the original private key file
- preserve the full key contents, including the final newline

### Merge Mate posts manual re-sign instructions

Signing was required, but the runner could not safely produce a push that GitHub would accept.

Fix:

- verify the secret is present
- verify the bot name and email are configured
- verify the public key is registered as a signing key
- check whether GitHub rejected the push because signed commits are required

## Recommended First Rollout

Start with one protected branch and one bot identity.

Good first rollout path:

1. configure signing in a test repository
2. rebase a PR that already contains signed commits
3. confirm GitHub verifies the rewritten commits
4. enable the same setup in production repositories

## Related Docs

- [README](./README.md)
- [Workflow Examples](./EXAMPLES.md)
