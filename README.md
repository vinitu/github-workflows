# Shared GitHub Workflows

Reusable workflows to plug into other repos via `uses: vinitu/github-workflows/.github/workflows/<workflow>.yml@<tag>`.

- Always pin to a release tag (auto-created on every merge to `main`); avoid `@main`.
- Each workflow accepts `gh_token`; if omitted it falls back to the built-in `GITHUB_TOKEN`.
- The repository moved from the `vinitu-net` organization to the `vinitu` account. Use `vinitu/`
  in `uses:`. `vinitu-net` is now a separate, empty organization.

## The release rail

Merging and tagging live in two different workflows:

```
pull_request ──► checks ──► workflow-merge-pull-requests (needs: the checks)

push to main ──► workflow-determine-version-bump (PR looked up from the commit)
                 → workflow-compute-next-tag
                 → workflow-create-tag (on github.sha)
                 → workflow-create-release (only when created == 'true')
```

Do not tag in the pull request run. `gh pr merge --auto` returns as soon as auto-merge is
enabled, before GitHub merges, so a tag created in the same run lands on the previous commit
of `main`.

Two token rules:

- **Merge with a personal access token** (`secrets.TOKEN`). A merge made with `GITHUB_TOKEN`
  does not start the `on: push` workflow on `main`, so nothing would be tagged.
- **Create the tag with a personal access token** when an `on: push: tags` workflow must start
  from it (for example a deploy). A tag created with `GITHUB_TOKEN` starts nothing.

## Shared workflows (reusable)

### `workflow-determine-version-bump.yml`
Detects which semver segment to bump from the head branch of a pull request into a branch.

Where the pull request comes from:
- `pull-requests` input when given (e.g. `toJson(github.event.pull_request)`);
- on `pull_request`: the event;
- on `push` and `workflow_dispatch`: the pull request that merged `github.sha`, looked up with
  `repos/{repo}/commits/{sha}/pulls`. Open pull requests that only contain the commit do not
  count. A direct push has no pull request and gets `default`;
- on `workflow_run`: the event's pull requests.

An API failure fails the job; it does not fall back to `default`.

**Inputs**
- `target-branch` (default `main`): only consider PRs targeting this branch.
- `major-branch-prefixes`, `minor-branch-prefixes`, `patch-branch-prefixes`: comma or newline separated branch prefixes to map to a bump.
- `default` (default `patch`): fallback bump when no prefixes match.
- `pull-requests`: PR payload (object or array).

**Outputs**
- `version-bump`: `major`, `minor`, or `patch`.
- `matching_pr`: `true` if a PR against `target-branch` was found.

**Branch prefix rules**
- Major bump examples: `major/`.
- Minor bump examples: `feature/`, `features/`.
- Patch bump examples: `fix/`, `fixes/`.
- If none match, `default` is applied (by default, `patch`).

**Example** (on push to `main`)
```yaml
jobs:
  determine-version:
    permissions:
      contents: read
      pull-requests: read
    uses: vinitu/github-workflows/.github/workflows/workflow-determine-version-bump.yml@vX.Y.Z
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
```

### `workflow-merge-pull-requests.yml`
Merges same-repo PRs into a target branch. Skips forks, draft PRs, and branches starting with
`wip`.

It first runs `gh pr merge --auto`. When the repository does not allow auto-merge
("Allow auto-merge" is off in the settings), it merges directly instead. **The direct merge does
not wait for any check**, so call this workflow only from a job that `needs:` your checks in the
same workflow run.

After the merge call it reads the pull request again:
- `MERGED` → counted in `merged` and `merged-prs`;
- still open with auto-merge on → counted in `auto-merge-enabled` only. GitHub merges it later,
  after this run ends.

Never tag or release from `merged` in the same run. Tag in the `on: push` workflow of the target
branch.

**Inputs**
- `target-branch` (default `main`): required base branch to merge into.
- `merge-method` (default `merge`): `merge`, `squash`, or `rebase`.
- `pull-requests`: PR payload for `workflow_call` (e.g., `toJson(github.event.pull_request)`).

**Outputs**
- `merged`: `true` only if at least one PR is really merged by the end of the run.
- `auto-merge-enabled`: `true` if at least one PR is left open with auto-merge enabled.
- `merged-prs`: JSON array with `number`, `title`, `author`, `head` of the really merged PRs.

**Example**
```yaml
jobs:
  merge:
    needs: tests
    permissions:
      contents: write
      pull-requests: write
    uses: vinitu/github-workflows/.github/workflows/workflow-merge-pull-requests.yml@vX.Y.Z
    with:
      target-branch: main
      merge-method: squash
      pull-requests: ${{ toJson(github.event.pull_request) }}
    secrets:
      gh_token: ${{ secrets.TOKEN }}
```

### `workflow-compute-next-tag.yml`
Calculates the next semver tag from the bump and the newest tag that is exactly `vX.Y.Z`.
Pre-release tags such as `v1.7.0-alpha` are ignored. No such tag at all starts from `v0.0.0`.
Tags are read through the GitHub API; an API failure fails the job instead of looking like "no
tags".

**Inputs**
- `version-bump` (required): `major`, `minor`, or `patch`.
- `target-branch`: deprecated and ignored; kept so existing callers do not break.

**Outputs**
- `new-tag`: computed next tag (e.g., `v1.2.3`).
- `previous-tag`: newest existing `vX.Y.Z` tag (or `v0.0.0` if none).

**Example**
```yaml
jobs:
  calculate-tag:
    needs: determine-version
    uses: vinitu/github-workflows/.github/workflows/workflow-compute-next-tag.yml@vX.Y.Z
    with:
      version-bump: ${{ needs.determine-version.outputs.version-bump }}
```

### `workflow-create-tag.yml`
Creates an annotated tag on one exact commit (`sha`) through the GitHub API, with `gh_token`.
No version calculation inside this workflow.

It skips, with `created=false`, when:
- the commit message contains `skip-marker` (default `[skip release]`, case-insensitive);
- the commit already has any tag, so a hand-pushed tag keeps working and a commit is never
  tagged twice.

It fails when `next-tag` already exists on another commit: two runs computed the same tag.

**Inputs**
- `sha` (required): full commit SHA to tag. On push to the release branch pass `github.sha`;
  after `workflow-update-version-file.yml` pass its `sha` output.
- `next-tag` (required): tag to create (e.g., `v1.2.3`).
- `previous-tag` (optional): passed through to the outputs.
- `skip-marker` (default `[skip release]`): empty turns the check off.

**Outputs**
- `created`: `true` when the tag was created, `false` when it was skipped.
- `new-tag`: tag that was created, empty when skipped.
- `previous-tag`: the `previous-tag` input.

**Example**
```yaml
jobs:
  create-tag:
    needs: calculate-tag
    permissions:
      contents: write
    uses: vinitu/github-workflows/.github/workflows/workflow-create-tag.yml@vX.Y.Z
    with:
      sha: ${{ github.sha }}
      next-tag: ${{ needs.calculate-tag.outputs.new-tag }}
      previous-tag: ${{ needs.calculate-tag.outputs.previous-tag }}
    secrets:
      gh_token: ${{ secrets.TOKEN }}
```

### `workflow-update-version-file.yml`
Writes the provided tag into a version file on a target branch and pushes the commit.

**Inputs**
- `target-branch` (default `master`): branch to check out before writing the version file.
- `version-file` (default `public/version.txt`): path to overwrite with the new tag; pass multiple paths one per line to update them all in a single commit.
- `next-tag` (required): tag value to write.

**Outputs**
- `new-tag`: tag that was written.
- `sha`: commit that holds the version file (the pushed commit, or the branch head when the file
  was already up to date). Pass it as `sha` to `workflow-create-tag.yml`.

**Example**
```yaml
jobs:
  write-version:
    uses: vinitu/github-workflows/.github/workflows/workflow-update-version-file.yml@vX.Y.Z
    with:
      target-branch: master
      version-file: public/version.txt
      next-tag: ${{ needs.calculate-tag.outputs.new-tag }}
    secrets:
      gh_token: ${{ secrets.TOKEN }}
```

### `workflow-create-release.yml`
Publishes a GitHub Release for a given tag. If `merged-prs` is omitted or empty, it collects merged PRs between `previous-tag` and `tag-name`.
The job inherits the caller's token: grant `contents: write`, and `pull-requests: read` for the
notes to list the pull requests. Without the second one the notes say "No pull requests were
merged in this release."

**Inputs**
- `tag-name` (required): tag to publish.
- `previous-tag` (default `v0.0.0`): used for changelog comparison.
- `merged-prs` (default `[]`): JSON array of merged PR metadata.

**Example**
```yaml
jobs:
  create-release:
    needs: create-tag
    if: ${{ needs.create-tag.outputs.created == 'true' }}
    permissions:
      contents: write
      pull-requests: read
    uses: vinitu/github-workflows/.github/workflows/workflow-create-release.yml@vX.Y.Z
    with:
      tag-name: ${{ needs.create-tag.outputs.new-tag }}
      previous-tag: ${{ needs.create-tag.outputs.previous-tag }}
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
    uses: vinitu/github-workflows/.github/workflows/workflow-deploy-to-s3.yml@vX.Y.Z
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

## End-to-end usage in a caller repo

`.github/workflows/ci-pr.yml` — checks, then merge:

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: make test

  merge:
    needs: tests
    permissions:
      contents: write
      pull-requests: write
    uses: vinitu/github-workflows/.github/workflows/workflow-merge-pull-requests.yml@vX.Y.Z
    with:
      target-branch: main
      merge-method: squash
      pull-requests: ${{ toJson(github.event.pull_request) }}
    secrets:
      gh_token: ${{ secrets.TOKEN }}
```

`.github/workflows/ci-main.yml` — tag and release on the push the merge produces:

```yaml
on:
  push:
    branches: [main]

permissions:
  contents: read

# Never cancel between the tag and the release.
concurrency:
  group: ci-main
  cancel-in-progress: false

jobs:
  determine-version:
    permissions:
      contents: read
      pull-requests: read
    uses: vinitu/github-workflows/.github/workflows/workflow-determine-version-bump.yml@vX.Y.Z
    with:
      target-branch: main
      major-branch-prefixes: major/
      minor-branch-prefixes: |
        feature/
        features/

  calculate-tag:
    needs: determine-version
    uses: vinitu/github-workflows/.github/workflows/workflow-compute-next-tag.yml@vX.Y.Z
    with:
      version-bump: ${{ needs.determine-version.outputs.version-bump }}

  create-tag:
    needs: calculate-tag
    permissions:
      contents: write
    uses: vinitu/github-workflows/.github/workflows/workflow-create-tag.yml@vX.Y.Z
    with:
      sha: ${{ github.sha }}
      next-tag: ${{ needs.calculate-tag.outputs.new-tag }}
      previous-tag: ${{ needs.calculate-tag.outputs.previous-tag }}
    secrets:
      gh_token: ${{ secrets.TOKEN }}

  create-release:
    needs: create-tag
    if: ${{ needs.create-tag.outputs.created == 'true' }}
    permissions:
      contents: write
      pull-requests: read
    uses: vinitu/github-workflows/.github/workflows/workflow-create-release.yml@vX.Y.Z
    with:
      tag-name: ${{ needs.create-tag.outputs.new-tag }}
      previous-tag: ${{ needs.create-tag.outputs.previous-tag }}
```

With `workflow-update-version-file.yml` between `calculate-tag` and `create-tag`, pass
`sha: ${{ needs.write-version.outputs.sha }}` instead of `github.sha`, so the tag holds the
version file.

## Upgrading from v0.1.x

- `workflow-create-tag.yml`: `sha` is new and required; `target-branch` is gone; `previous-tag`
  is optional. Run the release job on `needs.<tag job>.outputs.created == 'true'`, not on
  `result == 'success'`: a skipped tag succeeds with `created=false` and an empty `new-tag`.
- `workflow-merge-pull-requests.yml`: `merged` is no longer `true` when auto-merge was only
  enabled. Read `auto-merge-enabled` for that case. Move any tag or release out of the pull
  request run.
- `workflow-compute-next-tag.yml`: `target-branch` is ignored; tags are read from the API.

## Repo-local workflows (used only in this repo)
- `.github/workflows/ci-pr.yml` — runs `actionlint` on PRs to `main`, then merges same-repo PRs
  with `workflow-merge-pull-requests.yml` and the `TOKEN` secret. It never tags.
- `.github/workflows/ci-main.yml` — on push to `main`: `actionlint`, then bump from the merged
  PR's branch, tag `github.sha`, and publish a release. `cancel-in-progress` is `false`, so a run
  is never killed between the tag and the release. `workflow_dispatch` re-runs it for the head of
  `main`, for a merge that did not start it.

Run `actionlint` locally before you push:

```bash
bash <(curl -sSfL https://raw.githubusercontent.com/rhysd/actionlint/v1.7.12/scripts/download-actionlint.bash) 1.7.12 .
./actionlint
```
