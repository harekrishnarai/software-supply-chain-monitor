# Bitwarden CLI Shai-Hulud Third Coming — npm CI/CD Pipeline Compromise, MCP-Aware Credential Worm

**Date:** April 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** CI/CD pipeline compromise / Credential stealer / Supply chain worm
**Sources:**
- [Aikido Security — Is Shai-Hulud Back? Compromised Bitwarden CLI Contains a Self-Propagating npm Worm](https://www.aikido.dev/blog/shai-hulud-npm-bitwarden-cli-compromise)
- [Socket — Bitwarden CLI Compromised in Ongoing Checkmarx Supply Chain Campaign](https://socket.dev/blog/bitwarden-cli-compromised)
- [The Hacker News — Self-Propagating Supply Chain Worm Hijacks npm Packages to Steal Developer Tokens](https://thehackernews.com/2026/04/self-propagating-supply-chain-worm.html)

---

## Summary

On April 23, 2026, Aikido Security identified that `@bitwarden/cli@2026.4.0` — the official command-line interface for the Bitwarden password manager, with ~78,000 weekly downloads — had been compromised with a multi-stage credential theft and supply chain worm. The malware explicitly names itself **"Shai-Hulud: The Third Coming"**, a direct callback to the earlier Shai-Hulud npm worm campaigns from September and November 2025, and is laced with Dune universe theming throughout.

The attacker bypassed Bitwarden's trusted publishing controls by compromising Bitwarden's own CI/CD pipeline (`publish-ci.yml` in `github.com/bitwarden/clients`), allowing a malicious package to be published under the legitimate `@bitwarden` npm scope. The attack is linked to the concurrent Checkmarx KICS Docker Hub compromise via identical C2 infrastructure (`audit.checkmarx[.]cx/v1/telemetry`), Dune-vocabulary GitHub exfil repo naming, and the `LongLiveTheResistanceAgainstMachines` commit pattern — indicating a single coordinated campaign, likely by TeamPCP, targeting multiple security tool and developer tool publishers simultaneously.

Notably, the malware is the first documented instance of a supply chain worm explicitly targeting **MCP (Model Context Protocol) configuration files**, collecting both Claude Code's authentication token (`~/.claude.json`) and MCP server configurations (`~/.claude/mcp.json`, `~/.kiro/settings/mcp.json`) — which frequently contain API keys, database connection strings, and credentials for connected services. The malware also actively queries cloud secret manager services (AWS SSM Parameter Store, Secrets Manager; Azure Key Vault; GCP Secret Manager) using ambient cloud credentials, meaning any developer running this on a cloud-connected machine loses their entire secrets infrastructure.

---

## Compromised Artifacts

| Package | Malicious Version | Weekly Downloads | Install Trigger |
|---------|-------------------|-----------------|----------------|
| `@bitwarden/cli` | `2026.4.0` | ~78,000 | `preinstall` hook → `bw_setup.js` |

---

## How It Worked

### Entry Point: Compromised CI/CD Pipeline

The attacker did not exploit a weak maintainer account password — they compromised Bitwarden's automated publishing GitHub Actions workflow (`publish-ci.yml`), allowing the malicious `2026.4.0` to be published under the legitimate `@bitwarden` npm scope with all expected provenance metadata. The CI pipeline compromise is believed to have leveraged credentials or tokens obtained during the concurrent Checkmarx CI/CD pipeline compromise that same week.

### Stage 1: bw_setup.js (Cross-Platform Bootstrapper)

The `preinstall` hook fires automatically on `npm install` before any user interaction:

```json
"preinstall": "node bw_setup.js"
```

`bw_setup.js` is a lightweight cross-platform bootstrapper that:
1. Detects the victim's OS and architecture (macOS arm64/x64, Linux, Windows)
2. Downloads the legitimate Bun JavaScript runtime directly from `github.com/oven-sh/bun` using the OS-appropriate release URL
3. Uses the downloaded Bun binary to execute Stage 2 (`bw1.js`)

Downloading Bun from GitHub's official release page makes the outbound network request appear benign to any network monitoring that allows `*.github.com`.

**SHA256 (bw_setup.js):** `37f34aa3b86db6898065f3ca886031978580a15251f2576f6d24c3b778907336`

### Stage 2: bw1.js (Credential Harvester + Worm)

`bw1.js` is a ~10 MB heavily obfuscated payload executed by Bun. Once deobfuscated, it is a full-featured credential harvester and Shai-Hulud propagation engine containing:

- An **anti-AI manifesto** (written to victim shell config files)
- An **ideological "resistance" string** embedded throughout
- Hardcoded Dune vocabulary for naming exfiltration repos
- The string **"Shai-Hulud: The Third Coming"** as the GitHub repo description

**SHA256 (bw1.js):** `18f784b3bc9a0bcdcb1a8d7f51bc5f54323fc40cbd874119354ab609bef6e4cb`

#### What It Steals

The malware targets a hardcoded list of high-value credential files:

| Target | Description |
|--------|-------------|
| `~/.ssh/id*`, `~/.ssh/id_*` | SSH private keys |
| `~/.ssh/known_hosts` | SSH fingerprints |
| `~/.aws/credentials` | AWS access keys |
| `~/.config/gcloud/credentials.db` | GCP credentials |
| `~/.npmrc`, `.npmrc` | npm auth tokens |
| `~/.claude.json`, `.claude.json` | Claude Code authentication token |
| `~/.claude/mcp.json` | Claude Code MCP server configs (may contain API keys, DB credentials) |
| `~/.kiro/settings/mcp.json` | Kiro IDE MCP server configs |
| `.git/config` | Git remote URLs and embedded credentials |
| `.git-credentials` | Stored Git passwords |
| `.env` | Project environment variables and API keys |
| `~/.bash_history`, `~/.zsh_history` | Shell history |

Beyond files, the malware also runs **live collectors** against cloud secret management services using ambient credentials already present on the machine:
- AWS SSM Parameter Store (`get-parameters-by-path`, decrypted)
- AWS Secrets Manager (`list-secrets` + `get-secret-value`)
- Azure Key Vault (enumerates vaults, retrieves all secrets)
- GCP Secret Manager (`list + access latest version`)

A developer running this on any cloud-connected workstation or CI runner loses their entire secrets infrastructure, not just filesystem credentials.

#### Exfiltration Channels

Stolen data is exfiltrated via two channels:

| Channel | Endpoint | Notes |
|---------|----------|-------|
| Primary C2 | `hxxps://audit.checkmarx[.]cx/v1/telemetry` | Same endpoint as concurrent Checkmarx KICS Docker attack; `.cx` = Christmas Island TLD, not official Checkmarx domain |
| Fallback C2 | GitHub commit dead-drop | Malware searches `api.github.com/search/commits?q=beautifulcastle%20` for a signed replacement hostname if primary C2 is down |

#### Shai-Hulud Worm Propagation

Following credential theft, the malware propagates using the classic Shai-Hulud pattern:

1. **GitHub staging repos:** Creates a public repo under the victim's GitHub account using a Dune-vocabulary name (e.g., `fremen-sandworm-441`, `harkonnen-melange-7`), with the description **"Shai-Hulud: The Third Coming"**, and commits encrypted exfiltration results. Commit messages use the pattern `LongLiveTheResistanceAgainstMachines:<encoded-token>`.

2. **Token sharing (public dead-drop):** For victims without org membership, the stolen GitHub token is also published into a public GitHub commit dead-drop where other infected machines can discover and reuse it to fund their own exfiltration repos. Victims with org membership have tokens kept within the encrypted exfil data only.

3. **npm propagation:** If npm publish tokens are found, the worm identifies all packages the victim can publish and re-injects itself, bumping patch versions and republishing — identical to the CanisterWorm and CanisterSprawl technique.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 22, 2026 | Checkmarx KICS Docker Hub and VS Code extensions compromised (same campaign, same C2 infrastructure) |
| Apr 22–23, 2026 | Bitwarden CI/CD pipeline (`publish-ci.yml`) compromised; `@bitwarden/cli@2026.4.0` published to npm |
| Apr 23, 2026 | Aikido Security identifies malicious `@bitwarden/cli@2026.4.0` and publishes disclosure |
| Apr 23, 2026 | Socket publishes linked analysis connecting Bitwarden compromise to the Checkmarx campaign |
| Apr 23, 2026 | npm unpublishes `@bitwarden/cli@2026.4.0`; Bitwarden issues security advisory |

---

## Detection

```bash
# Check installed @bitwarden/cli version
npm ls -g @bitwarden/cli 2>/dev/null || npm ls @bitwarden/cli 2>/dev/null

# If version is 2026.4.0, the system is compromised
# Immediately check for the malicious preinstall files:
ls "$(npm root -g)/@bitwarden/cli/bw_setup.js" 2>/dev/null
ls "$(npm root -g)/@bitwarden/cli/bw1.js" 2>/dev/null

# Hash verification of the malicious files:
sha256sum "$(npm root -g)/@bitwarden/cli/bw_setup.js" 2>/dev/null
# MALICIOUS: 37f34aa3b86db6898065f3ca886031978580a15251f2576f6d24c3b778907336

sha256sum "$(npm root -g)/@bitwarden/cli/bw1.js" 2>/dev/null
# MALICIOUS: 18f784b3bc9a0bcdcb1a8d7f51bc5f54323fc40cbd874119354ab609bef6e4cb

# Check for outbound connections to C2 in network logs
grep -r "audit\.checkmarx\.cx\|beautifulcastle" /var/log/ 2>/dev/null

# Check for the anti-AI manifesto in shell config files:
grep -l "LongLiveTheResistanceAgainstMachines\|resistance.*machines\|anti.*AI" \
  ~/.bashrc ~/.zshrc ~/.bash_profile ~/.profile 2>/dev/null

# Check for Shai-Hulud GitHub staging repos in your GitHub account
# Pattern: Dune vocabulary name, description "Shai-Hulud: The Third Coming"
gh repo list --limit 100 --json name,description 2>/dev/null | \
  grep -i "shai.hulud\|third.coming"

# Broader check: look for recently created public repos with Dune vocabulary
# (shared pattern with Checkmarx KICS campaign)
gh repo list --limit 100 --json name,description,createdAt 2>/dev/null | \
  grep -i "melange\|sandworm\|harkonnen\|fremen\|atreides\|gesserit\|checkmarx.configuration"

# Check for GitHub commit dead-drop pattern
gh api /repos/{owner}/{repo}/commits --jq '.[].commit.message' 2>/dev/null | \
  grep "LongLiveTheResistanceAgainstMachines"

# Check if @bitwarden/cli was installed globally via npm
npm list -g --depth=0 2>/dev/null | grep bitwarden

# Check for Bun binary in unusual locations
find / -name "bun" -type f -newer /tmp 2>/dev/null | grep -v "\.bun\|homebrew\|nix"
```

---

## Remediation

1. **Remove the compromised package:**
   ```bash
   npm uninstall -g @bitwarden/cli
   # Reinstall only from a verified clean version:
   npm install -g @bitwarden/cli@2026.5.0  # or latest verified safe version
   ```

2. **Treat the machine as fully compromised.** Rotate in priority order:
   - **GitHub personal access tokens** — check for unauthorized repos and revoke all active tokens at github.com/settings/tokens
   - **npm auth tokens** — revoke at npmjs.com → Account Settings → Access Tokens; audit all packages you can publish for unauthorized new versions
   - **SSH keys** — revoke from GitHub, GitLab, and any servers; regenerate
   - **AWS credentials** — check CloudTrail for unusual `GetSecretValue`, `GetParameter`, or unusual API calls; rotate all IAM credentials
   - **GCP service account keys** — audit and rotate
   - **Azure credentials** — audit Key Vault access logs; rotate
   - **Claude Code auth token** (`~/.claude.json`) — sign out and re-authenticate in Claude Code
   - **MCP server configs** (`~/.claude/mcp.json`, `~/.kiro/settings/mcp.json`) — audit for exposed API keys; rotate all credentials referenced in MCP configs

3. **Remove any shell config injections:**
   ```bash
   # Check for and remove anti-AI manifesto / malicious injections:
   grep -n "LongLiveTheResistanceAgainstMachines\|resistance" ~/.bashrc ~/.zshrc 2>/dev/null
   # Edit the file to remove any injected lines
   ```

4. **Audit GitHub account for attacker-created exfil repos:**
   - Search your account for repos with description "Shai-Hulud: The Third Coming" or "Checkmarx Configuration Storage"
   - Delete any such repos immediately
   - Review GitHub audit log for unexpected repo creation events

5. **If npm tokens were present:** check all packages you publish for unauthorized patch versions published on or after April 22-23, 2026. Unpublish any unauthorized releases immediately.

6. **Block C2 at the firewall/DNS level:**
   - `audit.checkmarx[.]cx` (not a legitimate Checkmarx domain; `.cx` = Christmas Island TLD)
   - GitHub commit dead-drop: monitor for unusual `api.github.com/search/commits?q=beautifulcastle` queries

---

## Lessons Learned

- **CI/CD pipeline trust is transitive and exploitable at scale:** A compromised build pipeline allows publishing malicious packages under fully trusted scopes (`@bitwarden`) with legitimate provenance metadata — no account phishing required, and all existing users of the package are at risk on the next install
- **MCP configuration files are a new, high-value attack target:** Developer tooling such as Claude Code and Kiro stores credentials for every connected service in a single JSON file; any supply chain attack that executes code on a developer machine now has a clear path to a complete credentials harvest of all AI agent integrations
- **Cloud secret manager APIs amplify credential theft significantly:** A single compromised developer machine with ambient cloud credentials can expose secrets that were never stored on disk, defeating the assumption that keeping secrets "off disk" protects them from malware
- **Worm propagation via npm tokens turns individual compromises into cascade events:** The Shai-Hulud pattern (steal npm token → republish all writable packages) means one infected machine can immediately infect dozens of downstream packages before detection
- **Trusted publishing controls only protect the publishing credential path, not the CI/CD pipeline that generates those credentials:** Bitwarden used npm trusted publishing (provenance attestation), but the attacker bypassed this by compromising the CI workflow itself — signing happens after the attacker's code is already in the build

---

## Related Incidents

- [Checkmarx KICS Docker Hub & VS Code Second Compromise (April 2026)](./2026-04-checkmarx-kics-docker-vscode.md) — Same campaign; identical C2, naming patterns, and commit message format; likely same attacker
- [Shai-Hulud Worm Wave 2 (November 2025)](./2025-late-shai-hulud-worm.md) — Prior Shai-Hulud generation: same propagation pattern, different packages
- [CanisterWorm — TeamPCP npm Worm (March 2026)](./2026-03-canisterworm-npm.md) — TeamPCP's own npm worm using ICP canister exfil
- [CanisterSprawl — pgserve npm Compromise (April 2026)](./2026-04-pgserve-npm-canistersprawl.md) — Concurrent April 2026 npm worm attack with similar propagation logic
- [Checkmarx KICS GitHub Action Compromised (March 2026)](./2026-03-checkmarx-kics-action.md) — TeamPCP's first Checkmarx attack; the foothold for the April campaign
