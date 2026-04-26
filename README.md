# ci-templates

Shared GitHub Actions reusable workflows for the org. Consumed via
`uses: <org>/ci-templates/.github/workflows/<name>.yml@<sha>`.

## Workflows

### `secrets-scan.yml`

Scans the consuming repo's full git history with
[gitleaks](https://github.com/gitleaks/gitleaks) and uploads SARIF
findings to the GitHub Code Scanning Security tab.

**Consumer-side usage:**

```yaml
name: check
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    # Daily full-history scan as a safety net for pre-existing leaks
    # and for new gitleaks rules that catch previously-missed patterns.
    - cron: '0 8 * * *'

permissions:
  contents: read
  pull-requests: write    # gitleaks PR comments
  security-events: write  # SARIF → Code Scanning Security tab

jobs:
  secrets-scan:
    uses: <org>/ci-templates/.github/workflows/secrets-scan.yml@<sha>
    secrets: inherit
```

The caller must grant the same permissions the called workflow needs;
GitHub fails the run at startup if any are missing.

To use a self-hosted runner instead of `ubuntu-latest`:

```yaml
  secrets-scan:
    uses: <org>/ci-templates/.github/workflows/secrets-scan.yml@<sha>
    with:
      runs-on: self-hosted
    secrets: inherit
```

**Pre-requisites:**

- Org-level secret `GITLEAKS_LICENSE` (free for orgs — register at
  <https://gitleaks.io>, add as Organization Secret with access scoped
  to the repos that should consume this workflow).

**Behavior:**

- Default runner: `ubuntu-latest`. Override via `with: runs-on: ...`.
- Fetches the consumer repo with `fetch-depth: 0` (full history).
- Does **not** fetch submodules — each repo scans itself by org policy.
- Gitleaks default config; suppress known-OK historical hits via a
  `.gitleaksignore` file in the consumer repo.

## Versioning

Tag releases as `vMAJOR.MINOR.PATCH`. Consumers SHA-pin (with a
`# vX.Y.Z` comment) and bump via Dependabot.
