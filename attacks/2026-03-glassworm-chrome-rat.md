# GlassWorm Chrome Extension RAT

**Date:** March 18, 2026
**Ecosystem:** npm / PyPI / GitHub / OpenVSX / Chrome
**Severity:** Critical
**Type:** Multi-stage RAT / Chrome extension malware / Hardware wallet phishing / HVNC
**Sources:**
- [Aikido — GlassWorm Hides a RAT Inside a Malicious Chrome Extension](https://www.aikido.dev/blog/glassworm-chrome-extension-rat)

---

## Summary

Aikido's deep-dive into the GlassWorm payload chain revealed a sophisticated multi-stage remote access trojan that extends far beyond the previously documented Solana C2 loader. The full attack chain includes a Stage 2 data theft framework targeting crypto wallets, developer credentials, and cloud secrets; a persistent WebSocket-based RAT with DHT-based C2 resolution; a force-installed Chrome extension masquerading as "Google Docs Offline" that performs keylogging, cookie theft, and session hijacking; and a hardware wallet phishing binary that impersonates Ledger Live and Trezor to steal recovery phrases.

The RAT uses multiple layers of C2 indirection — Solana blockchain memos, BitTorrent DHT lookups, and Google Calendar invites — making it exceptionally resistant to takedown. At the time of Aikido's disclosure, the C2 infrastructure was still active.

---

## Compromised Artifacts

| Vector | Details |
|--------|---------|
| npm packages | Multiple packages across GlassWorm campaigns (Unicode loader + obfuscated preinstall variants) |
| PyPI packages | Compromised Python packages via account takeover |
| GitHub repos | Hundreds of repositories via force-push campaigns |
| OpenVSX extensions | Malicious VS Code extensions |
| Chrome extension | Force-installed "Google Docs Offline" v1.95.1 |

---

## How It Worked

### Stage 1: Initial Infection & Solana C2 Beacon

GlassWorm uses two loader variants (invisible Unicode and obfuscated preinstall) that both perform the same execution logic: a 10-second startup delay, geofencing (exits if Russian locale detected), rate limiting via `~/init.json`, and Solana blockchain C2 resolution. The loader cycles through 9 public Solana RPC endpoints to fetch the C2 URL from a transaction memo on one of two wallets.

### Stage 2: Credential & Wallet Theft

The Stage 2 payload is a comprehensive data-theft framework that stages all collected data under `%TEMP%\hJxPxpHP\`, zips it, and POSTs to `http://217.69.3[.]152/wall`:

- **Crypto wallets:** Searches `%APPDATA%` and `%LOCALAPPDATA%` for 71 browser extension wallet IDs (MetaMask, Phantom, Coinbase, Exodus, etc.) and standalone wallet data
- **Developer credentials:** Reads `~/.npmrc`, `NPM_TOKEN` env var (validates against npm registry); extracts git credentials and VS Code tokens
- **Cloud secrets:** AWS, GCP, Azure, Docker, Kubernetes configs, SSH keys, Heroku `.netrc`, DigitalOcean `doctl`, Terraform state
- **Seed phrase images:** Copies images with filenames matching crypto/seed keywords from Documents and Desktop

### Stage 3a: Hardware Wallet Phishing

On machines with Ledger Live installed, a .NET WPF binary (`Assaac.exe`) is dropped and persisted via registry Run key. It monitors for USB device connections via WMI event subscription. When a Ledger or Trezor device is plugged in, a phishing window opens displaying a fake configuration error with 24 recovery phrase input fields. The binary kills any real Ledger Live processes in a 1-second loop and intercepts close attempts with a warning about losing crypto assets. Submitted phrases are exfiltrated to `45.150.34[.]158`. CIS-region geofencing excludes Russia, Kazakhstan, and 7 other countries.

### Stage 3b: WebSocket RAT with DHT C2

The persistent RAT (`%APPDATA%\QtCvyfVWKH\index.js`) uses two persistence mechanisms (scheduled task `UpdateApp` + registry Run key) and hooks all termination signals to re-download and restart itself. C2 resolution uses a DHT lookup for pinned public key `3c90fa0e84dd76c94b1468f38ed640945d72bc12`, with Solana memo fallback.

RAT commands include: `start_hvnc`/`stop_hvnc` (hidden remote desktop), `start_socks`/`stop_socks` (WebRTC SOCKS proxy), `reget_log` (browser credential theft), `get_system_info`, and `command` (arbitrary JavaScript execution via `eval()`).

### Malicious Chrome Extension

The RAT force-installs a Chrome extension posing as "Google Docs Offline" (v1.95.1) with its own Solana-based C2 resolution (wallet `DSRUBTziADDHSik7WQvSMjvwCHFsbsThrbbjWMoJPUiW`). Extension capabilities include:

- `domsnapshot` — full page HTML exfiltration
- `getcookies` — cookie theft (filterable by domain)
- `localstoragedump` — all localStorage data
- `take_screenshot` — tab screenshot capture
- `capture_clipboard` — clipboard reading
- `history` — up to 5,000 browser history entries
- `bookmarks` — full bookmark tree export
- `enumeration` — browser/hardware/GPU fingerprinting + extension probing
- `startkeylogger` / `stopkeylogger` / `getkeyloggerdata` — hooks keydown/keyup/keypress/input/change/focus/blur across all pages; flushes every 5 seconds
- Targeted session surveillance with Bybit pre-configured; watches for `secure-token` and `deviceid` cookies

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 18, 2026 | Aikido publishes full GlassWorm payload analysis |
| Mar 18, 2026 | C2 at 45.32.150[.]251 confirmed still active, returning win32 payloads |
| Ongoing | Solana wallets continue to serve C2 URLs via transaction memos |

---

## Detection

```bash
# Check for Stage 2 staging directory
ls -la "$TEMP/hJxPxpHP/" 2>/dev/null || ls -la /tmp/hJxPxpHP/ 2>/dev/null

# Check for Stage 3 RAT artifacts (Windows)
dir "%APPDATA%\QtCvyfVWKH\index.js" 2>nul
dir "%LOCALAPPDATA%\QtCvyfVWKH\AghzgY.ps1" 2>nul

# Check for rate-limiting file
ls -la ~/init.json 2>/dev/null

# Check for Ledger phishing binary
dir "%TEMP%\SKuyzYcDD.exe" 2>nul

# Check for malicious Chrome extension (Windows)
dir "%LOCALAPPDATA%\Google\Chrome\jucku\" 2>nul

# Check for malicious Chrome extension (macOS)
ls -la "/Library/Application Support/Google/Chrome/myextension/" 2>/dev/null

# Check for Node.js runtimes silently downloaded
ls -la "%APPDATA%\_node_x64\node\node.exe" 2>/dev/null

# Check registry persistence (Windows PowerShell)
# Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" | Select UpdateApp, UpdateLedger

# Check scheduled tasks (Windows)
# schtasks /query /tn UpdateApp

# Check for network IOCs
# Monitor for connections to: 45.32.150.251, 217.69.3.152, 217.69.0.159, 45.150.34.158

# Verify file hashes
sha256sum /path/to/suspect/file | grep -E "06fab21d|f171c383|9df62cef|43253a88|4a60afa0|de81eacd|fdba5be3|ee3e4dd5"
```

---

## Remediation

1. **Remove the malicious Chrome extension** — delete the `jucku` (Windows) or `myextension` (macOS) directory from Chrome's profile
2. **Remove RAT persistence:**
   - Delete scheduled task `UpdateApp`
   - Remove registry Run keys `UpdateApp` and `UpdateLedger`
   - Delete `%APPDATA%\QtCvyfVWKH\` and `%LOCALAPPDATA%\QtCvyfVWKH\`
3. **Remove the Ledger phishing binary** — delete `%TEMP%\SKuyzYcDD.exe`
4. **Transfer all crypto funds** to new wallets if any wallet extensions were present on the machine
5. **Rotate all developer credentials** — npm tokens, git credentials, VS Code tokens, cloud provider keys
6. **Rotate all cloud secrets** — AWS, GCP, Azure, SSH keys, Kubernetes configs
7. **Clear browser data** — all saved passwords, cookies, and sessions should be considered compromised
8. **Check browser extension list** — ensure "Google Docs Offline" is the legitimate version (not v1.95.1 from an unknown source)

---

## Lessons Learned

- Solana blockchain memos, BitTorrent DHT, and Google Calendar invites create a triple-layered C2 indirection that is exceptionally difficult to take down
- Force-installed Chrome extensions provide persistent access even after the initial RAT is removed
- Hardware wallet phishing binaries that monitor USB events represent a targeted escalation against crypto holders
- The RAT's self-resurrection capability (signal hooks + re-download) makes traditional process killing insufficient
- CIS-region geofencing across all payload stages is a strong indicator of Russian-origin threat actors

---

## Related Incidents

- [GlassWorm Returns — Unicode Mass Campaign](./2026-03-glassworm-returns.md)
- [GlassWorm / CanisterWorm](./2026-03-glassworm-canisterworm.md)
- [React Native Phone Number Packages Compromise](./2026-03-react-native-compromise.md)
- [ForceMemo](./2026-03-forcememo.md)
