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
Creates and pushes the next semver tag based on the provided bump.

**Inputs**
- `target-branch` (default `main`): branch to check out before tagging.
- `version-bump` (required): `major`, `minor`, or `patch`.

**Outputs**
- `new-tag`: tag that was created (e.g., `v1.2.3`).
- `previous-tag`: latest existing tag before the bump.

**Example**
```yaml
jobs:
  create-tag:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-create-tag.yml@vX.Y.Z
    with:
      target-branch: main
      version-bump: ${{ needs.determine-version.outputs.version-bump }}
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

  create-tag:
    needs: [determine-version, merge]
    if: ${{ needs.merge.outputs.merged == 'true' }}
    uses: vinitu-net/github-workflows/.github/workflows/workflow-create-tag.yml@vX.Y.Z
    with:
      target-branch: main
      version-bump: ${{ needs.determine-version.outputs.version-bump }}
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
