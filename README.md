# qa-helm-charts

QA-owned Helm chart repository for Avalanche load-testing chart versions, used to
test chart changes in DS4 without the SRE-gated `snowplow-devops/helm-charts` merge
(QA-811). Charts under `charts/` are published to GitHub Pages by `.github/workflows/release.yml`.

Repo URL: `https://matt-snowplow.github.io/qa-helm-charts`

## Publishing a chart update

1. Bump `version:` in the chart's `Chart.yaml` in the **avalanche** repo (e.g. `0.6.3` → `0.6.4`).
2. Sync it into `charts/<chart>` here. From the avalanche repo:
   ```
   HELM_CHARTS_DIR=/path/to/qa-helm-charts ./scripts/publish-chart.sh
   ```
   > NB: `publish-chart.sh` currently has `CHART_DIR=.../helm/avalanche`, but the chart
   > is at `helm/` — point it at `helm/` (or fix that line). Tracked under QA-811.
3. Commit + push to `main`. The push triggers `.github/workflows/release.yml`
   (chart-releaser), which packages the chart, creates a GitHub Release, and updates
   the Pages index.
4. Point DS4 at it (snowman QA-811 vars):
   ```
   AVALANCHE_HELM_REPOSITORY=https://matt-snowplow.github.io/qa-helm-charts
   AVALANCHE_CHART_VERSION=<new version>
   ```

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
