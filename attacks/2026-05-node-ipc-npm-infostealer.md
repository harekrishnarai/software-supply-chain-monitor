# node-ipc npm Infostealer — DNS-Exfiltration Credential Stealer in Widely-Used IPC Library

**Date:** May 2026
**Ecosystem:** npm
**Severity:** High
**Type:** Account Takeover / Infostealer / DNS Exfiltration
**Sources:**
- [Semgrep — Not Your IPC, but node-ipc: npm Hit Again with Supply Chain Attack (But This Time It's Not a Worm)](https://semgrep.dev/blog/2026/not-your-ipc-but-node-ipc-npm-hit-again-with-supply-chain-attack-but-this-time-its-not-a-worm)

---

## Summary

Malicious versions of `node-ipc` — a widely-used npm package for inter-process communication — were published to the npm registry and remained available for approximately 2 hours before removal. The compromised versions contained an obfuscated infostealer that fingerprints the host environment, enumerates and reads local files covering every major cloud provider, developer tooling, and AI coding assistant configuration, compresses the collected data, wraps it in a cryptographic envelope, and exfiltrates it exclusively through DNS queries to a custom DNS server. The malware leaves no persistent files on disk after execution and no lingering processes — it fires once, grabs everything reachable, and disappears.

This is not the first compromise of `node-ipc`. In 2022, the package maintainer deliberately inserted "protestware" that wiped files and replaced them with a heart emoji on systems with Russian or Belarusian IP addresses in protest of the Ukraine-Russia war. The 2026 incident differs: this is a straightforward credential theft campaign, not maintainer-driven protestware. The malware targets an exceptionally broad set of credentials including cloud providers (AWS, Azure, GCP, OCI, DigitalOcean, Fly.io, Vercel, Railway, Snowflake), developer infrastructure (SSH, Git, npm, PyPI, Kubernetes, Docker, Terraform, Ansible), SaaS tools (Salesforce, Microsoft 365/PowerApps, GitHub/GitLab), and AI coding tools (Claude, Kiro IDE). DNS-only exfiltration is notable for its stealth: DNS queries are rarely blocked by developer machine or CI/CD egress policies, and individual queries look like background noise.

---

## Compromised Artifacts

| Package | Malicious Version |
|---------|------------------|
| `node-ipc` | 9.1.6 |
| `node-ipc` | 9.2.3 |
| `node-ipc` | 12.0.1 |

---

## How It Worked

### Entry Point

The malicious code was injected into `node-ipc.cjs` at line 1271. It executes when the module is loaded.

### Execution Flow

1. **Environment fingerprinting**: The malware identifies the host OS, installed tools, and available credential file paths before collection
2. **File enumeration and collection**: Reads a comprehensive list of credential file paths (see IOCs below), collecting any that exist on the host
3. **Staging**: Collected files are staged in a `fixtures/` directory using the naming scheme `fixtures/f_{sha256hash}_{filename}` with paths enumerated in `fixtures/_paths.txt`
4. **Compression and encryption**: Collected data is compressed and wrapped in a cryptographic envelope — only the attacker can decrypt the exfiltrated data
5. **DNS exfiltration**: Encoded data is transmitted exclusively via DNS queries to a custom DNS server (`sh.azurestaticprovider.net`), routed through `bt.node.js`
6. **Cleanup**: After exfiltration, the staging files are deleted and no processes are left running

The DNS-only exfiltration is specifically designed for stealth: there is no HTTPS C2 connection to block, no persistent process to detect, and DNS traffic to `azurestaticprovider.net` is crafted to appear as legitimate Azure infrastructure traffic.

### Credential Targets

The malware targets an exceptionally wide range of credential files across categories:

**Cloud Providers**: AWS (`~/.aws/credentials`, `~/.aws/config`, SSO cache), Azure (`~/.azure/accessTokens.json`, MSAL token cache, Azure Developer CLI), GCP (`~/.config/gcloud/credentials.db`, application default credentials, legacy credentials), OCI, Alibaba Cloud, IBM Bluemix, DigitalOcean, Hetzner, Scaleway, Linode, Fly.io, Vercel, Railway, Snowflake, Doppler

**Secrets Management**: All `.env`, `.env.local`, `.env.production` files

**Developer Tools**: SSH keys and config (`~/.ssh/id*`, `~/.ssh/config`), Git credentials (`~/.gitconfig`, `.git-credentials`), package manager tokens (`~/.npmrc`, `~/.pypirc`, `~/.yarnrc`), container configs (`~/.docker/config.json`, Podman auth), Kubernetes (`~/.kube/config`, in-cluster service account tokens, k3s config, GitLab runner config, Helm)

**IaC**: Terraform credentials, `terraform.tfvars`, Ansible configs

**SaaS/CRM**: Salesforce CLI (`~/.sf/*`, `~/.sfdx/*`), Microsoft 365 and PowerApps CLI configs, Microsoft Teams

**AI Coding Tools**: Claude (`~/.claude.json`, `~/.claude/mcp.json`), Kiro IDE (`~/.kiro/settings/mcp.json`)

**Shell and History**: `~/.bash_history`, `~/.zsh_history`, `~/.mysql_history`, `~/.psql_history`, `~/.python_history`, `~/.node_repl_history`

**CI/CD Workflows**: GitHub Actions workflow YAML files, GitLab CI configs

**macOS Additional**: Keychain databases (`~/Library/Keychains/*.keychain-db`), Firefox key databases, shell sessions, VPN configs

---

## Timeline

| Date | Event |
|------|-------|
| May 14, 2026 | Malicious versions published (approx. 2-hour exposure window) |
| May 14, 2026 | Semgrep advisory published; versions flagged and removed |

---

## Detection

```bash
# Check installed version
npm ls node-ipc
# Affected versions: 9.1.6, 9.2.3, 12.0.1

# Check for staging artifacts (may already be cleaned up)
ls -la fixtures/
ls -la fixtures/_paths.txt 2>/dev/null
find . -name "f_*" -path "*/fixtures/*" 2>/dev/null

# Check for injection point in installed module
grep -n "azurestaticprovider\|bt\.node\.js" node_modules/node-ipc/node-ipc.cjs 2>/dev/null

# Monitor DNS traffic for exfiltration queries:
# Look for DNS queries to: bt.node.js (via sh.azurestaticprovider.net)
# DNS exfil queries will contain encoded credential data as subdomains

# Check npm audit for advisory
npm audit | grep node-ipc

# Semgrep advisory reference:
# https://semgrep.dev/orgs/-/advisories/ssc-a83754ff-9ed1-42bb-b6b0-dd65980b1fb4
```

---

## Remediation

1. **Update or remove `node-ipc`**: Pin to a version not in the malicious list (9.1.6, 9.2.3, 12.0.1)
   ```bash
   npm install node-ipc@latest  # or pin to a specific clean version
   ```
2. **Rotate all credentials** listed in the IOCs section — the malware is broad and targets cloud credentials, SSH keys, GitHub/GitLab tokens, Claude configs, Salesforce, M365, and more
3. **macOS users**: Also rotate Keychain credentials and Firefox-stored passwords if the compromised version ran on a macOS machine
4. **Audit DNS logs** if available for queries to `bt.node.js` / `sh.azurestaticprovider.net` during the exposure window
5. **Check `fixtures/` directory** for any staging artifacts (may have been cleaned, but worth checking)
6. **Review npm access tokens**: `npm token list` and revoke any tokens you don't recognize
7. **Scan CI/CD workflows** that may have installed the compromised version and rotate all secrets accessible in those environments

---

## Lessons Learned

- DNS-only exfiltration is extremely difficult to detect and block: developers and CI/CD pipelines rarely restrict DNS traffic, and individual queries look like routine background noise
- The 2-hour exposure window is enough for wide distribution given `node-ipc`'s popularity in the npm ecosystem
- AI coding assistant configs (`~/.claude.json`, `~/.claude/mcp.json`, `~/.kiro/settings/mcp.json`) are now explicitly targeted — attackers understand that MCP server configurations contain API keys and sensitive tool integrations
- Historical compromises of a package (like the 2022 `node-ipc` protestware) signal that it is a high-value target for future attacks, both by the original maintainers and by outside attackers who gain access to publishing credentials
- The complete lack of on-disk persistence and process artifacts after execution makes forensic analysis difficult — defenders need real-time DNS monitoring rather than post-incident artifact analysis

---

## Related Incidents

- [./2026-05-antv-npm-shai-hulud.md](./2026-05-antv-npm-shai-hulud.md) — Concurrent TeamPCP npm wave with similar credential sweep scope
- [./2026-04-pgserve-npm-canistersprawl.md](./2026-04-pgserve-npm-canistersprawl.md) — npm infostealer with ICP blockchain exfiltration
- [./2026-01-gwagon-npm-infostealer.md](./2026-01-gwagon-npm-infostealer.md) — npm infostealer targeting crypto wallets and cloud keys
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — Chrome credential theft via fake React/Solana packages
