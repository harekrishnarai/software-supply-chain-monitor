# axios npm Compromise — Maintainer Account Hijacked, Cross-Platform RAT Deployed

**Date:** March 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Account Takeover / Postinstall RAT / Dependency Injection
**Sources:**
- [StepSecurity — axios Compromised on npm - Malicious Versions Drop Remote Access Trojan](https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan)
- [Aikido Security — axios compromised on npm: maintainer account hijacked, RAT deployed](https://www.aikido.dev/blog/axios-npm-compromised-maintainer-hijacked-rat)
- [Semgrep — Axios Supply Chain Incident: Indicators of Compromise](https://semgrep.dev/blog/2026/axios-supply-chain-incident-indicators-of-compromise-and-how-to-contain-the-threat)

---

## Summary

On March 31, 2026, attackers compromised the npm account of `jasonsaayman`, the primary maintainer of `axios` — the most popular JavaScript HTTP client library with approximately 100 million weekly downloads. Two malicious versions were published in rapid succession: `axios@1.14.1` at 00:21 UTC and `axios@0.30.4` at 01:00 UTC, targeting both the modern 1.x and legacy 0.x release branches within a 39-minute window. npm unpublished both versions within roughly three hours.

Neither malicious version contains a single line of malicious code inside the axios source itself. Instead, both inject a fake runtime dependency — `plain-crypto-js@4.2.1` — a package never imported anywhere in axios, whose only purpose is to fire a postinstall hook that deploys a cross-platform Remote Access Trojan. The malware fingerprints the OS, contacts a command-and-control server, and delivers a platform-specific second-stage payload for macOS, Windows, and Linux. After execution, the dropper deletes itself and replaces its own `package.json` with a pre-staged clean stub, leaving `node_modules` looking completely unmodified to any post-infection inspection.

The attack was operationally sophisticated and pre-meditated: the malicious dependency was staged 18 hours in advance of the axios releases, three separate payloads were pre-built for three operating systems, and every trace was designed to self-destruct. StepSecurity independently confirmed live C2 callbacks via Harden-Runner instrumentation. This is among the most impactful npm supply chain attacks on record given axios's download volume.

---

## Compromised Artifacts

| Package | Malicious Version | Shasum |
|---------|------------------|--------|
| `axios` | 1.14.1 | `2553649f2322049666871cea80a5d0d6adc700ca` |
| `axios` | 0.30.4 | `d6f3f62fd3b9f5432f5782b62d8cfd5247d5ee71` |
| `plain-crypto-js` (dropper) | 4.2.1 | `07d889e2dadce6f3910dcbc253317d28ca61c766` |

Safe versions: `axios@1.14.0` (shasum: `7c29f4cf2ea91ef05018d5aa5399bf23ed3120eb`), `axios@0.30.3`

Exposure window:
- `axios@1.14.1`: ~2h 54m (2026-03-31 00:21 UTC — 03:15 UTC)
- `axios@0.30.4`: ~2h 15m (2026-03-31 01:00 UTC — 03:15 UTC)
- `plain-crypto-js@4.2.1`: ~4h 27m (2026-03-30 23:59 UTC — 2026-03-31 04:26 UTC)

---

## How It Worked

### Step 1 — Maintainer Account Hijack

The attacker compromised the `jasonsaayman` npm account, the primary maintainer of the axios project, and changed the account email to `ifstap@proton.me` — an attacker-controlled ProtonMail address. Using a stolen long-lived npm access token, the attacker published the poisoned packages manually via the npm CLI, completely bypassing the project's normal GitHub Actions CI/CD pipeline.

A critical forensic signal is visible in the npm registry metadata: every legitimate axios 1.x release is published via GitHub Actions with npm's OIDC Trusted Publisher mechanism, cryptographically tying the publish to a verified workflow. `axios@1.14.1` breaks this pattern — it has no `trustedPublisher` field, no `gitHead`, and no corresponding commit or tag in the GitHub repository. The OIDC token used in legitimate releases is ephemeral and workflow-scoped; it cannot be stolen. The manual publish via stolen classic access token is the definitive forensic indicator that the release is illegitimate.

### Step 2 — Pre-Staging the Malicious Dependency

Before publishing the backdoored axios releases, the attacker pre-staged `plain-crypto-js` on npm using a separate throwaway account (`nrwise`, `nrwise@proton.me`). Both attacker-controlled accounts used ProtonMail — a consistent operational pattern.

The pre-staging strategy was deliberate:
- `plain-crypto-js@4.2.0` (clean decoy) was published first at 2026-03-30 05:57 UTC to establish publishing history and avoid "zero-history account" alarms from security scanners
- `plain-crypto-js@4.2.1` (malicious) was published ~18 hours later at 2026-03-30 23:59 UTC
- The package was designed to masquerade as `crypto-js` — same description, same author attribution (Evan Vosberg), same repository URL pointing to `github.com/brix/crypto-js`
- A clean `package.md` stub (version 4.2.0, no postinstall) was bundled inside, ready to overwrite the real `package.json` after execution to erase forensic evidence

### Step 3 — Dependency Injection into axios

Both malicious axios versions add `plain-crypto-js: "^4.2.1"` as a runtime dependency — a package that has never appeared in any legitimate axios release. The diff is surgical: every other dependency is identical to the prior clean version. A grep across all 86 files in `axios@1.14.1` confirms `plain-crypto-js` is never imported or `require()`'d anywhere in the source code.

When a developer runs `npm install axios@1.14.1`, npm resolves and installs `plain-crypto-js@4.2.1` automatically and executes its postinstall script.

### Step 4 — RAT Dropper: setup.js

The postinstall hook executes `node setup.js`. The dropper uses a two-layer obfuscation scheme:

**Obfuscation Layer 1 (`_trans_1`):** XOR cipher using the key `"OrDeR_7077"` — only the digits at positions 6–9 (`7, 0, 7, 7`) survive JavaScript's bitwise coercions, giving an effective key of `[0,0,0,0,0,0,7,0,7,7]`. Each character at position `r` is decoded as `charCode XOR key[(7 × r × r) % 10] XOR 333`.

**Obfuscation Layer 2 (`_trans_2`):** Reverses the encoded string, replaces `_` with `=`, base64-decodes, then passes through `_trans_1`.

All sensitive strings — module names, OS identifiers, shell commands, the C2 URL, and filesystem paths — are stored in an array `stq[]` and decoded at runtime.

The dropper entry point is `_entry("6202033")`, where `6202033` is the C2 URL path segment. Full C2 URL: `http://sfrclak.com:8000/6202033`

### Step 5 — Platform-Specific Payloads

The dropper checks `os.platform()` and branches into one of three attack paths:

**macOS (darwin):** Writes an AppleScript file to the system temp directory and executes it via `nohup osascript`. The script contacts the C2 with POST body `packages.npm.org/product0`, downloads the macOS RAT binary to `/Library/Caches/com.apple.act.mond`, makes it executable (`chmod 770`), and launches it via `/bin/zsh`. The path deliberately mimics Apple's reverse-DNS daemon naming convention to blend in with legitimate Apple background processes. The AppleScript is deleted after execution.

**Windows (win32):** Three-stage chain — (1) locates PowerShell binary path via `where powershell`; (2) copies the PowerShell binary to `%PROGRAMDATA%\wt.exe` (disguised as Windows Terminal); (3) writes a VBScript to `%TEMP%\6202033.vbs` and executes it via `cscript //nolog`. The VBScript runs a fully hidden `cmd.exe` window that curls the C2 with POST body `packages.npm.org/product1` to retrieve a PowerShell RAT script, saved to `%TEMP%\6202033.ps1`, executed with `-ExecutionPolicy Bypass -WindowStyle Hidden`. Both temp files self-delete; the only persistent artifact is `%PROGRAMDATA%\wt.exe`.

**Linux (all other):** Executes a direct shell command via Node.js `execSync`: curls the C2 with POST body `packages.npm.org/product2` to `/tmp/ld.py`, then launches it with `nohup python3`. The Python RAT is not self-deleted and persists at `/tmp/ld.py`.

All three POST body strings use the `packages.npm.org/` prefix to make outbound traffic appear as benign npm registry communication in network logs and SIEM rules at a glance. Note: `packages.npm.org` is not the npm registry — it belongs to the National Association of Pastoral Musicians.

Once running, the RAT beacons system information and file listings to `sfrclak.com:8000` every 60 seconds.

### Step 6 — Anti-Forensics (Evidence Swap)

After launching the platform payload, `setup.js` performs three cleanup steps:
1. **Deletes itself** — `fs.unlink(__filename)` removes `setup.js`
2. **Deletes `package.json`** — removes the file containing the incriminating `"postinstall": "node setup.js"` hook
3. **Renames `package.md` → `package.json`** — the pre-staged clean stub (version 4.2.0, no postinstall) is moved into place

Any post-infection inspection of `node_modules/plain-crypto-js/package.json` shows a completely clean manifest. The 36-second gap between the two `package.json` writes (captured by Harden-Runner's kernel-level file event monitoring) is the only runtime artifact confirming the swap occurred.

The RAT process is orphaned to PID 1 via `nohup ... &`, severing all parent-child relationships and evading process attribution — confirmed by StepSecurity's live Harden-Runner capture showing `ppid: 1` for the background Python process, which continued beaconing across subsequent CI workflow steps.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-03-30 05:57 | `plain-crypto-js@4.2.0` (clean decoy) published by attacker account `nrwise@proton.me` to establish npm publishing history |
| 2026-03-30 23:59 | `plain-crypto-js@4.2.1` (malicious dropper) published by `nrwise@proton.me` |
| 2026-03-31 00:21 | `axios@1.14.1` published by compromised `jasonsaayman` account (email: `ifstap@proton.me`); targets modern 1.x users |
| 2026-03-31 01:00 | `axios@0.30.4` published by same compromised account; targets legacy 0.x users — both branches hit within 39 minutes |
| 2026-03-31 ~03:15 | npm unpublishes `axios@1.14.1` and `axios@0.30.4`; `latest` dist-tag reverts to 1.14.0 |
| 2026-03-31 03:25 | npm initiates security hold on `plain-crypto-js` |
| 2026-03-31 04:26 | npm publishes security-holder stub `plain-crypto-js@0.0.1-security.0`, formally replacing the malicious package |
| 2026-03-31 | StepSecurity and Aikido Security publish full technical analyses with IOCs |

---

## Detection

```bash
# ── STEP 1: Check for malicious axios versions ──────────────────────────────

# Installed package tree
npm list axios 2>/dev/null | grep -E "1\.14\.1|0\.30\.4"

# Lock file check
grep -A1 '"axios"' package-lock.json | grep -E "1\.14\.1|0\.30\.4"

# ── STEP 2: Check for the hidden dropper package ─────────────────────────────
# Even if setup.js self-deleted, the directory still exists.
# Its presence alone confirms the dropper ran.
ls node_modules/plain-crypto-js 2>/dev/null && echo "POTENTIALLY AFFECTED"

# ── STEP 3: Check for RAT artifacts on disk ─────────────────────────────────

# macOS
ls -la /Library/Caches/com.apple.act.mond 2>/dev/null && echo "COMPROMISED"

# Linux
ls -la /tmp/ld.py 2>/dev/null && echo "COMPROMISED"

# Windows (cmd.exe)
# dir "%PROGRAMDATA%\wt.exe" 2>nul && echo COMPROMISED

# ── STEP 4: Check for active C2 connections ──────────────────────────────────

# macOS / Linux
lsof -i -nP | grep sfrclak

# Linux — also check for process beaconing every 60s
ss -tnp | grep 142.11.206.73

# Windows
# netstat -ano | findstr "142.11.206.73"

# ── STEP 5: Check CI/CD pipeline logs ────────────────────────────────────────
grep -r "sfrclak" ~/.npm/_logs/ 2>/dev/null

# ── STEP 6: Verify shasum of any cached tarballs ─────────────────────────────
# Malicious axios@1.14.1 shasum: 2553649f2322049666871cea80a5d0d6adc700ca
# Malicious axios@0.30.4 shasum: d6f3f62fd3b9f5432f5782b62d8cfd5247d5ee71
# Malicious plain-crypto-js@4.2.1 shasum: 07d889e2dadce6f3910dcbc253317d28ca61c766
npm cache verify 2>/dev/null | grep -E "axios|plain-crypto-js"

# ── STEP 7: Check private registries / CI cache ──────────────────────────────
# If your org runs JFrog Artifactory, Nexus, Verdaccio, etc., search for and
# delete cached copies of axios@1.14.1, axios@0.30.4, plain-crypto-js@4.2.1.
# Also invalidate any Docker image layers or node_modules cache keys created
# during 2026-03-30 23:00 UTC – 2026-03-31 04:00 UTC.
```

---

## Remediation

1. **Pin to safe versions immediately:**
   ```
   npm install axios@1.14.0   # 1.x users
   npm install axios@0.30.3   # 0.x users
   ```
2. **Add overrides to prevent transitive resolution to malicious versions:**
   ```json
   {
     "dependencies": { "axios": "1.14.0" },
     "overrides": { "axios": "1.14.0" },
     "resolutions": { "axios": "1.14.0" }
   }
   ```
3. **Remove the dropper package and reinstall without scripts:**
   ```
   rm -rf node_modules/plain-crypto-js
   npm install --ignore-scripts
   ```
4. **If a RAT artifact is present (`com.apple.act.mond`, `wt.exe`, `ld.py`): do not attempt to clean in place.** Treat the system as fully compromised. Rebuild from a known-good state.
5. **Rotate all credentials accessible on the affected system:** npm tokens, AWS access keys, SSH private keys, CI/CD secrets, `.env` values, cloud credentials (GCP, Azure).
6. **Audit CI/CD pipeline logs** for any runs that installed `axios@1.14.1` or `axios@0.30.4` during 2026-03-30 23:00 – 2026-03-31 04:00 UTC. Rotate all injected secrets from any affected workflow.
7. **Clean package manager caches** on all affected machines and runners:
   ```
   npm cache clean --force
   yarn cache clean        # if using Yarn
   pnpm store prune        # if using pnpm
   ```
8. **Block C2 at the network/DNS layer** as a precaution on any potentially exposed system:
   ```bash
   # Linux firewall
   iptables -A OUTPUT -d 142.11.206.73 -j DROP
   # /etc/hosts (macOS/Linux)
   echo "0.0.0.0 sfrclak.com" >> /etc/hosts
   ```
9. **Adopt `npm ci --ignore-scripts`** as a standing CI/CD policy to prevent postinstall hooks from executing during automated builds.

---

## Lessons Learned

- **OIDC Trusted Publishing is a critical forensic signal.** Legitimate axios releases are tied to a verified GitHub Actions workflow via OIDC. A manual CLI publish with a stolen classic npm token breaks this pattern immediately and should trigger automated alerts from any supply chain monitoring tool.
- **A dependency with zero imports in the codebase is a high-confidence IOC.** `plain-crypto-js` appears in `package.json` but is never `require()`'d anywhere across 86 files — this is trivially detectable by static analysis and should be automated.
- **Pre-staging malicious packages days in advance defeats "brand-new package" heuristics.** Maintaining a publish-age minimum for newly introduced dependencies (e.g., 48h cooldown) is a practical mitigation.
- **Self-destructing malware makes post-infection forensics unreliable.** Runtime kernel-level monitoring (process trees, file events, network events) is the only reliable way to detect and attribute these attacks after the fact. Examining `node_modules` after install is insufficient.
- **The blast radius scales with download count.** 100 million weekly downloads means even a 3-hour exposure window can touch millions of developer machines and CI/CD pipelines globally.
- **IDE extensions can trigger postinstall hooks independently of lockfiles.** Extensions that auto-resolve dependencies (e.g., NX Console for VSCode) can pull transitive deps and run postinstall scripts even when `package-lock.json` pins a safe version.
- **The `packages.npm.org/` POST body prefix is a deliberate SIEM evasion technique.** Network rules should match on actual `packages.npm.org` ownership, not just the string prefix.

---

## Related Incidents

- [./2026-01-dydx-npm-pypi.md](./2026-01-dydx-npm-pypi.md) — dYdX npm/PyPI compromise; C2 pre-staged 18 days before attack (same pre-staging pattern)
- [./2026-03-canisterworm-npm.md](./2026-03-canisterworm-npm.md) — CanisterWorm/TeamPCP npm campaign; postinstall-based propagation
- [./2026-02-cline-openclaw.md](./2026-02-cline-openclaw.md) — Cline OpenClaw; postinstall-based persistent agent deployment
- [./2025-04-xrp-supply-chain.md](./2025-04-xrp-supply-chain.md) — XRP Ledger SDK; compromised official npm package via maintainer account
