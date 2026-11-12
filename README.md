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

## Licence

The action is released under the MIT License (see `LICENSE`). The bundled third-party
packages keep their own licences, reproduced in `THIRD_PARTY_NOTICES.md`. The MergeDelta
backend and dashboard are separate hosted services and are not part of this repository.

## Releases

Each release is a commit tagged `v1.x.y`. The release tree (`scan/`, `publish/`, this README,
and a LICENSE when the source repo carries one) is produced by `scripts/release-action.mjs` in
the private monorepo, which rebuilds both actions and checks their bundle-freshness before
writing the tree. To update a customer's workflow, bump the SHA to a newer release's commit.
