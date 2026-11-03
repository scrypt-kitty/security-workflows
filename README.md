# security-workflows

Reusable GitHub Actions workflows for dependency, secret, and static-analysis
scanning, built entirely from free, open-source tools. Pull one `uses:` line
into a repo's own workflow and it gets the same scan every other repo in this
ecosystem gets, without re-deriving which tool to run or re-licensing
anything.

## Why this exists, instead of just enabling GitHub's own features

GitHub's native security tooling (CodeQL code scanning, secret scanning,
Dependabot) splits unevenly by repo visibility:

| Feature | Public repos | Private repos |
|---|---|---|
| Dependabot alerts / security updates | Free | Free |
| Code scanning (CodeQL) | Free | Requires GitHub Advanced Security (paid) |
| Secret scanning | Free | Requires GitHub Advanced Security (paid) |
| Copilot code review | Free | Needs a Copilot seat; as of June 2026 also consumes Actions minutes on private repos |

Dependabot is already free everywhere and worth enabling directly in each
repo's settings — this repo doesn't duplicate it. Everything here exists to
cover the gap: equivalents for code scanning and secret scanning that work on
private repos without an Advanced Security license, plus a couple of scanners
GitHub doesn't offer a native equivalent for at all (container/SBOM scanning,
a second dependency-vulnerability feed).

**The one thing to know before using any of this on a private repo:**
GitHub's Security tab (where SARIF results normally render) is gated by
Advanced Security *regardless of which tool produced the SARIF* — it is not
CodeQL-specific. Uploading Trivy's or Semgrep's SARIF there on a private repo
without that license either silently does nothing or errors. Every workflow
here defaults to **not** uploading SARIF; findings instead fail the job and
print to the step summary and job logs, which works everywhere. Flip
`upload-sarif: true` only on repos that actually have the license (or are
public).

## What's here

| Workflow | Tool(s) | Replaces |
|---|---|---|
| [`dependency-scan.yml`](.github/workflows/dependency-scan.yml) | Trivy + osv-scanner | A second opinion beyond Dependabot — two independent vulnerability feeds |
| [`secret-scan.yml`](.github/workflows/secret-scan.yml) | gitleaks | Secret scanning, on private repos |
| [`sast-scan.yml`](.github/workflows/sast-scan.yml) | Semgrep (Community rules) | CodeQL code scanning, on private repos |
| [`container-scan.yml`](.github/workflows/container-scan.yml) | Grype + Syft | Container image vulnerability scanning and SBOM generation |

[`examples/full-scan.yml`](examples/full-scan.yml) is a complete caller
workflow to copy into a consuming repo — trim whichever jobs don't apply.

## Using one of these in another repo

```yaml
# .github/workflows/security.yml
on: [push, pull_request]
jobs:
  deps:
    uses: scrypt-kitty/security-workflows/.github/workflows/dependency-scan.yml@main
  secrets:
    uses: scrypt-kitty/security-workflows/.github/workflows/secret-scan.yml@main
```

`security-workflows` is public, so any repo — regardless of its own
visibility — can reference `@main` without needing a cross-repo token.
Pin to a tag instead of `@main` once this repo has a release, so a change
here doesn't silently change behavior in every repo that calls it on the
next run.

## Where results go on a private repo

Without SARIF upload, a finding fails the calling job and shows in that job's
log and step summary — enough to block a PR via a required status check, but
nothing persists across runs. If you want a standing dashboard instead,
point the scanners' JSON/SARIF output at a self-hosted vulnerability manager
(e.g. DefectDojo) rather than GitHub's Security tab. That's not wired up here
yet — each scanner's `format: sarif` or `format: json` output is already
produced as a workflow artifact in these jobs; importing it into DefectDojo
is a small follow-up per consuming repo, not something this repo needs to
know about.

## Conventions

- Every workflow here is `workflow_call`-only — none run standing scans of
  this repo itself. There is no application code here to scan.
- Pin action versions (`@v4`, `@0.28.0`), not `@latest` or an unpinned major,
  so a workflow's behavior doesn't change out from under every repo calling
  it without a deliberate bump here.
- No credentials required for any default path. The one optional secret
  (`SEMGREP_APP_TOKEN`) only enables Semgrep's own dashboard and is never
  required for the scan to run or to fail correctly.
- Fixes and new scanners go through a branch and PR, same discipline as
  `claude-skills` — a workflow that is wrong fails silently everywhere it's
  called, which is worse than one that's merely missing.
