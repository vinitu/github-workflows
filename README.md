# Shared GitHub Workflows

Reusable workflows to plug into other repos via `uses: vinitu-net/github-workflows/.github/workflows/<workflow>.yml@<tag>`. After every merge to `main` this repo auto-tags and publishes a release, so always depend on a version tag, not `@main`.

## Workflows in this repo

### Repo-local (used only here)
- `.github/workflows/auto-merge.yml` — PR CI for this repo: runs `actionlint` on PRs to `main` and auto-merges same-repo PRs after the check passes.
- `.github/workflows/release.yml` — release pipeline for this repo, triggered after the `Auto Merge PRs` workflow completes successfully. It determines the version bump, creates a tag, and publishes a GitHub Release.

### Reusable (shared) workflows
- `.github/workflows/workflow-determine-version-bump.yml` — picks the semver bump based on branch prefixes (`major/`, `feature*/features*/`, `fix*/fixes*/`) and outputs `version-bump` plus `matching_pr`. When invoked via `workflow_call`, pass the triggering PR payload: `pull-requests: ${{ toJson(github.event.pull_request) }}`.
- `.github/workflows/workflow-merge-pull-requests.yml` — auto-merges PRs into a target branch (skips forks and WIP branches), returns JSON with merged PR metadata. When invoked via `workflow_call`, pass the triggering PR payload: `pull-requests: ${{ toJson(github.event.pull_request) }}`. You can set `merge-method` to `merge` (default), `squash`, or `rebase`.
- `.github/workflows/workflow-create-tag.yml` — bumps the version and creates/pushes a git tag.
- `.github/workflows/workflow-create-release.yml` — creates a GitHub Release for the given tag.

## Example usage (in a caller repo)
```yaml
jobs:
  determine-version:
    uses: vinitu-net/github-workflows/.github/workflows/workflow-determine-version-bump.yml@vX.Y.Z
    with:
      target-branch: main
      major-branch-prefixes: |
        major/
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

Replace `vX.Y.Z` with the latest release tag from this repo.
