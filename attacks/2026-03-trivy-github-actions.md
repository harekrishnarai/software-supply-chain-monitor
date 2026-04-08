# Trivy GitHub Actions Tag Compromise

**Date:** March 20, 2026
**Ecosystem:** GitHub Actions
**Severity:** Critical
**Type:** Tag hijacking / CI/CD secret theft
**Author of disclosure:** Philipp Burckhardt (Socket Research Team)
**Source:** [socket.dev](https://socket.dev/blog/trivy-under-attack-again-github-actions-compromise)

---

## Summary

Attackers compromised the `aquasecurity/trivy-action` GitHub Actions repository by force-pushing malicious commits to **75 out of 76 version tags**. Any CI/CD pipeline referencing `aquasecurity/trivy-action` by version tag — including widely used tags — was exposed to an infostealer payload designed to harvest secrets and exfiltrate them to an attacker-controlled endpoint.

This is the **second** supply chain incident involving Trivy in March 2026. An earlier, separate attack compromised the Trivy VS Code extension on OpenVSX.

---

## How It Worked

### Tag Rewriting Without Touching Branches

The most notable aspect of this attack was its delivery mechanism. The attacker did not modify any branches — they **only rewrote git tags**. For each of the 75 poisoned tags, the attacker:

1. Started from the current `master` HEAD tree (`57a97c7e`)
2. Swapped `entrypoint.sh` with an infostealer payload, leaving all other files from master intact
3. Looked up the original commit the tag previously pointed to (e.g., the PR #481 merge for tag `0.33.0`)
4. Cloned that commit's metadata: author name, email, committer, both timestamps, and the full commit message including PR number
5. Set the parent to master HEAD (`57a97c7e`) rather than the original parent
6. Force-pushed the tag to this new commit

The result: all 75 malicious commits share an identical file tree, and `git log` output looks nearly identical to the legitimate history.

### Why Tag `0.35.0` Was Spared

Tag `0.35.0` was the only clean tag because it already pointed to the master HEAD commit (`57a97c7e`) — the same commit the attacker used as the base. Rewriting it would have been a no-op.

### Indicators of Forgery

Despite the sophisticated spoofing, a few tells betray the malicious commits:
- The original commits were GPG-signed by GitHub when merged via the web UI; the attacker's commits are **unsigned**
- Each commit claims a date from the original release year (2021, 2022, etc.) but has a parent dated **March 2026**
- Original commits typically touched multiple files; each malicious commit only modifies `entrypoint.sh`
- GitHub's release UI showed an **"Immutable" badge** — but this was misleading, as the tags had been rewritten before immutability was applied

---

## The Payload

The malicious `entrypoint.sh` is **204 lines** long. Lines 4–105 contain the injected infostealer; lines 106 onward are the legitimate original content.

### Stage 1: Secret Collection

**Process Environment Scraping (Lines 4–29)**
The script targets GitHub Actions runner processes (`Runner.Worker`, `Runner.Listener`, `runsvc`, `run.sh`), reads `/proc/<pid>/environ`, and extracts any environment variable containing `env` or `ssh` in its key name.

**GitHub-Hosted Runners: Process Memory Dump (Lines 30–32)**
On GitHub-hosted Linux runners (which grant `sudo` by default), a base64-encoded Python script dumps the full memory of the `Runner.Worker` process via `/proc/<pid>/mem`, searching for JSON blobs matching `{"value":"<secret>","isSecret":true}`.

**Self-Hosted Runners: Filesystem Credential Stealer (Lines 34–36)**
A second Python payload (labelled `## TeamPCP Cloud stealer`) walks the filesystem for credentials. Targeted categories:

| Category | Targets |
|----------|---------|
| Reconnaissance | `hostname`, `whoami`, `uname -a`, `ip addr`, `ip route`, `printenv` |
| SSH | `~/.ssh/id_rsa`, `id_ed25519`, `id_ecdsa`, `id_dsa`, `authorized_keys`, `known_hosts`, `/etc/ssh/ssh_host_*_key` |
| Git | `~/.git-credentials`, `~/.gitconfig` |
| AWS | `~/.aws/credentials`, `~/.aws/config`, `AWS_*` env vars, EC2 IMDS (`169.254.169.254`), ECS credentials |
| GCP | `~/.config/gcloud/*`, `application_default_credentials.json`, `GOOGLE_APPLICATION_CREDENTIALS` |
| Azure | `~/.azure/*`, `AZURE_*` env vars |
| Kubernetes | `~/.kube/config`, `/etc/kubernetes/*.conf`, `kubectl get secrets --all-namespaces -o json` |
| Docker | `~/.docker/config.json`, `/kaniko/.docker/config.json` |
| Env files | `.env`, `.env.local`, `.env.production`, `.env.development`, `.env.staging`, `.env.test` — searched in CWD, parent dirs, and `/opt`, `/srv`, `/var/www`, `/app`, `/data` |

### Stage 2: Encryption (Lines 39–68)

Collected data is encrypted with **AES-256-CBC**, and the AES key is wrapped with **RSA-4096**, making the stolen data unreadable without the attacker's private key.

### Stage 3: Exfiltration (Lines 70–104)

The encrypted bundle is transmitted to an attacker-controlled endpoint using a GitHub-based relay to blend in with expected CI/CD traffic.

---

## Root Cause

Per an update from Trivy maintainers on March 20, 2026, the attack was enabled by a **compromised credential** — a token with sufficient privileges to force-push tags in the `aquasecurity/trivy-action` repository.

---

## Detection

- Socket detected this activity in real time, beginning at **19:15 UTC** on March 20, 2026
- Socket users can check affected workflows in the dashboard under **Threat Intel → [Trivy GitHub Actions Compromise](https://socket.dev/supply-chain-attacks/trivy-github-actions-compromise)**

---

## Remediation

1. **Pin GitHub Actions to a full commit SHA**, not a tag or branch name
   ```yaml
   # Vulnerable
   - uses: aquasecurity/trivy-action@0.33.0
   # Safe
   - uses: aquasecurity/trivy-action@<full-commit-sha>
   ```
2. Rotate **all secrets** present in any CI/CD pipeline that used a compromised tag
3. Audit CI/CD logs for unexpected outbound network requests
4. Do not rely on GitHub's "Immutable" release badge as a security guarantee

---

## Related Incidents

- [Trivy OpenVSX / VS Code Extension Compromise](https://socket.dev/blog/unauthorized-ai-agent-execution-code-published-to-openvsx-in-aqua-trivy-vs-code-extension) — earlier March 2026
- [tj-actions/changed-files compromise](https://www.stepsecurity.io) — similar GitHub Actions tag hijacking pattern
