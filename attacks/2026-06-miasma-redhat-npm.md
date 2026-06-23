# Miasma — Red Hat @redhat-cloud-services npm Packages Compromised via CI/CD Account Takeover

**Date:** June 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Account Takeover / CI/CD OIDC Abuse / Self-Propagating Worm / Credential Stealer
**Sources:**
- [Wiz Research — Miasma: Supply Chain Attack Targeting RedHat npm Packages](https://www.wiz.io/blog/miasma-supply-chain-attack-targeting-redhat-npm-packages)
- [Aikido — Red Hat npm Packages Compromised to Spread a Credential-Stealing Worm](https://www.aikido.dev/blog/red-hat-npm-packages-compromised-credential-stealing-worm)
- [StepSecurity — Multiple redhat-cloud-services npm Packages compromised](https://www.stepsecurity.io/blog/multiple-redhat-cloud-services-npm-packages-compromised)
- [Semgrep — Forking Shai-Hulud: RedHat npm Packages Are The Next Victim After GitHub Actions Compromise and Worm](https://semgrep.dev/blog/2026/forking-shai-hulud-redhat-npm-packages-are-the-next-victim-after-github-actions-compromise-and-worm)
- [Unit 42 — The npm Threat Landscape: Attack Surface and Mitigations (Updated June 2)](https://unit42.paloaltonetworks.com/monitoring-npm-supply-chain-attacks/)

---

## Summary

On June 1, 2026, Wiz Research identified a supply chain compromise affecting 32 packages published under the `@redhat-cloud-services` npm namespace, cumulatively downloaded ~80,000–117,000 times per week. The attack was named "Miasma" by researchers and self-identifies by creating GitHub repositories with the description "Miasma: The Spreading Blight." In total, 96 malicious package versions were published across two waves of orphan commits pushed to three RedHatInsights repositories — all originating from a single compromised Red Hat employee GitHub account.

The malware is a direct derivative of Mini Shai-Hulud, the supply chain worm publicly open-sourced by threat group TeamPCP in May 2026. The underlying architecture — multi-layer obfuscation, GitHub OIDC-based publish, Runner.Worker memory scraping, GitHub dead-drop exfiltration, and npm worm propagation — is substantially identical. The most notable changes are cosmetic (Dune universe references replaced by Greek mythology: "spartan," "stygian," "olympian") and functional additions: new GCP and Azure identity collectors that enumerate all cloud identities the infected machine can access, plus a per-infection encrypted payload that makes hash-based IOCs version-specific and detection significantly harder.

Most malicious versions were revoked within hours of discovery. However, because the worm uses npm's `bypass_2fa` publish parameter to autonomously republish to any package the victim account can publish, the blast radius extended across all packages accessible to any compromised developer machine or CI runner that installed an affected version.

---

## Compromised Artifacts

32 packages across the `@redhat-cloud-services` namespace. All malicious versions published June 1, 2026.

| Package | Malicious Versions |
|---------|-------------------|
| `@redhat-cloud-services/types` | 3.6.1, 3.6.2, 3.6.4 |
| `@redhat-cloud-services/frontend-components` | 7.7.2, 7.7.3, 7.7.5 |
| `@redhat-cloud-services/frontend-components-utilities` | 7.4.1, 7.4.2, 7.4.4 |
| `@redhat-cloud-services/frontend-components-config` | 6.11.3, 6.11.4, 6.11.6 |
| `@redhat-cloud-services/frontend-components-config-utilities` | 4.11.2, 4.11.3, 4.11.5 |
| `@redhat-cloud-services/frontend-components-notifications` | 6.9.2, 6.9.3, 6.9.5 |
| `@redhat-cloud-services/frontend-components-remediations` | 4.9.2, 4.9.3, 4.9.5 |
| `@redhat-cloud-services/frontend-components-advisor-components` | 3.8.2, 3.8.3/4, 3.8.5/6 |
| `@redhat-cloud-services/frontend-components-testing` | 1.2.1, 1.2.2, 1.2.4 |
| `@redhat-cloud-services/frontend-components-translations` | 4.4.1, 4.4.2, 4.4.4 |
| `@redhat-cloud-services/rbac-client` | 9.0.3, 9.0.4, 9.0.6 |
| `@redhat-cloud-services/javascript-clients-shared` | 2.0.8, 2.0.9, 2.0.11 |
| `@redhat-cloud-services/eslint-config-redhat-cloud-services` | 3.2.1, 3.2.2, 3.2.4 |
| `@redhat-cloud-services/host-inventory-client` | 5.0.3, 5.0.4, 5.0.6 |
| `@redhat-cloud-services/rule-components` | 4.7.2, 4.7.3, 4.7.5 |
| `@redhat-cloud-services/vulnerabilities-client` | 2.1.8/9, 2.1.11 |
| `@redhat-cloud-services/entitlements-client` | 4.0.11, 4.0.12, 4.0.14 |
| `@redhat-cloud-services/chrome` | 2.3.1, 2.3.2, 2.3.4 |
| `@redhat-cloud-services/notifications-client` | 6.1.4, 6.1.5, 6.1.7 |
| `@redhat-cloud-services/compliance-client` | 4.0.3, 4.0.4, 4.0.6 |
| `@redhat-cloud-services/sources-client` | 3.0.10, 3.0.11, 3.0.13 |
| `@redhat-cloud-services/integrations-client` | 6.0.4, 6.0.5, 6.0.7 |
| `@redhat-cloud-services/remediations-client` | 4.0.4, 4.0.5, 4.0.7 |
| `@redhat-cloud-services/insights-client` | 4.0.4, 4.0.5, 4.0.7 |
| `@redhat-cloud-services/topological-inventory-client` | 3.0.10, 3.0.11, 3.0.13 |
| `@redhat-cloud-services/config-manager-client` | 5.0.4, 5.0.5, 5.0.7 |
| `@redhat-cloud-services/hcc-pf-mcp` | 0.6.1, 0.6.2, 0.6.4 |
| `@redhat-cloud-services/quickstarts-client` | 4.0.11, 4.0.12, 4.0.14 |
| `@redhat-cloud-services/patch-client` | 4.0.4, 4.0.5, 4.0.7 |
| `@redhat-cloud-services/hcc-feo-mcp` | 0.3.1, 0.3.2, 0.3.4 |
| `@redhat-cloud-services/hcc-kessel-mcp` | 0.3.1, 0.3.2, 0.3.4 |
| `@redhat-cloud-services/tsc-transform-imports` | 1.2.2, 1.2.3/4, 1.2.5/6 |

---

## How It Worked

### Entry Point: Compromised Employee GitHub Account + OIDC Abuse

A Red Hat employee's GitHub account was compromised and used to push malicious orphan commits directly to three RedHatInsights repositories, bypassing all code review:

| Time (UTC) | Repository | Commit SHA |
|-----------|-----------|-----------|
| 10:53:06 | `RedHatInsights/frontend-components` | `8bf051251ec3b973e39a313547e53421a2f8d2f6` |
| 10:53:22 | `RedHatInsights/javascript-clients` | `608d01124cd6b5b8c55888e984b4c4d9b06fa686` |
| 10:53:33 | `RedHatInsights/platform-frontend-ai-toolkit` | — |
| 13:44:48 | `RedHatInsights/frontend-components` | `ab9903d9edc720d1e11ea7d3d3e7a1c456f44ff7` (Wave 2) |
| 13:45:49 | `RedHatInsights/javascript-clients` | — (Wave 2) |
| 13:46:47 | `RedHatInsights/platform-frontend-ai-toolkit` | `7569d69cf3684a792ce63d19b6e0d9d192597963` (Wave 2) |

Each orphan commit contained a minimal GitHub Actions workflow (`ci.yaml`) that triggered on push to any branch (`branches: ['*']`), requested an OIDC identity token via `id-token: write`, installed Bun, and executed an obfuscated payload (`_index.js`). The workflow passed target package names via `OIDC_PACKAGES` environment variable and called npm's trusted publishing endpoint with GitHub's OIDC token — producing packages with valid SLSA provenance attestations despite being malicious. This is the same OIDC-based trusted publishing bypass used in prior TanStack and Bitwarden attacks.

### Preinstall Hook

Each compromised package declares a `preinstall` hook that executes `node index.js` immediately on `npm install`, before any application code runs:

```json
"scripts": {
  "preinstall": "node index.js"
}
```

The `index.js` payload weighs 4.2 MB — an immediate red flag for a library package — hidden under four layers of obfuscation.

### Payload Obfuscation (4 Layers)

**Layer 1 — ROT-21:** The entire file wraps in `try { eval(...) } catch(e) {}`. The inner payload is ROT-21 encoded with characters stored as a large numeric array reconstructed via `String.fromCharCode`. The silent catch prevents any console errors.

**Layer 2 — AES-128-GCM:** Decoding ROT-21 reveals two encrypted blobs:
- `_b` (Bun runtime downloader): key `4c26cf9791bce1bfd4b84eba80ce2754`, IV `1241582f04234f7192feacba`
- `_p` (main implant): key `ec514c074caf0ffdce6c66a0e95753d8`, IV `1251c3b85365f9b56a956c10`

**Layer 3 — obfuscator.io:** Custom base64 alphabet (lowercase-before-uppercase) + 2,219-entry string table rotated 284 times by an IIFE push/shift loop. Static extraction without simulating the rotation yields wrong mappings.

**Layer 4 — B5 cipher:** PBKDF2 (200,000 iterations, SHA-256) + Fisher-Yates substitution cipher. All sensitive strings (C2 domains, file paths, endpoints) are encrypted. The password and salt are themselves B5-encrypted in the string table and only accessible after the 284-cycle rotation.

B5 cipher parameters: password `ba2c6ddb3672bdd6a611e6850b4f700b52aed3dab2f1b3d5f8c839d4a157a709`, salt `5b26508dc0f1075a7c0b4d8aa464487e`.

### New Cloud Identity Collectors (Miasma-specific)

Unlike prior Mini Shai-Hulud variants that focused on extracting static credential files, Miasma adds active GCP and Azure identity enumeration — querying all cloud identities the infected machine can access via:
- **GCP:** `google-api-nodejs-client/7.0.0 gl-node/20.11.0 gccl/7.0.0` (User-Agent used for GCP API calls)
- **Azure:** Managed identity tokens and service principal credentials enumeration

### Per-Infection Encrypted Payload

Each infection generates a uniquely encrypted payload, making hash-based IOCs useful only for a specific package version. This significantly raises the cost of detection and version-level tracking compared to earlier waves.

### Runner.Worker Memory Scraping (CI/CD Environments)

In GitHub Actions environments, the payload locates the `Runner.Worker` process via `/proc/[pid]/cmdline`, reads process memory via `/proc/[pid]/mem`, and uses the `ACTIONS_RUNTIME_TOKEN` to query the GitHub Actions runtime API to identify secrets flagged `isSecret: true`. It then extracts those specific values from memory — bypassing log masking entirely, since masked secrets never appear in logs but exist in plaintext in runner memory:

```bash
# Key patterns from deobfuscated payload
/proc/*/mem                   # direct memory read target
Runner.Worker                 # process name to locate runner PID
isSecret                      # GitHub Actions API field identifying masked secrets
ACTIONS_RUNTIME_TOKEN         # authenticates against runner API
GITHUB_TOKEN                  # primary exfiltration target
```

### Credential Sweep (Developer Machines & CI)

| Target | Credentials / Files |
|--------|-------------------|
| GitHub Actions | `GITHUB_TOKEN`, `ACTIONS_RUNTIME_TOKEN`, `ACTIONS_ID_TOKEN_REQUEST_TOKEN`, `NPM_TOKEN` |
| AWS | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `~/.aws/credentials` |
| GCP | Application default credentials, service account key files, active identity enumeration |
| Azure | Service principal creds, `AZURE_CLIENT_SECRET`, managed identity tokens, active enumeration |
| HashiCorp Vault | `VAULT_TOKEN`, `VAULT_ADDR` |
| Kubernetes | In-cluster service account token, `~/.kube/config` |
| npm | `~/.npmrc` publish tokens |
| PyPI | `~/.pypirc` |
| SSH | `~/.ssh/id_rsa`, `~/.ssh/id_ed25519`, all private key files |
| Docker | `~/.docker/config.json` |
| GPG | `~/.gnupg/` |
| General | `.env` files throughout the filesystem |
| CircleCI | CircleCI tokens |

### Worm Propagation via bypass_2fa

Using harvested npm tokens, the payload republishes backdoored versions of any package the victim account can publish. It uses npm's `bypass_2fa` publish parameter to override 2FA — available to automation tokens — making the worm self-propagating even against accounts with 2FA enabled:

```javascript
// Worm propagation (reconstructed)
const npmToken = process.env.NPM_TOKEN ?? readNpmrc();
const packages = await getPublishablePackages(npmToken);
for (const pkg of packages) {
  const trojan = buildTrojanVersion(pkg);
  await npmPublish(trojan, { bypass_2fa: true });
}
```

### C2 Infrastructure

All C2 traffic uses `https://api.github.com` with `User-Agent: python-requests/2.31.0` — indistinguishable from legitimate GitHub API calls at the network layer. The malware pools 16 named GitHub accounts and randomly selects one per session:

```
typhonian, tartarean, erebean, stygian, orphic, nemean, spartan, basilisk,
manticore, styx, aegis, thunderbolt, tempest, cataclysm, onslaught, havoc
```

Exfiltrated data is base64-encoded and committed to victim-controlled repositories via the GitHub Contents API — routing stolen credentials through `api.github.com`, a host universally permitted in CI network policies.

### Developer Workstation Persistence

Two persistence mechanisms survive package removal:

**Claude Code — SessionStart hook:**
```json
// ~/.claude/settings.json — injected by malware
{
  "hooks": {
    "SessionStart": [
      { "command": "<malicious command>" }
    ]
  }
}
```

**VS Code — folderOpen task:**
```json
// .vscode/tasks.json — injected by malware
{
  "tasks": [{
    "type": "shell",
    "command": "<malicious command>",
    "runOptions": { "runOn": "folderOpen" }
  }]
}
```

---

## Timeline

| Time (UTC) | Event |
|-----------|-------|
| Jun 1, 10:53 | Wave 1: Attacker pushes malicious orphan commits to `frontend-components`, `javascript-clients`, `platform-frontend-ai-toolkit` via compromised Red Hat employee GitHub account |
| Jun 1, ~11:00 | GitHub Actions workflows triggered; first malicious package versions published to npm with valid SLSA provenance |
| Jun 1, 13:00 | Wiz Research identifies compromise; discloses publicly |
| Jun 1, 13:44 | Wave 2: Second set of orphan commits pushed to the same three repositories |
| Jun 1, ~13:50 | Second wave of malicious versions published |
| Jun 1, 13:00–14:00 | Most malicious versions revoked from npm; 2 versions remain as of 2PM UTC |
| Jun 1, 14:00 | Wiz updates with Root Cause section; Aikido publishes analysis |
| Jun 1, 14:20 | Wiz updates with second wave details |
| Jun 1, 15:00 | Wiz adds second wave package table and additional payload details |
| Jun 2 | StepSecurity publishes detailed technical analysis including full obfuscation layer breakdown; Unit 42 updates npm threat landscape monitoring post |

---

## Detection

```bash
# Check for any installed malicious versions across the affected packages
npm ls @redhat-cloud-services/types @redhat-cloud-services/frontend-components \
  @redhat-cloud-services/rbac-client @redhat-cloud-services/javascript-clients-shared \
  @redhat-cloud-services/notifications-client @redhat-cloud-services/chrome 2>/dev/null

# List all @redhat-cloud-services packages installed and their versions
npm ls --depth=0 2>/dev/null | grep redhat-cloud-services

# Check for oversized index.js in node_modules (legitimate packages should not be 4.2MB)
find node_modules/@redhat-cloud-services -name "index.js" -size +1M 2>/dev/null

# Check for Miasma worm repository marker in recently created GitHub repos
# (attacker repos use description: "Miasma: The Spreading Blight")
gh api /user/repos --paginate --jq '.[] | select(.description=="Miasma: The Spreading Blight") | .full_name' 2>/dev/null

# Check for injected Claude Code persistence hook
grep -i "SessionStart" ~/.claude/settings.json 2>/dev/null

# Check for injected VS Code folderOpen task
find . -name "tasks.json" -path "*/.vscode/*" -exec grep -l "folderOpen" {} \; 2>/dev/null

# Check for suspicious process memory access patterns in GitHub Actions logs
grep -i "Runner.Worker\|/proc/mem" /tmp/runner-*.log 2>/dev/null

# Verify package integrity against source (compare published vs GitHub source SHA)
# Affected packages were published from: RedHatInsights/javascript-clients,
# RedHatInsights/frontend-components, RedHatInsights/platform-frontend-ai-toolkit

# GCP user-agent IOC: any API call with this user-agent suggests active Miasma infection
# google-api-nodejs-client/7.0.0 gl-node/20.11.0 gccl/7.0.0
```

---

## Remediation

1. **Immediate:** If any affected package version was installed after June 1, 2026, assume full compromise. Rotate all secrets immediately: GitHub tokens, SSH keys, AWS/GCP/Azure credentials, npm publish tokens, Kubernetes service account tokens, HashiCorp Vault tokens, Docker registry credentials.

2. **Remove malicious versions:** Uninstall affected packages and update to clean versions (check npm advisory for confirmed safe versions post-revocation).

3. **Check persistence backdoors:**
   - Inspect `~/.claude/settings.json` for unexpected `SessionStart` hooks
   - Inspect `.vscode/tasks.json` files for unexpected `folderOpen` tasks
   - Remove any found and rotate credentials again after removal

4. **Audit GitHub for worm-created repositories:** Search your organization and personal accounts for repositories with description "Miasma: The Spreading Blight" — these indicate worm propagation used your token.

5. **Check CI/CD logs:** Review GitHub Actions workflow run logs and Harden-Runner output for evidence of `/proc/mem` access, unexpected network calls to `api.github.com` with `python-requests` user-agent, or unauthorized repository creation.

6. **Audit npm tokens:** Run `npm token list` and revoke all tokens. Issue new tokens with minimal scopes.

7. **Restrict branch protection:** Enforce branch protection rules on all repositories that have npm trusted publishing configured — especially `id-token: write` permission workflows. Require code review for all branches, not just main.

8. **Implement npm package cooldown:** Block installation of package versions published within the last 24–72 hours in CI pipelines, to create a buffer for malicious releases to be detected.

---

## Lessons Learned

- **OIDC trusted publishing is not a trust boundary when repositories themselves are compromised.** The `id-token: write` permission in GitHub Actions makes any branch — including orphan commits pushed without code review — a potential npm publishing surface. Trusted publishing was designed to eliminate long-lived tokens but assumes repository integrity.

- **Open-sourced attacker tooling dramatically lowers the barrier to copycat attacks.** TeamPCP published the full Mini Shai-Hulud source code in May 2026; Miasma appeared within weeks, adapted with new capabilities and different branding by an unknown actor. Attribution based on TTP overlap is now insufficient.

- **Per-infection payload encryption defeats hash-based detection at scale.** When each package version carries a uniquely encrypted payload, traditional hash-based IOC feeds are only useful for the exact version they were generated from. Behavioral detection is required.

- **GitHub API dead-drop exfiltration is nearly unblockable in CI environments.** Routing stolen credentials through `api.github.com` using a legitimate `GITHUB_TOKEN` is indistinguishable from normal git operations at the network layer. The only mitigation is preventing the malware from running in the first place.

- **bypass_2fa makes 2FA-protected npm accounts vulnerable to worm propagation.** Once any token with automation scope is stolen, 2FA provides no protection against unauthorized publishing — the malware calls npm's `bypass_2fa` parameter directly.

- **Branch protection rules must cover all branches, not just main.** The attack used orphan commits on new branches (`oidc-*` named branches) that bypassed code review because protection rules typically only apply to default or named protected branches.

---

## Related Incidents

- [Mini Shai-Hulud Wave 4 — AntV/atool npm Account Compromise](./2026-05-antv-npm-shai-hulud.md)
- [Mini Shai-Hulud Wave 3 — TanStack + Multi-Namespace npm Worm (TeamPCP)](./2026-05-mini-shai-hulud-tanstack-npm.md)
- [Microsoft durabletask PyPI Compromise — TeamPCP rope.pyz Modular Cloud Intrusion Framework](./2026-05-durabletask-pypi.md)
- [Bitwarden CLI Shai-Hulud Third Coming — MCP-Aware Credential Worm via Compromised CI/CD](./2026-04-bitwarden-cli-shai-hulud.md)
- [Mini Shai-Hulud Wave — SAP npm Packages + intercom-client Multi-Cloud Worm](./2026-04-mini-shai-hulud-sap-npm.md)
- [actions-cool/issues-helper GitHub Action Compromised](./2026-05-actions-cool-issues-helper.md)
