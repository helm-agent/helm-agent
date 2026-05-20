# Trigger helmagent.dev Deploy from helm-agent

## 1. Background

helmagent.dev's site build pulls `skills/helm/SKILL.md` from `helm-agent/helm-agent@main` (see `scripts/fetch-skills.mjs`) and releases via the GitHub API (`scripts/fetch-releases.mjs`). Today the site only rebuilds on push to its own repo. We want pushes to `skills/helm/**` and release events in helm-agent to automatically redeploy the site.

## 2. Requirements Summary

- Add a workflow in helm-agent that triggers helmagent.dev's existing `deploy.yml` via `workflow_dispatch`.
- Triggers: release `[edited, released]`, push to `main` with path filter `skills/helm/**`, manual `workflow_dispatch`.
- Use a fine-grained PAT stored as `HELMAGENT_DEPLOY_TOKEN`.
- No changes to helmagent.dev (its workflow already accepts `workflow_dispatch`).

## 3. Acceptance Criteria

1. `.github/workflows/trigger-deploy.yml` exists in helm-agent.
2. Triggers declared: release `[edited, released]` (gated by AC5's `if:`), push to `main` with `paths: ['skills/helm/**']`, `workflow_dispatch`.
3. Job invokes `gh workflow run deploy.yml --repo helm-agent/helmagent.dev --ref main` with `GH_TOKEN: ${{ secrets.HELMAGENT_DEPLOY_TOKEN }}`.
4. Workflow YAML is valid (passes `actionlint`); runs on `ubuntu-latest`; no `actions/checkout` step.
5. Release events are filtered so drafts and prereleases do not deploy.
6. Workflow-level `concurrency` group prevents parallel dispatches.
7. Job has explicit `permissions: {}`.

## 4. Problem Analysis

- **Approach A — `repository_dispatch`**: needs adding a listener to helmagent.dev/deploy.yml. Two files to touch. Rejected — extra surface for no gain.
- **Approach B — `gh workflow run` (chosen)**: hits the existing `workflow_dispatch` entry already in helmagent.dev/deploy.yml. One file, no cross-repo changes. `gh` CLI ships preinstalled on GitHub-hosted runners.
- **Approach C — third-party action (`benc-uk/workflow-dispatch`)**: extra dependency. Rejected — `gh` is built in.

## 5. Decision Log

**1. Trigger mechanism**
- Options: A) `repository_dispatch` · B) `workflow_dispatch` via `gh` · C) third-party action
- Decision: **B)** — single-file change, uses GitHub-shipped tooling.

**2. Release event types**
- Options: A) `[published]` · B) `[edited, released]` · C) `[published, edited]`
- Decision: **B)** — user choice; combined with an `if:` guard so that `edited` only deploys when the release is neither a draft nor a prerelease.

**3. Path filter scope**
- Options: A) `skills/helm/**` · B) `skills/**`
- Decision: **A)** — matches what `fetch-skills.mjs` actually pulls; future skills will need the script updated regardless.

**4. Secret name**
- Options: A) `HELMAGENT_DEPLOY_TOKEN` · B) `DEPLOY_TOKEN` · C) `DEPLOY_PAT`
- Decision: **A)** — intent-revealing.

**5. Checkout step?**
- Options: A) include `actions/checkout` · B) skip
- Decision: **B)** — `gh workflow run` needs no repo contents; `gh` is preinstalled on runners.

**6. Downstream failure visibility**
- Options: A) fire-and-forget · B) `gh run watch` the downstream run
- Decision: **A)** — KISS. The dispatch step exits 0 once accepted; deploy failures surface in helmagent.dev's own Actions tab. Watching adds polling complexity for marginal value.

**7. Concurrency**
- Options: A) none · B) workflow-level `concurrency` group with `cancel-in-progress: false`
- Decision: **B)** — release-publish-then-edit bursts and rapid pushes can otherwise dispatch parallel deploys that race in Cloudflare. Don't cancel — every dispatched deploy fetches the *current* main, so dropping one could miss a change. Note: this serializes *dispatches*, not downstream deploys. helmagent.dev/deploy.yml has no concurrency group, so two dispatched runs can still overlap there. Out of scope for this change.

**8. Job permissions**
- Options: A) inherit default · B) explicit `permissions: {}`
- Decision: **B)** — least-privilege hygiene; the PAT (not `GITHUB_TOKEN`) does all the work, so the job needs no repo permissions.

## 6. Design

Single workflow file. No build, no checkout, no setup — one shell step that calls `gh`.

Notes:
- **No `pull_request` trigger** — deploys only fire from `main`. Adding `pull_request` later would deploy from forks, which is wrong.
- **`gh workflow run` is fire-and-forget** — exits 0 once the dispatch is accepted. Downstream deploy failures appear in helmagent.dev's Actions tab, not here.
- **PAT (`HELMAGENT_DEPLOY_TOKEN`)** must be a fine-grained PAT: resource owner `helm-agent`, repository access **only** `helm-agent/helmagent.dev`, permissions **Actions: Read and write** + **Metadata: Read** (mandatory). Document expiry/rotation owner in repo settings notes.
- **PAT expiry**: fine-grained PATs max out at ~1 year. On expiry, the dispatch step fails with HTTP 401 — visible in helm-agent Actions. Rotation owner: repo admin.

```yaml
name: Trigger helmagent.dev deploy

on:
  release:
    types: [edited, released]
  push:
    branches: [main]
    paths:
      - 'skills/helm/**'
  workflow_dispatch:

concurrency:
  group: trigger-helmagent-deploy
  cancel-in-progress: false

jobs:
  trigger:
    runs-on: ubuntu-latest
    permissions: {}
    # Guard: `edited` fires for any release edit, including drafts and prereleases.
    # Only deploy when the edited/released release is a real, public, non-prerelease.
    if: >-
      github.event_name != 'release' ||
      (github.event.release.draft == false && github.event.release.prerelease == false)
    steps:
      - name: Dispatch deploy in helmagent.dev
        env:
          GH_TOKEN: ${{ secrets.HELMAGENT_DEPLOY_TOKEN }}
        run: gh workflow run deploy.yml --repo helm-agent/helmagent.dev --ref main
```

## 7. Files Changed

- `.github/workflows/trigger-deploy.yml` — new workflow file.

## 8. Verification

1. [AC1] `ls .github/workflows/trigger-deploy.yml` succeeds.
2. [AC4] `actionlint .github/workflows/trigger-deploy.yml` passes; `grep -q actions/checkout .github/workflows/trigger-deploy.yml` returns non-zero (no checkout step).
3. [AC2,3,5,6,7] Visual inspection confirms triggers, command, `if:` guard, `concurrency`, `permissions: {}`.
4. **Manual dry-run (safe)**: from the helm-agent Actions UI, run "Trigger helmagent.dev deploy" via `workflow_dispatch`. Confirm helm-agent run succeeds and a new Deploy run appears in helmagent.dev Actions.
5. **End-to-end push**: edit `skills/helm/SKILL.md`, push to main → confirm chain fires.
6. **End-to-end release**: edit notes on a published (non-prerelease) release in helm-agent → confirm chain fires. Then edit a prerelease → confirm the helm-agent run shows the `trigger` job as **Skipped**, and **no** new Deploy run appears in helmagent.dev.
