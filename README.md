# Shared GitHub Workflows

Reusable workflows to plug into other repos via `uses: vinitu-net/github-workflows/.github/workflows/<workflow>.yml@<tag>`.

- Always pin to a release tag (auto-created on every merge to `main`); avoid `@main`.
- Each workflow accepts `gh_token`; if omitted it falls back to the built-in `GITHUB_TOKEN`.

## Shared workflows (reusable)

### `workflow-determine-version-bump.yml`
Detects which semver segment to bump for a PR targeting a branch.

**Inputs**
- `target-branch` (default `main`): only consider PRs targeting this branch.
- `major-branch-prefixes`, `minor-branch-prefixes`, `patch-branch-prefixes`: comma or newline separated branch prefixes to map to a bump.
- `default` (default `patch`): fallback bump when no prefixes match.
- `pull-requests`: PR payload when calling from `workflow_call` (e.g., `toJson(github.event.pull_request)`).

**Outputs**
- `version-bump`: `major`, `minor`, or `patch`.
- `matching_pr`: `true` if a PR against `target-branch` was found.

**Example**
```yaml
jobs:
  determine-version:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-determine-version-bump.yml@vX.Y.Z
    with:
      target-branch: main
      major-branch-prefixes: major/
      minor-branch-prefixes: |
        feature/
        features/
      patch-branch-prefixes: |
        fix/
        fixes/
      default: patch
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
```

### `workflow-merge-pull-requests.yml`
Auto-merges same-repo PRs into a target branch after checks pass. Skips forks, draft PRs, and branches starting with `wip`.

**Inputs**
- `target-branch` (default `main`): required base branch to merge into.
- `merge-method` (default `merge`): `merge`, `squash`, or `rebase`.
- `pull-requests`: PR payload for `workflow_call` (e.g., `toJson(github.event.pull_request)`).

**Outputs**
- `merged`: `true` if at least one PR was merged.
- `merged-prs`: JSON array with `number`, `title`, `author`, `head`.

**Example**
```yaml
jobs:
  merge:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-merge-pull-requests.yml@vX.Y.Z
    with:
      target-branch: main
      merge-method: squash
      pull-requests: ${{ toJson(github.event.pull_request) }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
```

### `workflow-create-tag.yml`
Creates and pushes the provided tag (no version calculation inside this workflow).

**Inputs**
- `target-branch` (default `main`): branch to check out before tagging.
- `next-tag` (required): tag to create (e.g., `v1.2.3`).
- `previous-tag` (required): previous tag that `next-tag` is based on.

**Outputs**
- `new-tag`: tag that was created (e.g., `v1.2.3`).
- `previous-tag`: previous tag that was passed in.

**Example**
```yaml
jobs:
  create-tag:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-create-tag.yml@vX.Y.Z
    with:
      target-branch: main
      next-tag: ${{ needs.calculate-tag.outputs.new-tag }}
      previous-tag: ${{ needs.calculate-tag.outputs.previous-tag }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
```

### `workflow-compute-next-tag.yml`
Calculates the next semver tag from the provided bump and latest existing `v*` tag.

**Inputs**
- `target-branch` (default `main`): branch to check out before reading tags.
- `version-bump` (required): `major`, `minor`, or `patch`.

**Outputs**
- `new-tag`: computed next tag (e.g., `v1.2.3`).
- `previous-tag`: latest existing tag before the bump (or `v0.0.0` if none).

**Example**
```yaml
jobs:
  calculate-tag:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-compute-next-tag.yml@vX.Y.Z
    with:
      target-branch: main
      version-bump: ${{ needs.determine-version.outputs.version-bump }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
```

### `workflow-update-version-file.yml`
Writes the provided tag into a version file on a target branch and pushes the commit.

**Inputs**
- `target-branch` (default `master`): branch to check out before writing the version file.
- `version-file` (default `public/version.txt`): path to overwrite with the new tag.
- `next-tag` (required): tag value to write.

**Outputs**
- `new-tag`: tag that was written.

**Example**
```yaml
jobs:
  write-version:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-update-version-file.yml@vX.Y.Z
    with:
      target-branch: master
      version-file: public/version.txt
      next-tag: ${{ needs.calculate-tag.outputs.new-tag }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
```

### `workflow-create-release.yml`
Publishes a GitHub Release for a given tag. If `merged-prs` is omitted or empty, it collects merged PRs between `previous-tag` and `tag-name`.

**Inputs**
- `tag-name` (required): tag to publish.
- `previous-tag` (default `v0.0.0`): used for changelog comparison.
- `merged-prs` (default `[]`): JSON array of merged PR metadata.

**Example**
```yaml
jobs:
  create-release:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-create-release.yml@vX.Y.Z
    with:
      tag-name: ${{ needs.create-tag.outputs.new-tag }}
      previous-tag: ${{ needs.create-tag.outputs.previous-tag }}
      merged-prs: ${{ needs.merge.outputs.merged-prs }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
```

### `workflow-deploy-to-s3.yml`
Syncs a directory to an S3 bucket with optional Cloudflare cache purge and SES notification.

**Inputs**
- `bucket` (required): destination S3 bucket (without `s3://`).
- `source` (default `public`): local directory to sync.
- `aws-region` (default `us-west-2`): region for S3/SES calls.
- `delete-extra-files` (default `true`): remove objects not present locally.
- `target-branch` (default `master`): branch to check out before syncing.
- `ref` (optional): explicit git ref (commit SHA/tag/branch) to deploy; overrides `target-branch` when set.
- `cloudflare-zone-id` (optional): zone to purge after deploy.
- `purge-cloudflare` (default `true`): whether to purge the zone when credentials are provided.
- `email-subject` (optional): SES email subject (defaults to the bucket name).
- `email-body` (optional): SES email body (defaults to an auto-generated message).

**Secrets**
- `aws_access_key_id` (required)
- `aws_secret_access_key` (required)
- `aws_session_token` (optional)
- `cloudflare_api_token` (optional)
- `email_from` (optional)
- `email_to` (optional)

**Outputs**
- `deployed`: `true` when the S3 sync completes.

**Example**
```yaml
jobs:
  deploy-static:
    needs: tests
    uses: vinitu-net/github-workflows/.github/workflows/workflow-deploy-to-s3.yml@vX.Y.Z
    with:
      ref: ${{ github.sha }}
      bucket: www.example.com
      source: public
      aws-region: us-west-2
      delete-extra-files: true
      cloudflare-zone-id: ${{ secrets.CLOUDFLARE_ZONE_ID }}
      email-subject: "Site deployed"
    secrets:
      aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      email_from: ${{ secrets.EMAIL_FROM }}
      email_to: ${{ secrets.EMAIL_TO }}
```

### End-to-end usage in a caller repo
```yaml
jobs:
  determine-version:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-determine-version-bump.yml@vX.Y.Z
    with:
      target-branch: main
      major-branch-prefixes: major/
      minor-branch-prefixes: |
        feature/
        features/
      patch-branch-prefixes: |
        fix/
        fixes/
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}

  merge:
    needs: determine-version
    uses: vinitu-net/github-workflows/.github/workflows/workflow-merge-pull-requests.yml@vX.Y.Z
    with:
      target-branch: main
      pull-requests: ${{ toJson(github.event.pull_request) }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}

  calculate-tag:
    needs: [determine-version, merge]
    if: ${{ needs.merge.outputs.merged == 'true' }}
    uses: vinitu-net/github-workflows/.github/workflows/workflow-compute-next-tag.yml@vX.Y.Z
    with:
      target-branch: main
      version-bump: ${{ needs.determine-version.outputs.version-bump }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}

  create-tag:
    needs: [determine-version, merge, calculate-tag]
    if: ${{ needs.merge.outputs.merged == 'true' }}
    uses: vinitu-net/github-workflows/.github/workflows/workflow-create-tag.yml@vX.Y.Z
    with:
      target-branch: main
      next-tag: ${{ needs.calculate-tag.outputs.new-tag }}
      previous-tag: ${{ needs.calculate-tag.outputs.previous-tag }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}

  create-release:
    needs: [merge, create-tag]
    if: ${{ needs.merge.outputs.merged == 'true' }}
    uses: vinitu-net/github-workflows/.github/workflows/workflow-create-release.yml@vX.Y.Z
    with:
      tag-name: ${{ needs.create-tag.outputs.new-tag }}
      previous-tag: ${{ needs.create-tag.outputs.previous-tag }}
      merged-prs: ${{ needs.merge.outputs.merged-prs }}
    secrets:
      gh_token: ${{ secrets.GITHUB_TOKEN }}
```

## Repo-local workflows (used only in this repo)
- `.github/workflows/auto-merge.yml` — PR CI for this repo: runs `actionlint` on PRs to `main` and auto-merges same-repo PRs after checks pass.
- `.github/workflows/release.yml` — release pipeline for this repo, triggered after `Auto Merge PRs`; determines bump, tags, and publishes a release.
