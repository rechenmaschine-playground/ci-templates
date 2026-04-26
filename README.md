# ci-templates

Shared GitHub Actions reusable workflows for the org. Consumed via
`uses: <org>/ci-templates/.github/workflows/<name>.yml@<sha>`.

## Workflows

### `secrets-scan.yml`

Scans the consuming repo's full git history with
[gitleaks](https://github.com/gitleaks/gitleaks) on every PR, every
push to `main`, and once daily on cron. Uploads SARIF findings to the
GitHub Code Scanning Security tab when available.

**To onboard a repo:** copy [`templates/secrets-scan.yml`](templates/secrets-scan.yml)
into the consumer's `.github/workflows/` directory and replace `<sha>` /
`<vX.Y.Z>` with the latest release of this repo. Dependabot keeps it
fresh after that.

**Pre-requisites at the org level:**

- `GITLEAKS_LICENSE` secret (free for orgs at <https://gitleaks.io>).
  Scope it to the repos that need it (or "All repositories").
- Settings → Actions → General → Access: "Allow access to private repos"
  must be on if both this repo and the consumer are private.

**Behavior:**

- Default runner: `ubuntu-latest`. Override via `with: runs-on: self-hosted`.
- `fetch-depth: 0` — full git clone for full-history scanning.
- Submodules are **not** fetched — by org policy each repo scans itself.
- gitleaks default config. Suppress known-OK historical hits via a
  `.gitleaksignore` file in the consumer repo (one fingerprint per line,
  format `<commit>:<file>:<rule>:<line>`).
- Caller must grant `contents: read`, `pull-requests: write`, and
  `security-events: write`. GitHub fails the run at startup if any are
  missing — the consumer template has them set.

**On the BASE_REF override:**

gitleaks-action's push-event scan defaults to an incremental
`--first-parent --no-merges` range that silently misses merge commits.
We close that gap by setting `BASE_REF` to the second commit of history
on push and schedule events, which forces a full-history scan via the
action's hardcoded `${BASE_REF}^..${HEAD}` range. PR events keep the
default (PR's base..head) so the action's PR comment stays scoped to
the diff under review. `BASE_REF` is documented only in the action's
source (`src/index.js:165-168`); when Dependabot bumps the action,
re-verify this still works before merging.

**On the GitHub Advanced Security overlap:**

This workflow runs gitleaks via `gitleaks/gitleaks-action`, which
requires a license key for orgs. The reason we use the action (instead
of calling the gitleaks CLI directly) is its PR comment posting. On
public repos and private repos with GitHub Advanced Security, the
`github-advanced-security` bot already comments on PRs with secret
findings, making the action's comment redundant. If the org buys GHAS
for all private repos, this workflow + license + BASE_REF dance can be
torn out and replaced with native Code Scanning. Until then, the
workflow earns its keep on private-no-GHAS repos.

## Versioning

Tag releases as `vMAJOR.MINOR.PATCH`. Consumers SHA-pin (with a
`# vX.Y.Z` comment) and bump via Dependabot.

| Bump | When |
|---|---|
| Major | Breaking change to the workflow's public interface (input/secret signature, job name) |
| Minor | New behavior (e.g. closing a scan gap) |
| Patch | Bug fixes, comment/doc tweaks, dependency-only updates |
