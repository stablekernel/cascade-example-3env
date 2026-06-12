# cascade-example-3env

A worked example of a three-environment release pipeline (`test` -> `staging`
-> `prod`) driven by [cascade](https://github.com/stablekernel/cascade).

cascade reads a single manifest and generates the GitHub Actions workflows that
orchestrate builds, walk a promotion ladder across environments, and manage the
GitHub release lifecycle (draft -> prerelease -> published).

![scenario-suite](https://github.com/stablekernel/cascade-example-3env/actions/workflows/scenario-suite.yaml/badge.svg)

## What this example shows

This repo focuses on cascade's **inline callback** surface: build and deploy
steps written directly in the manifest as `run:` commands, instead of pointing
at separate reusable workflow files. On top of that it exercises the richer
per-callback knobs cascade supports:

- inline `run:` / `shell:` build and deploy callbacks
- a pre-build `validate` gate that aborts the run on failure
- per-callback `secrets`, OIDC `permissions` (`id-token: write`), `runs_on`,
  `concurrency`, `timeout_minutes`, and `retries`
- build ordering with `depends_on`
- `auto_commits` so generated state lands back on trunk

## Layout

- `.github/manifest.yaml` - the cascade manifest (`config:` plus the `state:`
  block cascade owns). Three environments, two inline builds (`base` -> `app`),
  one inline deploy, and a validate gate.
- `.github/workflows/orchestrate.yaml` - generated. Runs on pushes to `main`,
  executes validate then the inline builds, and cuts the draft release.
- `.github/workflows/promote.yaml` - generated. Promotes a build from one
  environment to the next on demand.
- `.github/workflows/cascade-hotfix.yaml` - generated. Hotfix integration lane.
- `.github/workflows/scenario-suite.yaml` - the end-to-end check (below).

The generated workflows are produced by `cascade generate-workflow` and are not
hand-edited; regenerate them after changing the manifest.

## Promotion ladder

| Step | test | staging | prod | GitHub release |
|------|------|---------|------|----------------|
| commit to trunk | rc | - | - | draft rc |
| promote test -> staging | rc | rc | - | prerelease rc |
| promote staging -> prod | rc | rc | published | published vX.Y.Z |

`staging` is the prerelease environment (last-but-one), so promoting into it
flips the GitHub release to a prerelease. Crossing into `prod` is the publish
boundary: the release is published and the release-candidate tags are cleaned.

Promote a step by hand from the Actions tab (run `promote`, pick a `mode` such
as `test-to-staging` or `staging-to-prod`).

## scenario-suite

`scenario-suite.yaml` is a self-checking driver. It resets the repo, commits to
trunk, lets orchestrate run, then walks the ladder one step at a time. After
each step it reads back the manifest state for all three environments plus the
git tags and GitHub releases and asserts the expected shape (versions present,
prerelease/published flags, tag set) rather than any fixed commit SHA. Polls are
bounded and a timeout emits a workflow error; a run summary is written to the
job summary.

It runs on demand, nightly, and when cascade asks the repo to re-validate after
a new CLI release. It needs a `CASCADE_STATE_TOKEN` repository secret with
permission to write trunk state and dispatch workflows.
