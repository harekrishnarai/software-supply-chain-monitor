# XCSSET v40 — Xcode Project Poisoning Turns Developer Workstations Into Self-Propagating Supply Chain Vectors

**Date:** April 2026
**Ecosystem:** macOS / Xcode / GitHub
**Severity:** High
**Type:** Supply Chain / Xcode Project Poisoning / Self-Propagating Worm / Infostealer
**Sources:**
- [Unit 42 (Palo Alto Networks) — The Xcode Assassin Returns: A Deep Dive Into the Latest XCSSET Version](https://unit42.paloaltonetworks.com/xcsset-v40-malware-analysis/)

---

## Summary

Since early April 2026, XCSSET v40 has spread through supply chain attacks by hiding malicious run-script build phases inside the Xcode projects of dozens of legitimate macOS applications hosted on GitHub, collectively reaching thousands of active users. When a developer clones one of these repositories and builds the project locally, the injected script silently executes, establishes C2 communication, and begins a four-stage infection chain that ultimately runs entirely in memory. A secondary wave in early May 2026 introduced an expanded module set including a Chrome DevTools Protocol (CDP) browser backdoor and a Telegram Desktop trojanizer.

XCSSET v40 represents a significant architectural evolution from prior versions. The malware rewrites its own binary and module payloads with fresh encryption keys every few hours using server-side polymorphic compilation, leaving no stable file signatures for scanners to anchor on. It achieves fileless persistence through the macOS `defaults` configuration system rather than Launch Daemons or .zshrc entries alone, stores a Base64-encoded staging payload under randomized preference keys, and actively impairs Apple's own defenses: it disables the SoftwareUpdate configuration channel, terminates CloudTelemetryService, holds an exclusive file lock on the XProtect YARA signature database, and resets TCC permission decisions to force re-approval of automation dialogs. Anti-VM checks halt module delivery to sandbox environments, ensuring the core logic reaches only physical developer machines.

Unit 42 tracked the attack to a Chinese-speaking threat actor primarily targeting developers across South Asia. The actor registered roughly 40 C2 domains in at least four short bursts in early 2026, aged them for months before deploying the attack, and bound all four operator IP addresses to a single shared SSL thumbprint — an OPSEC failure that enabled infrastructure attribution. XCSSET's original author tracked by Trend Micro in 2020 and Microsoft in 2025 continued to iterate on the codebase, with v40 representing the most sophisticated version publicly documented.

---

## Compromised Artifacts

| Artifact | Type | Notes |
|----------|------|-------|
| Dozens of legitimate Xcode projects on GitHub | Xcode project files (.xcodeproj) | Malicious run-script injected into build phases |
| Git pre-commit hooks across infected repositories | Git hooks | Propagation via `git_finder` module |
| ZIP archives containing Xcode projects | Archives | Propagation via `zip_infect_finder` module (new in v40) |
| `~/.tr` Telegram replacement app | macOS app bundle | Telegram Desktop trojanized binary (May 2026 wave) |
| Google Chrome browser binary wrapper | Shell script | CDP-based browser hijacking module |

---

## How It Worked

### Entry Point — Poisoned Xcode Build Phase

XCSSET v40 injects a malicious run-script phase into the `.xcodeproj` build configuration of legitimate projects. No file is dropped until a developer opens the infected project and builds it locally. At build time, the run script:

1. Dynamically scrambles a Base64/hex-encoded payload at compile time, rotating encoding layers per build.
2. Executes a curl request to the C2 `/a` endpoint with execution context (`p=xcode_phase`), retrieving a staging bash script obfuscated with a custom per-build substitution cipher.
3. Performs hardware fingerprinting, matching the host serial number against targeted profiles before proceeding.

### Stages 2–4 — In-Memory Loader Chain

If the C2 approves the host, it delivers the main loader as a compiled AppleScript wrapper (`/tmp/p.app`) alongside a companion bash binary (`/tmp/r`). The loader is invoked via `osascript` — executing entirely in memory — downloads the core orchestrator module (`boot`), and then self-destructs: both `/tmp/r` and `/tmp/p.app` are deleted from disk immediately after execution, leaving no installation artifacts.

The `boot` orchestrator runs a continuous dispatch loop, retrieving 17 specialized modules from `https://<C2>/s/<encoded_module_name>`, decrypting each with a per-module AES-256-CBC key and randomized IV (prepended to each payload), and piping them to `osascript` for in-memory execution. Because the IV shifts with every transmission, identical modules served seconds apart produce different encrypted blobs, defeating content-based network detection.

### Propagation — Worm Capabilities

XCSSET v40 self-propagates across all Xcode projects it can find on the infected system through three mechanisms:

- **`replicator_finder`**: Crawls the filesystem for `.xcodeproj` files and injects the malicious build phase into each one.
- **`git_finder`**: Installs malicious `pre-commit` hooks into local Git repositories so the payload re-propagates every time the developer commits code — and into remote forks when the infected project is pushed to GitHub.
- **`zip_infect_finder`** (new in v40): Recursively traverses user directories searching for ZIP archives containing Xcode projects, infects them, and rezips — targeting compressed project backups and distributed archives.

### Payload — 17 Operational Modules

XCSSET v40 ships the following modules, all executed in-memory via the `boot` orchestrator:

| Module | Functionality |
|--------|---------------|
| `boot` | Main orchestrator, module-dispatch loop |
| `stats` | Reconnaissance + anti-VM checks; exfiltrates browser extensions |
| `clipboard_v2` | Keyboard hijacker |
| `payloader` | Secondary module dispatcher; dynamic config download |
| `replicator_finder` | Xcode project file infector |
| `git_finder` | Git pre-commit hook infector |
| `zip_infect_finder` | ZIP archive Xcode project infector (new in v40) |
| `data_folders_finder` | C2-driven folder finder and data exfiltrator |
| `firefox_data` | Firefox credential stealer |
| `notes_app` | Apple Notes content exfiltrator |
| `settings_app` | LaunchDaemon persistence + XProtect defense impairment |
| `finder_app` | TCC permission misuse and reset; trojanized Finder/Xcode/Terminal |
| `persist` | `.zshrc` and Dock-app persistence |
| `browser_remote` | Browser hijack dispatcher (Chrome, Firefox, Safari, Brave, Edge, Opera, Yandex, 360) |
| `safari_remote` | Safari-specific browser hijacker |
| `chrome_remote` | **NEW in v40** — Chrome CDP backdoor with remote JS execution and fileless reverse shell |
| `tdesktop` | **NEW in May 2026 wave** — Telegram Desktop trojanizer |

### Chrome CDP Backdoor (`chrome_remote`)

The `chrome_remote` module drops a native binary (`chrome_remote`) and wraps the legitimate Google Chrome binary with a shell script that, on every browser launch: restarts the `boot` orchestrator, launches Chrome with CDP exposed on a pre-defined local port, and invokes `chrome_remote`. The binary establishes a persistent WebSocket to the C2 server and injects arbitrary JavaScript into every new tab before the page loads via CDP, enabling:

- **Traffic interception**: hooks on `window.fetch` and `XMLHttpRequest` exfiltrate credentials and API tokens.
- **Crypto wallet manipulation**: intercepts MetaMask's Ethereum provider to alter wallet addresses or manipulate dApp transactions.
- **Credential theft**: overrides password-manager autofill fields to capture credentials on submission.
- **Host-level RCE**: monitors active tabs for operator console.log commands with a recognized delimiter, strips the delimiter, and passes the payload to the host shell via `exec.Command` — establishing a fileless reverse shell through the browser's CDP connection.

### Telegram Trojanizer (`tdesktop`)

Introduced in the May 2026 wave, this module: downloads a pre-built malicious Telegram.app ZIP from the C2, wipes the legitimate Telegram Desktop installation, drops the replacement, ad-hoc code-signs it, and kills the original Telegram process so the victim relaunches the trojanized copy. An AES-encrypted per-host configuration (`~/.tr`) tracks Telegram-related state. Unlike prior XCSSET generations that merely exfiltrated Telegram chat history, v40 replaces the application binary itself.

### Defense Impairment

XCSSET v40 actively sabotages Apple's security mechanisms:

- **SoftwareUpdate**: disables auto-retrieval of XProtect, MRT, and TCC signature database updates and Apple Rapid Security Response.
- **CloudTelemetryService**: runs a constant termination loop preventing security telemetry from reaching Apple, ensuring the malware's signatures don't reach subsequent XProtect releases.
- **XProtect signature database**: spawns a Perl process holding an exclusive file lock on the YARA rule database (`XPdb`), preventing updates from being written even if they are delivered.
- **TCC database reset**: when automation permission is denied, invokes `tccutil reset AppleEvents` to clear the decision and re-present the consent dialog disguised as System Settings or Xcode.

### Fileless Persistence via macOS Defaults

Rather than dropping executable scripts between infection cycles, v40 writes a Base64-encoded staging payload into a per-host `defaults` preferences domain under keys with randomized names (e.g., `mpirv_eahpi_apm`, `ychax_muwch_ucy`). Any trojanized or hijacked application triggers a one-liner that decodes and re-executes the payload, re-arming the infection. A `SRC` tag tracks which vector triggered the rearm (hijacked browser, infected Xcode project, or trojanized app).

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Early 2026 | Threat actor registers ~40 C2 domains in four short bursts across four IP addresses; domains aged before deployment |
| Mid-April 2026 | Unit 42 begins tracking XCSSET v40; first wave of infected Xcode projects detected on GitHub |
| Early May 2026 | Second wave introduces `tdesktop` Telegram trojanizer and `zip_infect_finder` module |
| 2026-07-31 | Unit 42 publishes full technical analysis with IOCs, module breakdown, and C2 infrastructure attribution |

---

## Detection

```bash
# ---- Xcode project infection ----
# Check all local Xcode projects for malicious run-script build phases
find ~ -name "project.pbxproj" 2>/dev/null | xargs grep -l "base64\|xxd\|curl.*\/a\b" 2>/dev/null

# Grep for the XCSSET C2 URI pattern in Xcode project build scripts
find ~ -name "project.pbxproj" 2>/dev/null | xargs grep -E "curl.*/(a|s/|d/)" 2>/dev/null

# Check for malicious git pre-commit hooks
find ~ -path "*/.git/hooks/pre-commit" 2>/dev/null | xargs grep -l "base64\|curl" 2>/dev/null

# ---- Fileless persistence (macOS defaults) ----
# List all user preference domains (XCSSET creates random-looking domain names)
defaults domains 2>/dev/null | tr ',' '\n' | grep -vE "^(com\.|NSGlobal|Apple|loginwindow)" | head -30

# Check for XCSSET-style keys (random lowercase underscore strings) in any domain
defaults read | grep -E '"[a-z]{4,10}_[a-z]{4,10}_[a-z]{3,8}"' 2>/dev/null

# ---- Trojanized Telegram ----
# Verify Telegram.app code signature (trojanized copy is ad-hoc signed, not Apple notarized)
codesign -dv --verbose=4 /Applications/Telegram.app 2>&1 | grep -E "Authority|TeamIdentifier"
# Legitimate Telegram: TeamIdentifier=6N38VWS5BX (Telegram FZ-LLC)
# Trojanized: Authority=adhoc or no Authority line

# Check ~/.tr configuration file (XCSSET Telegram tracking)
ls -la ~/.tr ~/.tr_map 2>/dev/null

# ---- Chrome wrapper ----
# Check if Google Chrome binary has been replaced with a wrapper script
file /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome
# Should be: Mach-O 64-bit executable (not a shell script)
head -1 /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome 2>/dev/null | grep -c "^#!"
# Output > 0 means Chrome binary has been replaced with a shell script wrapper

# ---- SoftwareUpdate sabotage ----
defaults read /Library/Preferences/com.apple.SoftwareUpdate AutomaticCheckEnabled 2>/dev/null
defaults read /Library/Preferences/com.apple.SoftwareUpdate ConfigDataInstall 2>/dev/null
# Both should be 1 (true); value 0 indicates XCSSET may have disabled them

# ---- Network IOCs ----
# Check DNS lookups / network connections to known XCSSET v40 C2 domains
for domain in accapple.ru adschecks.ru adsmobi.ru amzndev.in amzndev.ru applecdn.ru cdnamz.ru cdnapple.in chromads.ru googlenets.ru netcdndev.in whitead.in whiteads.ru; do
  dns_result=$(dig +short $domain 2>/dev/null)
  [ -n "$dns_result" ] && echo "ALERT: $domain resolved to $dns_result"
done

# Check for active connections to known XCSSET C2 IPs
ss -tnp 2>/dev/null | grep -E '91\.108\.106\.229|95\.142\.35\.(34|206)|95\.142\.37\.159|151\.243\.109\.188|178\.208\.92\.(129|168)'
netstat -an 2>/dev/null | grep -E '91\.108\.106\.229|95\.142\.35\.(34|206)|95\.142\.37\.159|151\.243\.109\.188|178\.208\.92\.(129|168)'

# Check osascript network activity (unusual for normal use)
lsof -i -n -P 2>/dev/null | grep osascript

# ---- SSL thumbprint ----
# If checking TLS traffic from the host, flag certs with this shared thumbprint across all XCSSET IPs
# Thumbprint: 6e480d648fa1b70612f5d198a66875e28847547d

# ---- C2 Chrome CDP URLs ----
# Block or alert on outbound connections to these specific endpoints:
# https://amzndev[.]in/d/zw_sfp64
# https://amzndev[.]ru/d/zw_sfp64
# https://googlenets[.]ru/d/zw_sfp64
# https://netcdndev[.]in/d/zw_sfp64
# https://whitead[.]in/d/zw_sfp64
# https://whiteads[.]ru/d/zw_sfp64
```

---

## Remediation

1. **Audit all local Xcode projects** for malicious build phases (`project.pbxproj` files containing unexpected `base64`, `xxd`, or `curl` commands in run-script phases). Remove any such phases and rebuild from a clean state.
2. **Check Git pre-commit hooks** across all local repositories (`find ~ -path "*/.git/hooks/pre-commit"`). Replace any hooks containing shell payloads with expected content.
3. **Reinstall Telegram Desktop** from the official source (https://desktop.telegram.org) and verify the code signature: `codesign -dv --verbose=4 /Applications/Telegram.app`. The legitimate TeamIdentifier is `6N38VWS5BX`.
4. **Reinstall Google Chrome** if the main binary has been replaced with a shell script wrapper. Verify with `file /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome`.
5. **Purge XCSSET defaults domains**: list all user defaults domains (`defaults domains`) and delete any that use random-looking names (lowercase words joined by underscores). Example: `defaults delete <domain_name>`.
6. **Re-enable SoftwareUpdate**: `sudo defaults write /Library/Preferences/com.apple.SoftwareUpdate AutomaticCheckEnabled -bool true; sudo softwareupdate --background`. Then immediately run `sudo softwareupdate -ia` to pull any pending XProtect and security data file updates.
7. **Rotate all credentials** stored in Chrome, Firefox, and Safari (login databases, cookies). Assume any credential entered in a browser on the infected machine since April 2026 has been captured.
8. **Move crypto assets**: any MetaMask or browser-extension wallet on the machine should be treated as fully compromised. Move assets to a fresh wallet generated on a clean device before any further transactions.
9. **Check GitHub repositories** for malicious Xcode projects and warn downstream contributors/users. File a GitHub security advisory if your project was compromised.
10. **Block C2 domains and IPs** at the network perimeter using the IOC list below.

---

## Lessons Learned

- **Developer IDEs are a persistent high-value attack surface**: Xcode build phases execute arbitrary shell commands and are not subject to the same scrutiny as package lifecycle scripts. All CI/CD systems building Xcode projects should enforce build-phase audits.
- **GitHub open-source project poisoning propagates across the whole contributor graph**: forking an infected repo, cloning it for review, or downloading a ZIP distributes the malware — no `npm install` analogue required.
- **Fileless re-arming via macOS defaults** is an under-monitored persistence mechanism. EDR rules and YARA signatures should specifically cover abnormal user-created preference domains and their modification.
- **Server-side polymorphic compilation** (rotating binary hashes every few hours) renders file-hash-based detections ineffective within the attack window; behavioral detection is the only reliable signal.
- **Disabling XProtect update delivery** can allow a threat to persist even after Apple issues a signature update, stressing the importance of monitoring for SoftwareUpdate configuration changes.
- **Defense-impairment TTPs should be treated as immediate IOCs**: any process invoking `tccutil reset AppleEvents`, holding exclusive locks on XProtect databases, or killing CloudTelemetryService is a high-fidelity alert regardless of other indicators.
- **Supply chain propagation via trojanized tools** (Telegram Desktop, Chrome wrapper) affects all users of the compromised developer's communications, extending the blast radius beyond the development environment.

---

## Indicators of Compromise

### C2 Domains (partial list)
```
accapple[.]ru     adschecks[.]ru    adsmobi[.]ru       adsmorein[.]in    adsmoreme[.]in
amdcdn[.]ru       amzndev[.]in      amzndev[.]ru       amznprod[.]in     applecdn[.]ru
appledisk[.]ru    appledns[.]ru     applehosts[.]ru    appletime[.]in    bulksec[.]ru
cdnamz[.]in       cdnamz[.]ru       cdnapple[.]in      cdnatapple[.]ru   cdnroute[.]ru
checkcdn[.]ru     chromeads[.]ru    cnmag[.]ru         devnetaps[.]ru    dnsapple[.]ru
dnsrelays[.]ru    explorecdn[.]ru   figmacat[.]ru      figmanets[.]in    funchats[.]ru
googlenets[.]ru   icloudsnet[.]ru   imails[.]ru        legalads[.]in     littleads[.]in
littledns[.]ru    netapsdev[.]ru    netcdnads[.]in     netcdnamz[.]ru    netcdndev[.]in
netcorps[.]ru     netsprot[.]in     netsproto[.]in     networkads[.]in   rigacdn[.]in
rigmajoys[.]in    rigmanet[.]ru     rigmanets[.]in     sahusuzuki[.]in   stuffdns[.]in
timewebnet[.]in   vigmanet[.]ru     whitead[.]in       whiteads[.]ru     wincdn[.]ru
windsecure[.]ru
```

### C2 IPs
```
91.108.106.229
95.142.35.34
95.142.35.206
95.142.37.159
151.243.109.188
178.208.92.129
178.208.92.168
```

### Shared SSL Thumbprint (all operator IPs)
```
6e480d648fa1b70612f5d198a66875e28847547d
```

### Chrome CDP Helper Binary URLs
```
hxxps://amzndev[.]in/d/zw_sfp64
hxxps://amzndev[.]ru/d/zw_sfp64
hxxps://googlenets[.]ru/d/zw_sfp64
hxxps://netcdndev[.]in/d/zw_sfp64
hxxps://whitead[.]in/d/zw_sfp64
hxxps://whiteads[.]ru/d/zw_sfp64
```

---

## Related Incidents

- [./2026-07-ghostapproval-ai-coding-assistants.md](./2026-07-ghostapproval-ai-coding-assistants.md) — AI coding assistant trust boundary exploited to write attacker keys via symlink attack; similar developer-IDE attack surface
- [./2026-06-amazon-q-mcp-auto-execution.md](./2026-06-amazon-q-mcp-auto-execution.md) — VS Code extension auto-executes MCP config on folder open; same class of developer tooling compromise
- [./2026-06-jetbrains-ai-key-theft.md](./2026-06-jetbrains-ai-key-theft.md) — Malicious IDE plugins steal API keys from developer environments
- [./2026-04-glassworm-zig-dropper.md](./2026-04-glassworm-zig-dropper.md) — Trojanized IDE extension infects all compatible IDEs on the machine via Zig-compiled native dropper
- [./2026-03-iolitelabs-vscode-backdoor.md](./2026-03-iolitelabs-vscode-backdoor.md) — Dormant publisher account hijacked; multi-stage backdoor deployed via IDE extension
