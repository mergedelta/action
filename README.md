# MergeDelta action

This repository holds the **built** GitHub Actions for MergeDelta:

- `scan/` — the unprivileged scan action (runs on pull requests; measures and uploads one artifact).
- `publish/` — the privileged publish action (runs on `workflow_run`; validates the artifact, exchanges an OIDC token, diffs against the baseline, and posts the MergeDelta comment and check).

## Built from the private monorepo

These actions are built from the private MergeDelta monorepo and released here so customer
repositories can pin them by **full commit SHA** (the safest way to reference a GitHub Action).
The `dist/index.js` in each directory is the complete bundled action; no build step is needed
to use them.

## Pin by SHA

In your workflow:

```yaml
- uses: <this-repo>/scan@<full-40-char-commit-sha>   # pin to a release commit
- uses: <this-repo>/publish@<full-40-char-commit-sha>
```

Never pin to a tag (`@v1`) or a branch (`@main`): a tag can move, and a branch can be force-pushed.
A full commit SHA is immutable, so a pinned workflow runs exactly the code that was reviewed at
release time. MergeDelta's setup page fills the SHA in for you when you generate the workflow files.

## Configuration: `perf.config.json`

Both actions read `perf.config.json` at the repository root (change the path with each action's
`config` input). Unknown keys are errors. An invalid config fails the scan, and the pull request
gets a neutral "MergeDelta scan did not complete" check that links to the scan log, where the
exact error is printed.

```json
{
  "version": 1,
  "pages": [{ "path": "/", "pageGroup": "home" }],
  "runs": 3,
  "formFactor": "mobile",
  "localBuild": { "install": "npm ci", "build": "npm run build", "serve": { "staticDir": "dist" } },
  "engines": { "i18n": { "enabled": false }, "totalBudgetMinutes": 12 },
  "blocking": { "TBT": { "minIncrease": 300 } }
}
```

| Key | Default | What it does |
|---|---|---|
| `version` | required | Always `1`. |
| `pages` | required | 1–20 pages to measure. `path` starts with `/`; `pageGroup` names the page in the comment and in budgets. |
| `runs` | `3` | Lighthouse runs per side (1–9). More runs mean less noise and a longer scan. |
| `formFactor` | `"mobile"` | `"mobile"` or `"desktop"`. |
| `localBuild` | none | Strict merge-base mode: the scan builds the merge-base and the PR head itself and serves both on localhost, so both sides are measured in the same job. `install` and `build` are optional commands; `serve` is either `{ "staticDir": "dist" }` or `{ "start": "npm run preview -- --port $PORT", "readyPath": "/" }`. Without it, the PR preview is measured against production. |
| `engines` | all on | Turn a quality area off with `{ "<area>": { "enabled": false } }`. Areas: `a11y`, `seo`, `security`, `links`, `bundle`, `i18n`. `totalBudgetMinutes` (3–30, default 12) caps the time these areas share. Whether an area may block is decided by MergeDelta, not by this file. |
| `blocking` | none | Per-repository thresholds for the metrics that can fail the check (see below). |
| `budgetFile` | none | Path to a budgets file (for example `perf.budgets.json`). Over-budget metrics are reported in the comment; budgets never fail the check. |
| `lessonBaseUrl` | MergeDelta lessons | Base URL for the "Learn why" links. |
| `failOn`, `comment` | — | Read only by the deprecated single-workflow `pr` mode; ignored by `scan` + `publish`. |

### What fails the check

A confirmed **TBT** (Total Blocking Time) or **CLS** (Cumulative Layout Shift) regression fails the
check. "Confirmed" means every head run is worse than the base median by more than the
run-to-run noise. FCP, Speed Index and LCP regressions are reported as advisory, and the other
quality areas are advisory while they are in Beta.

### `blocking`: require a larger regression before the check fails

```json
"blocking": {
  "TBT": { "minIncrease": 300 },
  "CLS": { "minIncrease": 0.05 }
}
```

- A confirmed TBT or CLS regression fails the check only when the median got worse by at least
  `minIncrease`: milliseconds for TBT (0–10000), unitless for CLS (0–1). Smaller confirmed
  regressions are still reported, as advisory.
- Each entry is an object: `"TBT": 300` is invalid. Only `TBT` and `CLS` are accepted.
- The value is read from the config **at the merge-base**, never from the pull request, so a PR
  cannot loosen its own gate. A PR that edits `perf.config.json` or the MergeDelta workflows is
  an *Unverified run*: its results are shown, but it can neither pass nor fail the check.
- A `minIncrease` below the noise threshold has no effect: the regression must be confirmed first,
  so this setting can only let fewer regressions fail the check.

## Licence

The action is released under the MIT License (see `LICENSE`). The bundled third-party
packages keep their own licences, reproduced in `THIRD_PARTY_NOTICES.md`. The MergeDelta
backend and dashboard are separate hosted services and are not part of this repository.

## Releases

Each release is a commit tagged `v1.x.y`. The release tree (`scan/`, `publish/`, this README,
and a LICENSE when the source repo carries one) is produced by `scripts/release-action.mjs` in
the private monorepo, which rebuilds both actions and checks their bundle-freshness before
writing the tree. To update a customer's workflow, bump the SHA to a newer release's commit.
