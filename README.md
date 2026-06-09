# APIsec GitHub Actions Demo

This repository demonstrates APIsec's GitHub Actions CI/CD integration using **crAPI** — OWASP's intentionally vulnerable API — as a live scan target. Two pull requests show two different integration patterns side by side: one that triggers a scan and passes, one that runs a severity gate and blocks the merge.

---

## What you will learn

- How APIsec integrates with GitHub Actions to scan APIs automatically on every pull request
- Two integration styles: a lightweight REST API trigger vs. the `apisec-scan-and-gate` Docker container
- How severity thresholds gate merges — blocking on Critical/High while advisory scans never block
- Secure credential handling: `APISEC_TOKEN` as a GitHub Secret (masked in all logs), non-sensitive config as GitHub Variables
- How scan results surface as PR comments with a direct link into the APIsec dashboard

---

## The two demo branches

| Branch | PR | Integration style | Severity gate | Expected result |
|---|---|---|---|---|
| [`demo/scan-pass`](../../pull/1) | [PR #1](../../pull/1) | REST API call — `POST /v1/applications/{id}/instances/{id}/scan` | None — advisory only | ✅ Check passes |
| [`demo/scan-block-high`](../../pull/2) | [PR #2](../../pull/2) | `docker.io/apisec/apisec-scan-and-gate` container | `APISEC_MAX_CRITICAL=0`, `APISEC_MAX_HIGH=0` | ❌ Check fails, merge blocked |

Open either PR and watch the **Checks** tab. PR #2 will show a failed check and a locked merge button — crAPI has real High/Critical findings that trigger the gate.

---

## How it works

### Trigger
Both workflows fire on `on: pull_request` — automatically on every PR open, update, or sync. No manual steps required.

### Runner isolation
Each job runs on a fresh `ubuntu-latest` GitHub-hosted VM. The runner is created for the job and destroyed when it finishes — no state persists between runs.

### Credential handling
| Name | Type | Purpose |
|---|---|---|
| `APISEC_TOKEN` | **Secret** | Bearer token — masked in all logs, never written to disk |
| `APISEC_APP_ID` | Variable | APIsec application identifier |
| `APISEC_INSTANCE_ID` | Variable | APIsec instance identifier |
| `APISEC_AUTH_ID` | Variable | Auth configuration used by the scanner |

Credentials are injected exclusively as environment variables — never passed on the command line or written to the runner filesystem.

### Severity gating (PR #2)
```
APISEC_MAX_CRITICAL: "0"   # any Critical → container exits non-zero → check fails
APISEC_MAX_HIGH:     "0"   # any High     → container exits non-zero → check fails
```
A non-zero exit code marks the step as failed, which marks the PR check as failed, which locks the **Merge pull request** button until findings are resolved.

### PR comment
After every scan, a summary comment is posted directly on the PR — no need to open the Actions tab. The comment includes the gate result and a direct link to the scan in the APIsec dashboard.

---

## Where to see the runs

| | Link |
|---|---|
| PR #1 — advisory scan (passes) | https://github.com/dave-apisec/apisec-github-actions-demo/pull/1 |
| PR #2 — severity gate (blocks) | https://github.com/dave-apisec/apisec-github-actions-demo/pull/2 |
| All workflow runs | https://github.com/dave-apisec/apisec-github-actions-demo/actions |

---

## Required GitHub configuration

To run these workflows in your own repo, configure the following under **Settings → Secrets and variables → Actions**:

| Type | Name | Description |
|---|---|---|
| Secret | `APISEC_TOKEN` | APIsec API bearer token |
| Variable | `APISEC_APP_ID` | Application ID from the APIsec platform |
| Variable | `APISEC_INSTANCE_ID` | Instance ID from the APIsec platform |
| Variable | `APISEC_AUTH_ID` | Auth configuration ID used during scanning |

---

## About crAPI

[crAPI](https://github.com/OWASP/crAPI) (Completely Ridiculous API) is an OWASP project purpose-built for API security training — it ships with realistic vulnerabilities across the OWASP API Security Top 10, which means it produces genuine High and Critical findings that make the severity gate demo concrete and repeatable.
