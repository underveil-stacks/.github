# Org templates

Copy-paste starting points for files that GitHub cannot inherit org-wide and
that therefore must be committed into each repo.

## `dependabot.yml` / `dependabot-spa.yml`

Dependabot configuration is per-repo only — there is no org-level inheritance.
Copy the right template to `.github/dependabot.yml` in each service repo:

- **`dependabot.yml`** — Go service repos (gomod + github-actions, weekly).
- **`dependabot-spa.yml`** — repos that also ship an SPA / Node front-end
  (adds the npm ecosystem). `manuals-webui` is the current example.

Org-wide Dependabot **alerts** and **security updates** are a separate,
org-level setting (Settings → Code security) that only an org owner can enable;
these files control the *version-update* PRs, not the alert feed.

## Release pipeline

Service repos do not need a template file for releases — they call the org
reusable workflow directly:

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ['v*']
permissions:
  contents: write
jobs:
  release:
    uses: underveil-stacks/.github/.github/workflows/release-go-service.yml@main
    with:
      binaries: '[{"name":"my-api","main":"./cmd/server"}]'
```

See [`release-go-service.yml`](../.github/workflows/release-go-service.yml) for
all inputs (targets, multi-binary, CGO, GoReleaser mode, extra assets).
