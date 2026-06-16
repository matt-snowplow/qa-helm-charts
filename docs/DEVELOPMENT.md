# Avalanche chart staging playbook

This repo (`qa-helm-charts`) is a **QA-811 staging repo**, not the source of
truth. The avalanche chart's canonical home is the official
**`snowplow-devops/helm-charts`** repo (`charts/avalanche/`). Use this repo to
publish and validate a chart change **before** it merges to helm-charts, so DS4
can pull it without waiting on the SRE-gated merge.

- Source of truth: `snowplow-devops/helm-charts` → `charts/avalanche/`
  (published at `https://snowplow-devops.github.io/helm-charts`)
- This staging repo publishes to: `https://matt-snowplow.github.io/qa-helm-charts`
  (GitHub Pages, via `.github/workflows/release.yml` → chart-releaser)
- Consumed by: snowman (only when its `*_helm_repository` var is overridden to
  this repo) and avalanche's k8s integration tests (via `AVALANCHE_HELM_CHART_DIR`).

> The in-repo `helm/` copy in the avalanche repo and its `scripts/publish-chart.sh`
> sync were removed in **QA-1109**. Chart changes now originate as PRs against
> `snowplow-devops/helm-charts`; this repo stages them for DS4 validation in the
> meantime.

## The staging loop

```
make a chart change
   │
   ├─ (canonical) open a PR against snowplow-devops/helm-charts ─────────► merge
   │
   └─ (stage for DS4 now) copy charts/avalanche here, bump Chart.yaml,
        push → publishes → point snowman/DS4 at this repo to validate
                                   │
                                   └─ once the helm-charts PR merges,
                                      re-sync this repo to match and
                                      revert snowman to the official repo (QA-1086)
```

### 1. Stage the change here

Copy the changed `charts/avalanche/` from your helm-charts PR branch into this
repo, bump `version:` in `charts/avalanche/Chart.yaml` (the bump is what triggers
publication), commit, and push to `main`. chart-releaser packages it, creates a
GitHub Release, and updates the Pages `index.yaml`.

Confirm it published:

```bash
curl -sL https://matt-snowplow.github.io/qa-helm-charts/index.yaml \
  | grep -A2 'name: avalanche' | grep version
```

### 2. Validate in DS4 (snowman)

snowman defaults to the official repo, so override per-run to point at this
staging repo:

```bash
AVALANCHE_HELM_REPOSITORY=https://matt-snowplow.github.io/qa-helm-charts
AVALANCHE_CHART_VERSION=<staged version>
```

### 3. Test locally / in sandbox without publishing

avalanche's k8s tests and Makefile helm targets resolve the chart from a path,
so you can exercise an unpublished working tree directly:

```bash
AVALANCHE_HELM_CHART_DIR=/path/to/qa-helm-charts/charts/avalanche \
  go test -tags="integration kubernetes" ./pkg/controller -run TestController... -v
```

### 4. After the helm-charts PR merges

Re-sync `charts/avalanche/` here to match the merged official chart, and revert
the snowman `*_helm_repository` defaults to the official repo (QA-1086). Keeping
this repo a faithful mirror (plus any in-flight staged change) is what prevents
the same-version-different-content divergence (the QA-782 lesson).

## Chart dependencies

The avalanche chart depends on `dockerconfigjson` + `cloudserviceaccount` from
the SRE `snowplow-devops` Helm repo. The release workflow runs
`helm repo add snowplow-devops https://snowplow-devops.github.io/helm-charts`
before chart-releaser; without it packaging fails with
`no repository definition for …`. Locally, run the same `helm repo add` once (or
`helm dependency update charts/avalanche`) before templating.

## One-time setup gotchas (kept for reference)

- **`gh-pages` must be seeded once.** chart-releaser does not bootstrap it
  (`fatal: invalid reference: origin/gh-pages`). Created once via an orphan
  branch with an empty `index.yaml`; Pages then serves from `gh-pages`.
- **The first chart in a commit is skipped.** chart-releaser only publishes
  charts whose files changed since the last release, so the very first add
  reports "Nothing to do" — bump `Chart.yaml` in a follow-up commit to trigger
  publication.
