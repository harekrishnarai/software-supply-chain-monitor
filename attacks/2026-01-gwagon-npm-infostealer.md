# G_Wagon — npm Infostealer Targeting 100+ Crypto Wallets via ansi-universal-ui

**Date:** January 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Infostealer / Multi-stage dropper
**Sources:**
- [Aikido Security — G_Wagon: npm Package Deploys Python Stealer Targeting 100+ Crypto Wallets](https://www.aikido.dev/blog/npm-malware-g-wagon-python-stealer-crypto-wallets)

---

## Summary

On January 23, 2026 at 08:46 UTC, Aikido's malware detection system flagged a package called `ansi-universal-ui` on the npm registry. The description claimed it was "a lightweight, modular UI component system for modern web applications" — entirely fictitious. What Aikido found was a sophisticated multi-stage infostealer that downloads its own Python runtime, executes a heavily obfuscated payload entirely in memory (from version 1.4.0), and exfiltrates browser credentials, cryptocurrency wallet data, cloud credentials, and messaging tokens to an attacker-controlled Appwrite cloud bucket.

The malware was caught in active development: the attacker published 10 versions over two days, iterating through bugs, adding anti-forensics measures, switching C2 endpoints, and obfuscating variable names to mimic legitimate UI library code. The malware names itself "G_Wagon" internally. By version 1.4.0, the Python payload never touched disk — it was fetched as base64, decoded in memory, and piped directly to a Python interpreter via stdin.

A self-dependency trick caused the postinstall hook to fire twice: `ansi-universal-ui@1.3.7` declared `ansi-universal-ui: "^1.3.5"` as a dependency, meaning npm ran the install hook for both the outer and inner package during a single `npm install`.

---

## Compromised Artifacts

| Package | Ecosystem | Malicious Versions | Safe Versions |
|---|---|---|---|
| `ansi-universal-ui` | npm | 1.3.5, 1.3.6, 1.3.7, 1.4.0, 1.4.1 | None — package is entirely malicious; remove completely |

Versions 1.0.0–1.3.4 were the **development/testing phase** (dropper scaffolding without working C2).

---

## How It Worked

### Entry Point — Fake UI Component Package with postinstall Hook

The attacker published a convincing fake npm package with detailed (invented) feature descriptions — including a "Virtual Rendering Engine," "ThemeProvider," and "Layout Compute" — none of which existed. The `postinstall` hook in `package.json` executed `node index.js` on install.

### Phase 1: Development (January 21, versions 1.0.0–1.3.3)

- `v1.0.0` (15:54 UTC): Initial scaffold using npm's `tar` module
- `v1.2.0` (16:03 UTC): Switched to system `tar`, added first self-dependency
- `v1.3.2` (16:09 UTC): Added postinstall hook (no payload yet)
- `v1.3.3` (16:18 UTC): Bug fix on redirect handling

### Phase 2: Weaponization (January 23, versions 1.3.5–1.4.2)

- `v1.3.5` (08:46 UTC): Added working C2 URL (Appwrite bucket), fake branding, removed placeholder text — **first fully weaponized version**
- `v1.3.6` (08:53 UTC): Re-enabled self-dependency for double execution
- `v1.3.7` (09:09 UTC): Added anti-forensics (sanitized log messages; "Setting up Python environment" → "Initializing UI runtime")
- `v1.4.0` (12:27 UTC): Pivoted to Frankfurt C2 server; payload now piped through stdin — **never touches disk**
- `v1.4.1` (12:48 UTC): Added hex-encoded string obfuscation and a decoy `LayoutCompute` UI class; renamed directories (`python_runtime` → `lib_core/renderer`, `setupPython` → `_init_layer`)
- `v1.4.2` (13:06 UTC): Bug fix (v1.4.1 broke Python path due to aggressive renaming)

### Payload Mechanics

On execution, the dropper:
1. Downloaded a standalone Python runtime tarball from the Appwrite C2 bucket
2. Extracted the runtime to a temp directory
3. Fetched the Python infostealer payload (base64-encoded in v1.4.0+)
4. In v1.4.0+: decoded in memory and piped directly to `python -` via stdin — **zero disk artifact for the payload itself**

### Data Theft Targets

**Browser Credentials:**
- Chrome, Edge, Brave (Windows and macOS)
- Windows: terminates browser, spawns new instance with DevTools Protocol, extracts cookies; decrypts saved passwords via Windows Data Protection API (DPAPI)
- macOS: extracts encryption key from Keychain, uses OpenSSL to decrypt login data

**Cryptocurrency Wallets (100+ extensions):**
- MetaMask, Phantom, Coinbase Wallet, Trust Wallet, Ledger Live, Trezor, Exodus
- Full list covers Ethereum, Solana, Cosmos, Polkadot, Cardano, TON, Bitcoin Ordinals, and most major blockchain ecosystems
- Copies the entire extension data directory for each wallet found

**Cloud Credentials:**
- AWS CLI (`~/.aws/`), Azure CLI, Google Cloud SDK credentials
- SSH keys and kubeconfig files

**Messaging Tokens:**
- Discord token theft
- Telegram `tdata` directory
- Steam authentication files

### C2 / Exfiltration

All stolen data was zipped and uploaded to an Appwrite cloud storage bucket. Filenames followed the pattern: `{username}@{hostname}_{browser}_{profile}_{original_file}`.

- **C2 (early versions)**: Appwrite NYC endpoint (redacted in source)
- **C2 (v1.4.0+)**: Appwrite Frankfurt server — Bucket ID `6968ea5600316c128f22`

The Python payload also contained a large base64-encoded, XOR-encrypted Windows DLL. The DLL was injected into browser processes using NT native APIs (`NtAllocateVirtualMemory`, `NtWriteVirtualMemory`, `NtProtectVirtualMemory`, `NtCreateThreadEx`), with a full PE parser locating the `Initialize` export as its entry point.

### Anti-Forensics

- Log messages renamed to sound like legitimate UI library operations
- Internal variables renamed: `pythonCode` → `_texture_data`, `setupPython` → `_init_layer`
- Decoy UI class (`LayoutCompute`) added to make code appear legitimate
- From v1.4.0: Python payload never written to disk (memory-only execution)
- Execution counter written to `~/.gwagon_status` (hidden on Windows)

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2026-01-21 15:54 UTC | `ansi-universal-ui@1.0.0` published — dropper scaffold, no payload |
| 2026-01-21 16:03–16:18 UTC | Versions 1.2.0–1.3.3 published; testing self-dependency and postinstall hook |
| 2026-01-23 08:46 UTC | `v1.3.5` published with live C2 — **first weaponized version**; Aikido detection fires |
| 2026-01-23 08:53–09:09 UTC | `v1.3.6` and `v1.3.7` published (double execution, anti-forensics) |
| 2026-01-23 12:27–13:06 UTC | `v1.4.0`–`v1.4.2` published (Frankfurt C2, memory-only payload, obfuscation) |
| 2026-01-23 | Attacker pushes 3 more versions while Aikido is analyzing |
| 2026-01-28 | Article updated; package removed from npm registry |

---

## Detection

```bash
# Check if ansi-universal-ui is installed
npm ls ansi-universal-ui 2>/dev/null
grep '"ansi-universal-ui"' package-lock.json 2>/dev/null

# Check malicious versions (1.3.5, 1.3.6, 1.3.7, 1.4.0, 1.4.1)
npm ls ansi-universal-ui 2>/dev/null | grep -E "1\.(3\.[5-9]|4\.[01])"

# Check for execution counter artifact (indicates successful execution)
ls -la ~/.gwagon_status 2>/dev/null    # macOS/Linux
# Windows (PowerShell):
# Test-Path "$env:USERPROFILE\.gwagon_status"

# Check for suspicious temp directories (Python runtime extraction)
ls /tmp/lib_core 2>/dev/null
ls /tmp/python_runtime 2>/dev/null
ls %TEMP%\lib_core 2>/dev/null  # Windows

# Check for C2 network activity (Appwrite Frankfurt)
grep -i "appwrite" /var/log/syslog 2>/dev/null
# macOS:
log show --last 7d 2>/dev/null | grep -i "appwrite"

# SHA256 hashes of malicious index.js files
# v1.3.5/1.3.6: ecde55186231f1220218880db30d704904dd3ff6b3096c745a1e15885d6e99cc
# v1.3.7:       eb19a25480916520aecc30c54afdf6a0ce465db39910a5c7a01b1b3d1f693c4c
# v1.4.0:       ff514331b93a76c9bbf1f16cdd04e79c576d8efd0d3587cb3665620c9bf49432
# v1.4.1:       a576844e131ed6b51ebdfa7cd509233723b441a340529441fb9612f226fafe52
# py.py stub (all versions): e25f5d5b46368ed03562625b53efd24533e20cd1d42bc64b1ebf041cacab8941

# Scan installed package hashes
find node_modules/ansi-universal-ui -name "index.js" -exec sha256sum {} \; 2>/dev/null
```

---

## Remediation

1. **Remove the package and purge node_modules**:
   ```bash
   npm uninstall ansi-universal-ui
   rm -rf node_modules
   npm install
   ```

2. **Check for the execution counter file** — if `~/.gwagon_status` exists, assume full compromise and proceed to step 3.

3. **Rotate all browser-saved passwords** immediately for Chrome, Edge, and Brave across all profiles.

4. **Revoke and regenerate tokens** for any cryptocurrency wallets installed as browser extensions (MetaMask, Phantom, Coinbase Wallet, Trust Wallet, Ledger, Trezor, Exodus, and others). Transfer funds to new wallets **from a clean, unaffected machine**.

5. **Rotate cloud credentials**: AWS access keys/secret keys, Azure service principals, GCP service account keys, and any kubeconfig tokens.

6. **Regenerate SSH keys** and update all servers, GitHub, and other SSH-authenticated services with new public keys.

7. **Invalidate Discord, Telegram, and Steam sessions** via account settings.

8. **Block at network layer**: Restrict outbound connections to `cloud.appwrite.io` and `*.appwrite.io` from developer machines, or alert on unexpected connections to these endpoints.

---

## Lessons Learned

- **Fake branding with real feature descriptions lowers suspicion**: Detailed, plausible-sounding feature documentation made `ansi-universal-ui` look like a legitimate UI library at a glance.
- **Self-dependency double-execution is a novel evasion technique**: Declaring a package as its own dependency causes the postinstall hook to fire twice, doubling the chance of successful execution.
- **Memory-only payload execution raises the bar for forensics**: From v1.4.0, no Python payload was ever written to disk — only the compiled dropper existed as a file, making post-incident analysis much harder.
- **Cloud object storage (Appwrite, S3, GCS) as exfiltration C2**: Using legitimate cloud storage services for data exfiltration bypasses domain-based threat intelligence feeds and firewall rules.
- **Live iteration during attack**: The attacker actively improved the malware while Aikido was analyzing it, pushing 3 more versions in real time. This demonstrates the industrialization of npm malware development.

---

## Related Incidents

- [./2026-01-dydx-npm-pypi.md](./2026-01-dydx-npm-pypi.md) — January 2026 npm/PyPI credential stealer targeting the dYdX DeFi ecosystem
- [./2026-01-spellcheckpy-pypi-rat.md](./2026-01-spellcheckpy-pypi-rat.md) — Concurrent PyPI infostealer campaign (same week)
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — npm infostealer campaign targeting Chrome credentials and crypto wallets
- [./2025-04-xrp-supply-chain.md](./2025-04-xrp-supply-chain.md) — Similar crypto SDK compromise with private key exfiltration
