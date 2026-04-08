# Polymarket Bot — Hijacked dev-protocol GitHub Org Wallet Stealer

**Date:** March 15, 2026
**Ecosystem:** npm / GitHub
**Severity:** Critical
**Type:** Organization hijack / Typosquatting / Wallet theft / SSH backdoor
**Sources:**
- [StepSecurity — Malicious Polymarket Bot Hides in Hijacked dev-protocol GitHub Org and Steals Wallet Keys](https://www.stepsecurity.io/blog/malicious-polymarket-bot-hides-in-hijacked-dev-protocol-github-org-and-steals-wallet-keys)

---

## Summary

The StepSecurity threat intelligence team discovered that `dev-protocol` — a verified GitHub organization with 568 followers belonging to a legitimate Japanese DeFi project established in 2019 — was hijacked and repurposed to distribute malicious Polymarket trading bots. The repository `dev-protocol/polymarket-copytrading-bot-sport` featured a polished README, hundreds of stars (bot-inflated), and a trading bot that actually connects to real Polymarket APIs. Hidden inside its npm dependencies were two typosquatted packages that steal wallet private keys, exfiltrate sensitive files, and install an SSH backdoor on the victim's machine.

The attack is particularly insidious because the bot genuinely works — a victim who follows the README instructions sees real Polymarket trading activity, with no visible indication that their wallet keys have already been exfiltrated and their machine has been backdoored.

---

## Compromised Artifacts

| Package | Version | Role |
|---------|---------|------|
| `ts-bign` | 1.2.8 | Wrapper carrying `levex-refa`; impersonates `big.js` |
| `big-nunber` | 5.0.2 | Wrapper carrying `lint-builder`; typosquat of `bignumber.js` |
| `levex-refa` | 1.0.0 | File stealer (J2TEAM obfuscated) |
| `lint-builder` | 1.0.1 | Backdoor installer (anti-tamper obfuscated); runs via postinstall |

**GitHub Organization:** `dev-protocol` (hijacked from legitimate Japanese DeFi project)
**Malicious Repos:** 20+ Polymarket variants committed by user `insionCEO`
**Stars/Forks:** 366/307 (bot-inflated via accounts like `starlight83636-arch`, `cryptohuan52-arch`)

---

## How It Worked

### Organization Hijack

The attackers hijacked the `dev-protocol` GitHub organization, which had been established in April 2019 with a verified badge and 568 real followers. Starting around February 26, 2026, the organization was flooded with 20+ Polymarket scam bot repositories. Of 830 total repositories, 664 are forks created after the hijack. All malicious repos were committed by user `insionCEO`. Security warnings filed by victims (e.g., Issue #6) were actively deleted by the attackers.

### Dependency Chain Attack

The typosquatted packages are both declared in `package.json` AND imported in source code, creating a two-stage attack:

**Chain 1: `ts-bign` → `levex-refa` (File Stealer)**
- `levex-refa` is obfuscated with J2TEAM obfuscator using a rotating string array with hex references
- Gets victim's local IP via UDP socket to 8.8.8.8:80
- Recursively searches CWD for `.env`, `*.env`, `id.json`, `config.toml`, `Config.toml`
- Prepends `username@localIP` header to each file
- POSTs each file to `cloudflareguard[.]vercel[.]app/api/v1` as `application/octet-stream`

**Chain 2: `big-nunber` → `lint-builder` (Backdoor Installer)**
- Anti-tamper obfuscated (causes `Maximum call stack size exceeded` on deobfuscation attempts)
- Auto-executes via `postinstall: "node test.js"` during `npm install`
- Fetches C2 config from `cloudflareinsights[.]vercel[.]app/api/scan-patterns`
- Fetches blocklist from `cloudflareinsights[.]vercel[.]app/api/block-patterns`
- Exfiltrates data to `cloudflareinsights[.]vercel[.]app/api/v1`
- Fingerprints victim IP via `api.ipify.org`
- Takes SSH ownership: `sudo chown -R runner:runner ~/.ssh`
- Enables firewall: `sudo ufw enable`
- Opens SSH port: `sudo ufw allow 22/tcp`

### Source Code Imports Ensure Double Execution

```typescript
// src/constant/index.ts (line 1)
import { from_str } from "ts-bign";
// src/trading/exit.ts (line 3)
import { from_str } from "big-nunber";
```

The `from_str()` function triggers the file stealer, ensuring the malware activates both during `npm install` (via lint-builder's postinstall) AND when the bot runs (via source imports).

### The Bot Actually Works

After stealing credentials and opening the SSH backdoor, the bot connects to real Polymarket APIs (`data-api.polymarket.com`, `clob.polymarket.com`), queries positions, and authenticates normally. A victim sees legitimate trading activity with no indication of compromise.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 2019 | `dev-protocol` GitHub organization created (legitimate DeFi project) |
| ~Feb 26, 2026 | Organization hijacked; Polymarket scam repos start appearing |
| Feb–Mar 2026 | 20+ Polymarket variant repos created; bot farms inflate stars/forks |
| Mar 15, 2026 | StepSecurity discovers and publicly reports the attack |
| Mar 15, 2026 | Victim simulation confirms attack chain via Harden-Runner monitoring |
| Mar 15, 2026 16:29:36 | Stage 1 (npm install): lint-builder postinstall executes; C2 config fetched; SSH backdoor installed |
| Mar 15, 2026 16:29:40 | Stage 2 (bot startup): levex-refa file stealer triggers via source imports; .env exfiltrated |
| Mar 15, 2026 | StepSecurity discloses to GitHub, npm, and dev-protocol organization |

---

## Detection

```bash
# Check for the typosquatted packages
npm ls ts-bign big-nunber levex-refa lint-builder 2>/dev/null

# Check for C2 connections in network logs
# Look for: cloudflareguard.vercel.app, cloudflareinsights.vercel.app

# Check if SSH was backdoored
cat ~/.ssh/authorized_keys  # Look for unauthorized keys
sudo ufw status  # Check if port 22 was opened unexpectedly

# Check for unexpected file access
# Look for processes reading .env, id.json, config.toml files

# Scan node_modules for the malicious packages
find node_modules/ -path "*/levex-refa/*" -o -path "*/lint-builder/*" -o -path "*/ts-bign/*" -o -path "*/big-nunber/*" 2>/dev/null

# Check for the dev-protocol repo
git remote -v | grep "dev-protocol/polymarket"
```

---

## Remediation

1. **Rotate all wallet keys immediately** — transfer funds from any wallet whose private key was in `.env`, `id.json`, or `config.toml`
2. **Check for SSH backdoors:**
   ```bash
   cat ~/.ssh/authorized_keys  # Remove unrecognized keys
   sudo ufw status  # Disable port 22 if not needed
   ```
3. **Revoke all API keys** — Polymarket credentials, exchange keys, and any secrets present on the machine
4. **Remove the malicious packages:**
   ```bash
   npm uninstall ts-bign big-nunber levex-refa lint-builder
   ```
5. **Delete the cloned repository** — do not trust any code from the hijacked `dev-protocol` organization
6. **Use `--ignore-scripts`** for future untrusted installs: `npm install --ignore-scripts`

---

## Lessons Learned

- Hijacking verified GitHub organizations with established history provides instant credibility for social engineering
- Bot-inflated stars and forks create a false sense of legitimacy that is difficult to distinguish at a glance
- Functional malware (a working Polymarket bot) lowers victim suspicion — the tool does what it claims while stealing credentials silently
- C2 domains named to impersonate legitimate services (`cloudflareguard`, `cloudflareinsights`) can evade casual network log review
- Anti-tamper obfuscation in npm packages (lint-builder) defeats static analysis, requiring runtime monitoring for detection
- Attackers actively suppress disclosure by deleting security warning issues

---

## Related Incidents

- [GlassWorm / CanisterWorm](./2026-03-glassworm-canisterworm.md) — Similar Vercel-hosted C2 pattern
- [Info-Stealer Campaign](./2025-08-infostealer-campaign.md) — Similar fake package + wallet theft pattern
- [XRP Ledger Supply Chain Attack](./2025-04-xrp-supply-chain.md) — Similar private key exfiltration from crypto SDKs
