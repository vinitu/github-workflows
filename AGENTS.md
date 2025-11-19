# AGENTS

- Only merge via PRs; forks are not auto-merged.
- After merging to `main` the pipeline auto-tags and releases — depend on those tags (`@vX.Y.Z`), not `@main`.
- Every PR runs `actionlint` (see `.github/workflows/auto-merge.yml`); add more checks there if needed.
- When workflow structure or inputs change, update the README example and bump the version.