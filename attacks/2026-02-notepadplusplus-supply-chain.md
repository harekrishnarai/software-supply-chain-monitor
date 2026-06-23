# Notepad++ Supply Chain Attack — Lotus Blossom APT Hijacks Update Mechanism to Deliver Chrysalis Backdoor

**Date:** Jul–Oct 2025 (attack); Feb 2026 (disclosure)
**Ecosystem:** Windows / Notepad++ update infrastructure (WinGUP)
**Severity:** High
**Type:** Update Mechanism Hijack / Nation-State Espionage / Targeted Delivery
**Sources:**
- [Unit 42 — Nation-State Actors Exploit Notepad++ Supply Chain](https://unit42.paloaltonetworks.com/notepad-infrastructure-compromise/)
- [Help Net Security — Notepad++ supply chain attack: Researchers reveal details, IoCs, targets](https://www.helpnetsecurity.com/2026/02/03/notepad-supply-chain-attack-iocs-targets/)
- [Kaspersky Securelist — The Notepad++ supply chain attack – unnoticed execution chains and new IoCs](https://securelist.com/notepad-supply-chain-attack/118708/)
- [Rapid7 — TR: Chrysalis Backdoor — Dive into Lotus Blossom's Toolkit](https://www.rapid7.com/blog/post/tr-chrysalis-backdoor-dive-into-lotus-blossoms-toolkit/)

---

## Summary

Between July and October 2025, a Chinese state-sponsored threat group known as Lotus Blossom (aka Billbug), tracked by Unit 42 as CL-STA-0062, compromised the official hosting infrastructure for the Notepad++ text editor and selectively poisoned its WinGUP auto-update mechanism. When targeted users attempted to update the software, they downloaded a malicious NSIS installer instead of a legitimate update, triggering infection chains that delivered either the newly-identified Chrysalis backdoor or a Cobalt Strike Beacon.

Unlike mass supply chain attacks that broadcast malicious packages indiscriminately, this campaign was surgical: only specific targeted users received the malicious update payload. The attackers rotated C2 servers, dropper mechanisms, and final payloads continuously over four months to frustrate detection. Kaspersky identified three distinct execution chains operating simultaneously, each using different sideloading techniques and final payloads.

Disclosure began in early February 2026. Unit 42 subsequently identified additional unreported infrastructure revealing the target pool was geographically broader than initially understood — extending from Southeast Asia and South America into the US and Europe, including organizations in energy, financial services, government, manufacturing, cloud hosting, and software development.

The strategic targeting of Notepad++ is deliberate: the text editor is deeply embedded in enterprise workflows for system administrators, network engineers, and DevOps personnel. Compromising this single tool gives attackers implicit access to the sessions of the most privileged users in an organization.

---

## Compromised Artifacts

| Component | Description |
|-----------|-------------|
| Notepad++ WinGUP updater | Official update mechanism compromised at hosting infrastructure level |
| `update.exe` | Malicious NSIS installer served to targeted users during update check |
| `log.dll` | Malicious DLL sideloaded by `BluetoothService.exe` (renamed Bitdefender Wizard binary) |
| `BluetoothService.exe` | Legitimate Bitdefender Submission Wizard, renamed and abused for DLL sideloading |

---

## How It Worked

### Entry Point

Attackers exploited insufficient verification controls in older versions of the Notepad++ WinGUP updater. WinGUP did not verify the certificate or signature of downloaded update installers, allowing the attackers to intercept and redirect traffic destined for the Notepad++ update server. This infrastructure-level hijack enabled selective, per-victim targeting — only chosen targets received the malicious update; other users received a legitimate installer.

### Execution Chains

Three distinct infection chains were identified operating simultaneously, each with different dropper mechanisms and final payloads:

**Chain 1: DLL Sideloading → Chrysalis Backdoor**

1. Malicious NSIS `update.exe` is downloaded by victim during update check
2. Installer sideloads `log.dll` via a renamed legitimate Bitdefender Submission Wizard binary (`BluetoothService.exe`)
3. `log.dll` decrypts and loads encrypted shellcode (`BluetoothService` shellcode) into the `BluetoothService.exe` process
4. Shellcode resolves APIs using custom hashing, bypassing straightforward detection
5. Shellcode is the Chrysalis backdoor — a feature-rich, persistent C2 implant

**Chain 2: ProShow Exploit → Cobalt Strike Beacon**

1. NSIS installer sends a heartbeat containing system information to the attacker
2. Installer drops a vulnerable version of legitimate ProShow software
3. ProShow is used to launch an exploit payload (also dropped by the installer)
4. Exploit payload decrypts a Metasploit downloader, which retrieves a Cobalt Strike Beacon
5. Beacon establishes C2 communication with attacker infrastructure

**Chain 3: Lua Script → Metasploit → Cobalt Strike Beacon**

1. NSIS installer executes a Lua script
2. Lua script launches a Metasploit downloader
3. Downloader retrieves and executes a Cobalt Strike Beacon payload

### Chrysalis Backdoor Capabilities

Chrysalis is described by Rapid7 as a sophisticated, permanent tool (not a throwaway utility) with a wide array of capabilities:

- Custom API hashing in both loader and main module, each with separate resolution logic
- Layered obfuscation with structured C2 communication protocols
- Feature-rich backdoor functionality suggesting active long-term development
- Uses legitimate binary sideloading to bypass filename-based detection
- Communicates with rotating C2 infrastructure (servers were rotated throughout the 4-month campaign)

### Campaign Operational Security

The attackers demonstrated strong OpSec throughout the campaign: C2 server addresses were rotated continuously from July to October 2025; dropper mechanisms changed across victims; final payloads differed per execution chain. This rotation made it difficult for any single telemetry source to observe the full campaign scope.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Jul 2025 | First observed compromise activity; WinGUP update mechanism hijacked |
| Jul–Oct 2025 | Continuous rotation of C2 servers, droppers, and final payloads |
| Oct 2025 | Last observed attack activity in primary telemetry |
| Feb 2, 2026 | Helpnetsecurity reports initial Notepad++ update hijack disclosure |
| Feb 3, 2026 | Rapid7 and Kaspersky publish detailed IOCs and attribution to Lotus Blossom |
| Feb 11, 2026 | Unit 42 publishes expanded infrastructure analysis revealing broader target geography |
| Feb 12, 2026 | Notepad++ maintainer hardens WinGUP: adds certificate + signature verification; migrates to new hosting provider |
| Feb 19, 2026 | Unit 42 article last updated with additional findings |

---

## Detection

```bash
# Check for Chrysalis-related DLL sideloading artifacts
# Malicious DLL deployed via legitimate Bitdefender binary
Get-ChildItem -Path C:\ -Filter "BluetoothService.exe" -Recurse 2>$null
Get-ChildItem -Path C:\ -Filter "log.dll" -Recurse 2>$null

# Check for unexpected processes spawned by BluetoothService.exe
# (PowerShell on Windows)
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4688} | 
  Where-Object {$_.Properties[13].Value -match 'BluetoothService'} | 
  Select-Object TimeCreated, Message

# Check for Notepad++ update artifacts
# Malicious NSIS installer typically named update.exe
Get-ChildItem -Path $env:TEMP -Filter "update.exe" 2>$null
Get-ChildItem -Path $env:LOCALAPPDATA -Filter "update.exe" -Recurse 2>$null

# Verify current Notepad++ WinGUP updater version
# Versions pre-fix do NOT verify installer signatures
# Check Notepad++ installation:
notepad++ --version 2>$null || ls "C:\Program Files\Notepad++" 2>$null

# Look for ProShow-related artifacts (Chain 2)
Get-ChildItem -Path C:\Users -Filter "ProShow*" -Recurse 2>$null

# Hash verification for known Chrysalis shellcode loader
# (see Rapid7 and Kaspersky IOC reports for specific hashes)
certutil -hashfile log.dll SHA256

# Network: look for outbound connections matching Chrysalis C2 pattern
# C2 addresses rotated frequently — check Kaspersky and Unit 42 IOC lists
# for current known infrastructure
```

For the complete set of C2 IP addresses, file hashes, and domain IOCs, refer to:
- Kaspersky Securelist report: https://securelist.com/notepad-supply-chain-attack/118708/
- Rapid7 technical report (includes Chrysalis YARA rules)
- Unit 42 expanded IOC list: https://unit42.paloaltonetworks.com/notepad-infrastructure-compromise/

---

## Remediation

1. Update Notepad++ to the latest version (post-February 2026 releases include hardened WinGUP with certificate + signature verification)
2. If running an older Notepad++ version, disable auto-update until the software is upgraded to a patched version
3. Use IOCs from Rapid7, Kaspersky, and Unit 42 to hunt for Chrysalis artifacts and C2 connections in endpoint and network telemetry
4. Review endpoint logs for `BluetoothService.exe` spawning unexpected child processes or loading unsigned DLLs from unexpected paths
5. Check for Cobalt Strike Beacon activity from endpoints where Notepad++ is installed (particularly sysadmin and DevOps workstations)
6. Prioritize investigation of privileged user workstations — the campaign specifically targeted sysadmins and DevOps engineers
7. Organizations in targeted sectors (energy, finance, government, manufacturing, cloud hosting, software dev) in SE Asia, South America, US, or Europe should treat this as a high-priority hunt

---

## Lessons Learned

- **Developer tooling for privileged users is high-value APT territory**: Notepad++ is mundane software — but its usage concentrated among sysadmins and DevOps engineers made it a strategically optimal target for nation-state actors seeking persistent access to privileged sessions.
- **Update mechanism integrity matters at the infrastructure level**: Application-level signature verification is insufficient if the hosting infrastructure can be compromised. WinGUP's pre-fix lack of installer signature verification was the direct enabler.
- **Surgical targeting limits mass detection**: Because only specific victims received the malicious update, no single monitoring source saw the full picture. This underscores the need for cross-vendor telemetry sharing for nation-state campaigns.
- **Payload rotation defeats IOC-based defense**: Rotating C2s, droppers, and final payloads across a 4-month campaign means static IOC feeds are insufficient — behavioral detection is required.

---

## Related Incidents

- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — Nation-state-adjacent campaign targeting CI/CD infrastructure via GitHub Actions tag poisoning
- [./2026-02-hackerbot-claw.md](./2026-02-hackerbot-claw.md) — AI-powered supply chain attack targeting developer infrastructure
