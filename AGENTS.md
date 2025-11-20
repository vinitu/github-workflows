# AGENTS

- Only merge via PRs; forks are not auto-merged.
- After merging to `main` the pipeline auto-tags and releases — depend on those tags (`@vX.Y.Z`), not `@main`.
- Every PR runs `actionlint` (see `.github/workflows/auto-merge.yml`); add more checks there if needed.
- When workflow structure or inputs change, update the README example and bump the version.
- `auto-merge.yml` and `release.yml` are repo-local; other workflows are meant as shared reusable workflows (`uses: vinitu/github-workflows/...`).
- `merge-pull-requests.yml` accepts `merge-method` (`merge`/`squash`/`rebase`) to control how PRs are merged.
