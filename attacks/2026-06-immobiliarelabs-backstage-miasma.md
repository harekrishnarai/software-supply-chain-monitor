# @immobiliarelabs Backstage Plugins Compromised — Miasma binding.gyp Wave Hits 4 npm Packages

**Date:** June 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Supply Chain Worm / Credential Stealer / AI Assistant Poisoning (Miasma variant)
**Sources:**
- [StepSecurity — Multiple @immobiliarelabs Backstage Plugins Compromised on npm](https://www.stepsecurity.io/blog/immobiliarelabs-npm-packages-compromised)

---

## Summary

On June 26, 2026, multiple versions across four npm packages maintained by Immobiliare Labs — an Italian real estate company — were found to carry a malicious payload consistent with the ongoing Miasma supply chain worm campaign. The affected packages are @immobiliarelabs/backstage-plugin-gitlab, @immobiliarelabs/backstage-plugin-gitlab-backend, @immobiliarelabs/backstage-plugin-ldap-auth, and @immobiliarelabs/backstage-plugin-ldap-auth-backend, all open-source Backstage plugins used by platform engineering teams running self-hosted Backstage instances. A total of 22 malicious versions were published in a 30-second window, inserting new patch releases into every supported major release series simultaneously.

The attack uses the same Phantom Gyp binding.gyp execution technique documented in Miasma v2: each compromised version includes a 5 MB index.js and a binding.gyp native addon manifest that triggers payload execution via node-gyp shell expansion during `npm install`, bypassing postinstall script monitoring tools entirely. The three-layer obfuscation stack (ROT-2 Caesar cipher → AES-128-GCM → obfuscator.io) and the use of Bun v1.3.13 as the execution runtime are byte-for-byte consistent with prior Miasma waves. The payload harvests credentials from GitHub Actions, all major cloud providers, HashiCorp Vault, package registries, and password managers, and persists in AI coding assistant configuration files. Worm propagation functions for npm and PyPI were also present in the payload.

StepSecurity discovered the compromise through static analysis comparing the tarball for @immobiliarelabs/backstage-plugin-gitlab@2.1.2 against the prior clean release 2.1.1, noting the presence of the previously-absent 5 MB index.js and binding.gyp files. They reported findings to Immobiliare Labs via GitHub issue #1052 on the same day.

---

## Compromised Artifacts

| Package | Malicious Version(s) |
|---------|----------------------|
| `@immobiliarelabs/backstage-plugin-gitlab` | 1.0.1, 2.1.2, 3.0.3, 4.0.2, 5.2.1, 6.13.1, 7.0.2 |
| `@immobiliarelabs/backstage-plugin-gitlab-backend` | 3.0.3, 4.0.2, 5.2.1, 6.13.1, 7.0.2 |
| `@immobiliarelabs/backstage-plugin-ldap-auth` | 1.1.4, 2.0.5, 3.0.2, 4.3.2, 5.2.1 |
| `@immobiliarelabs/backstage-plugin-ldap-auth-backend` | 1.1.3, 2.0.5, 3.0.2, 4.3.2, 5.2.1 |

All 22 versions were published within a 30-second window on June 26, 2026, each inserted as a new patch release into every supported major release series.

---

## How It Worked

### Entry Point — binding.gyp Hook (Phantom Gyp Technique)

Each compromised version contains two files absent from all prior releases: a 5 MB `index.js` and a `binding.gyp` native addon manifest. The binding.gyp contains:

```json
{
  "targets": [{
    "target_name": "Setup",
    "type": "none",
    "sources": ["<!(node index.js > /dev/null 2>&1 && echo stub.c)"]
  }]
}
```

The `<!(...)` source expansion causes node-gyp to execute the shell command during `npm install`, triggering `node index.js` without declaring a `scripts.postinstall` entry. This bypasses security tools that only monitor `package.json` lifecycle scripts.

### Payload Mechanics — Three-Layer Obfuscation

The 5 MB index.js uses a layered obfuscation stack:

1. **ROT-2 Caesar cipher** applied to all alphabetic characters, wrapping the entire payload in a `try { eval(...) }` block.
2. **AES-128-GCM decryption** of two embedded ciphertext blobs using hardcoded keys: blob `_b` decrypts to a Bun runtime downloader; blob `_p` decrypts to the main credential harvesting payload.
3. **obfuscator.io string table rotation** on the main payload, with a secondary encryption layer on the most sensitive strings.

After decryption, the outer wrapper downloads Bun v1.3.13 from the official GitHub releases endpoint (`https://github.com/oven-sh/bun/releases/download/bun-v1.3.13/bun-<os>-<arch>.zip`), writes the decrypted payload to a randomly named temp file, executes it with Bun, then deletes the file. Using Bun rather than Node prevents interception via Node's `--require` hook, a technique relied upon by many Node.js security monitoring tools.

### Credential Harvesting

The decoded payload string table reveals credential theft from:

- GitHub tokens (PATs, App JWTs, OIDC tokens, runner tokens)
- GitHub Actions masked secrets via direct read of the Runner.Worker process through `/proc/<pid>/mem`
- AWS credentials (environment variables, IMDS at 169.254.169.254, ECS task metadata at 169.254.170.2, SSM Parameter Store, Secrets Manager)
- Google Cloud Platform service account credentials
- Azure managed identity, service principal, and Key Vault credentials
- Kubernetes service account tokens and namespace secrets
- HashiCorp Vault tokens (`/home/runner/.vault-token`, `/root/.vault-token`, `/run/secrets/VAULT_TOKEN`)
- npm, PyPI, RubyGems, and JFrog Artifactory tokens
- Password manager databases (Bitwarden, 1Password, gopass)
- SSH keys (`~/.ssh/`)

### AI Coding Assistant Persistence

The payload's `infectHost` function writes persistent backdoors to configuration files for multiple AI coding assistants:

- **Claude Code**: `.claude/settings.json` (SessionStart hook)
- **GitHub Copilot**: `.github/copilot-instructions.md`
- **Cursor**: `.cursor/rules/setup.mdc`
- **VS Code**: `.vscode/tasks.json` (folderOpen trigger)
- **Aider**: `.aider.conf.yml`
- **Amazon Kiro, Sourcegraph Cody, Google Gemini, OpenAI Codex** (additional targets)

### Worm Propagation

The payload contains functions named `squatPackage`, `updateTarball`, `handleNpmTokens`, and `handlePypiTokens` indicating self-propagation capability to npm and PyPI using stolen registry credentials, consistent with prior Miasma wave behavior.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| June 26, 2026 | 22 malicious versions published across 4 packages within a 30-second window |
| June 26, 2026 | StepSecurity detects via static analysis of @immobiliarelabs/backstage-plugin-gitlab@2.1.2 vs 2.1.1 |
| June 26, 2026 | StepSecurity reports to Immobiliare Labs via GitHub issue #1052 |
| June 26, 2026 | Public disclosure by StepSecurity |

---

## Detection

```bash
# Check if any compromised versions are installed
npm ls @immobiliarelabs/backstage-plugin-gitlab 2>/dev/null | grep -E '1\.0\.1|2\.1\.2|3\.0\.3|4\.0\.2|5\.2\.1|6\.13\.1|7\.0\.2'
npm ls @immobiliarelabs/backstage-plugin-gitlab-backend 2>/dev/null | grep -E '3\.0\.3|4\.0\.2|5\.2\.1|6\.13\.1|7\.0\.2'
npm ls @immobiliarelabs/backstage-plugin-ldap-auth 2>/dev/null | grep -E '1\.1\.4|2\.0\.5|3\.0\.2|4\.3\.2|5\.2\.1'
npm ls @immobiliarelabs/backstage-plugin-ldap-auth-backend 2>/dev/null | grep -E '1\.1\.3|2\.0\.5|3\.0\.2|4\.3\.2|5\.2\.1'

# Verify package tarball integrity (for gitlab@2.1.2)
# Known malicious SHA512:
# sha512-k7pGY+wScfqX51fpF412dOze6kSIytHYwZAXPhu6pDV+R7JWnD98Uc0nzGVHFead99nwWU4x56fkre/jH3Q7Xg==

# Detect presence of the malicious 5 MB index.js
find $(npm root) -path '*immobiliarelabs*' -name 'index.js' -size +4M 2>/dev/null

# Detect presence of binding.gyp in package root (absent from clean versions)
find $(npm root) -path '*immobiliarelabs*' -name 'binding.gyp' 2>/dev/null

# Check for Bun runtime download artifacts (downloaded during execution)
find /tmp /var/tmp ~ -name 'bun' -newer /tmp 2>/dev/null | head -20

# Check for outbound connection to Bun release endpoint during install
# In CI: grep "oven-sh/bun" workflow runner network logs

# Check for AI assistant config poisoning
for f in .claude/settings.json .cursor/rules/setup.mdc .vscode/tasks.json .github/copilot-instructions.md .aider.conf.yml; do
  [ -f "$f" ] && echo "[CHECK] $f" && cat "$f"
done

# Grep for SessionStart hook (Claude Code persistence)
grep -r "SessionStart" .claude/ 2>/dev/null
```

---

## Remediation

1. Run the detection commands above to confirm whether any compromised version is installed.
2. Remove all compromised versions immediately: `npm uninstall @immobiliarelabs/backstage-plugin-gitlab @immobiliarelabs/backstage-plugin-gitlab-backend @immobiliarelabs/backstage-plugin-ldap-auth @immobiliarelabs/backstage-plugin-ldap-auth-backend`
3. Upgrade to a clean version: verify the target version predates June 26, 2026 (or check npm for a clean post-patch release).
4. **Rotate all credentials** that may have been present in the environment at install time: GitHub tokens, AWS access keys, GCP service account keys, Azure credentials, npm/PyPI publish tokens, HashiCorp Vault tokens, SSH keys.
5. Audit AI coding assistant configuration files (`.claude/settings.json`, `.cursor/rules/`, `.vscode/tasks.json`, `.github/copilot-instructions.md`, `.aider.conf.yml`) for unexpected hooks or injected content.
6. Audit npm and PyPI publish history for your organization's packages to detect any unauthorized versions published via worm propagation.
7. Check CI/CD logs for unexpected outbound connections (e.g., to `github.com/oven-sh/bun/releases/`, or attacker exfil endpoints).

---

## Lessons Learned

- **Semantic versioning coverage**: Inserting patch releases into every major version series simultaneously maximizes reach — organizations pinned to any supported major version are exposed regardless of which major they're on.
- **binding.gyp is a blind spot**: Most npm security monitoring focuses on `scripts.postinstall`. The Phantom Gyp technique (command substitution in binding.gyp targets) bypasses this control without touching package.json lifecycle scripts.
- **Bun evades Node.js hooking**: Using Bun as the execution runtime defeats monitoring tools that rely on Node's `--require` hook interception.
- **Platform engineering tools are high-value targets**: Backstage plugins are installed by platform teams who typically have broad credentials (cloud admin, LDAP service accounts, CI/CD tokens), making them extremely high-value compromise targets.
- **AI assistant config files are a persistent backdoor vector**: The `infectHost` pattern targeting Claude Code, Cursor, Copilot, VS Code, and others demonstrates that developer workstations — not just CI runners — are now first-class targets in supply chain campaigns.

---

## Related Incidents

- [Miasma v2 — binding.gyp Worm (Jun 2026)](./2026-06-miasma-v2-binding-gyp.md)
- [Leo Platform npm Miasma (Jun 2026)](./2026-06-leo-platform-npm-miasma.md)
- [Miasma Red Hat npm (Jun 2026)](./2026-06-miasma-redhat-npm.md)
- [Miasma Azure Repo Injection (Jun 2026)](./2026-06-miasma-azure-repo-injection.md)
- [GitHub Actions Miasma Tag Hijacking (Jun 2026)](./2026-06-github-actions-miasma-tag-hijacking.md)
