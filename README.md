# qa-helm-charts

QA-owned Helm chart repository for Avalanche load-testing chart versions, used to
test chart changes in DS4 without the SRE-gated `snowplow-devops/helm-charts` merge
(QA-811). Charts under `charts/` are published to GitHub Pages by `.github/workflows/release.yml`.

Repo URL: `https://matt-snowplow.github.io/qa-helm-charts`

This repo is a **staging** repo, **not** the source of truth. The avalanche
chart's canonical home is the official **`snowplow-devops/helm-charts`** repo
(`charts/avalanche/`). Use this repo to publish and validate a chart change in
DS4 *before* it merges to helm-charts (QA-811). The avalanche in-repo `helm/`
copy and its `scripts/publish-chart.sh` sync were removed in **QA-1109**.

📖 **Full workflow: [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)** — stage here →
PR to helm-charts → re-sync once merged.

## Staging a chart update (quick reference)

1. Copy the changed `charts/avalanche/` from your `snowplow-devops/helm-charts`
   PR branch into this repo and bump `version:` in `charts/avalanche/Chart.yaml`
   (the bump is what triggers publication).
2. Commit + push to `main`. The push triggers `.github/workflows/release.yml`
   (chart-releaser), which packages the chart, creates a GitHub Release, and updates
   the Pages index.
3. Confirm: `curl -sL https://matt-snowplow.github.io/qa-helm-charts/index.yaml | grep -A2 'name: avalanche' | grep version`.
4. Point DS4 at it (snowman QA-811 override vars):
   ```
   AVALANCHE_HELM_REPOSITORY=https://matt-snowplow.github.io/qa-helm-charts
   AVALANCHE_CHART_VERSION=<staged version>
   ```
5. Once the helm-charts PR merges, re-sync `charts/avalanche/` here to match and
   revert the snowman defaults to the official repo (QA-1086).

To test an **unpublished** working tree without publishing, point consumers at a
local checkout via `AVALANCHE_HELM_CHART_DIR=/path/to/qa-helm-charts/charts/avalanche`
(see [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md)).

## Setup gotchas (one-time, already done — kept for reference)

These bit us when first standing the repo up:

- **Chart-dependency repo must be added before packaging.** The avalanche chart depends
  on `dockerconfigjson` + `cloudserviceaccount` from the snowplow-devops repo, so the
  workflow runs `helm repo add snowplow-devops https://snowplow-devops.github.io/helm-charts`
  before chart-releaser. Without it: `no repository definition for …`.
- **`gh-pages` must be seeded once.** chart-releaser does **not** bootstrap it
  (`fatal: invalid reference: origin/gh-pages`). Created once via an orphan branch with
  an empty `index.yaml`; Pages then serves from `gh-pages`.
- **The first chart in a commit is skipped.** chart-releaser only publishes charts whose
  files changed since the last release, so the very first add reports "Nothing to do" —
  bump the `Chart.yaml` version in a follow-up commit to trigger publication.
