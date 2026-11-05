# AGENTS

Reusable GitHub Actions workflows for `vinitu` repositories. `README.md` is the reference for
callers; keep it in step with the workflow inputs and outputs.

- `workflow-*.yml` are reusable (`on: workflow_call`), called as
  `uses: vinitu/github-workflows/.github/workflows/workflow-<name>.yml@vX.Y.Z`. The repository
  moved from `vinitu-net`; never write that name in a `uses:` line.
- `ci-pr.yml` and `ci-main.yml` are repo-local. `ci-pr.yml` runs `actionlint` and merges;
  `ci-main.yml` runs on the push to `main` and tags and releases. Add more checks to both.
- Only merge via PRs; forks are not auto-merged. After a merge to `main` the pipeline tags and
  releases — depend on those tags (`@vX.Y.Z`), not `@main`.
- The branch prefix decides the bump: `major/` for a breaking change to an input, an output or a
  default; `feature/` or `features/` for a new input or workflow; anything else is a patch.
- When workflow structure or inputs change, update the README examples.
- Run `actionlint` before you push. It also runs `shellcheck` over every `run:` block when
  `shellcheck` is on `PATH`.
- Keep the guards. Each one exists because its absence broke a release:
  - tag and merge stay in separate workflows. `gh pr merge --auto` returns before GitHub merges,
    so a tag in the pull request run lands on the previous commit;
  - `workflow-merge-pull-requests.yml` reports `merged=true` only after a real merge. Its direct
    merge fallback (for repositories without auto-merge) waits for no check, so it must sit in
    `needs:` of the checks;
  - merges and tags in this repo use the `TOKEN` secret: events made by `GITHUB_TOKEN` start no
    workflows, so `ci-main.yml` would never run after the merge;
  - the tag filter is anchored to `^vX.Y.Z$` (pre-release tags break the arithmetic), and an API
    failure while reading tags fails the job;
  - `workflow-create-tag.yml` tags the exact `sha` it gets, never the branch head, and skips a
    commit that already has a tag or says `[skip release]`;
  - `ci-main.yml` keeps `cancel-in-progress: false`, so no run stops between tag and release.
- Do not pipe `gh` into `head` under `set -o pipefail`: `head` can kill `gh` with SIGPIPE.
  Capture the output first.
