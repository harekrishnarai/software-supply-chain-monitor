# NeoShadow — MSBuild & Blockchain npm Backdoor Campaign

**Date:** December 2025 – January 2026
**Ecosystem:** npm
**Severity:** High
**Type:** Typosquatting / Backdoor / Infostealer
**Sources:**
- [Aikido Security — NeoShadow npm Supply-Chain Attack: JavaScript, MSBuild & Blockchain](https://www.aikido.dev/blog/neoshadow-npm-supply-chain-attack-javascript-msbuild-blockchain)

---

## Summary

On December 30, 2025, Aikido Security's automated analysis engine detected a sudden burst of new npm packages published by a single author (`cjh97123`). The packages used typosquatting names targeting popular libraries and contained unusually sophisticated multi-stage malware that Aikido named **NeoShadow** — based on internal identifiers found in the decrypted payload (`NeoShadowV2DeriveKey2026`, `Global\NSV2_8e4b1d`). The C2 domain `metrics-flow[.]com` was registered on the same day the packages appeared, confirming a coordinated campaign.

A second wave of packages deployed on January 2, 2026, introduced a native Windows executable (`analytics.node`) with novel obfuscation techniques that defeated all VirusTotal detections at the time. The final payload is a fully featured backdoor designed for long-term persistence, using **ChaCha20** stream cipher with **Curve25519 ECDH** key exchange for encrypted C2 communications — a level of cryptographic sophistication rarely seen in npm-based supply chain attacks.

What made NeoShadow technically notable was its use of **MSBuild and C# code** for payload execution (using LOLbin techniques to avoid spawning `cmd.exe` or `powershell.exe`) combined with a blockchain-linked delivery mechanism to make the C2 infrastructure more resilient to takedown.

---

## Compromised Artifacts

| Package Author | Versions | Published |
|---|---|---|
| `cjh97123` (npm) | Wave 1 packages | 2025-12-30 |
| `cjh97123` (npm) | Wave 2 packages (includes `analytics.node`) | 2026-01-02 |

All packages are typosquats of legitimate, popular npm libraries. Specific names were documented in Aikido's full disclosure report. The C2 domain `metrics-flow[.]com` was registered 2025-12-30 — simultaneous with the first package publication.

---

## How It Worked

### Entry Point — Typosquatting

The attacker published a burst of new packages under the author name `cjh97123`, using names chosen to closely resemble popular npm packages (typosquatting). The packages contained heavily obfuscated JavaScript as their initial stage — obfuscation quality was high enough that standard deobfuscation tools failed, prompting Aikido to develop new deobfuscation toolchains specifically to analyze this campaign.

### Stage 1 — JavaScript Dropper

The initial JavaScript payload attempts to download the stage-2 payload from the C2 domain:
```
https://metrics-flow[.]com/assets/js/analytics.min.js
```
The C2 employs **anti-analysis evasion at the server level**: rather than serving a fixed response, it returns completely random fake content on most requests, making the domain appear benign to automated scanners. Actual payload delivery is gated by fingerprinting logic that checks request timing, headers, and other signals before serving the real payload.

### Stage 2 — MSBuild/C# Execution (Wave 1)

The downloaded payload is encrypted with an **RC4 key** and, after decryption, executes using **MSBuild** — a Windows built-in build tool — to run inline C# code without spawning `powershell.exe` or `cmd.exe`. This LOLbin technique evades EDR rules that monitor standard scripting interpreters.

### Stage 2 — analytics.node (Wave 2, January 2, 2026)

The second campaign wave included a pre-compiled Windows executable distributed as `analytics.node` (a `.node` native addon disguised by name). At time of discovery, **zero AV engines on VirusTotal flagged the binary as malicious**. The JavaScript in this wave also used a different, newer obfuscation technique compared to wave 1.

### Stage 3 — NeoShadow Backdoor

The final stage is a lightweight, long-term access implant. Once running, it:
1. Enters a **beacon loop** — periodically checks in with the C2, reports system information, and polls for tasking commands
2. Communicates exclusively over **ChaCha20-encrypted channels** with **Curve25519 ECDH** key establishment (preventing traffic decryption without the attacker's private key)
3. Acts as a **modular execution primitive** — all post-exploitation functionality (credential dumping, lateral movement, etc.) is delivered as disposable modules pushed from the C2, keeping the implant itself small and signature-resistant

Internal artifacts confirming the campaign name and version:
- String: `NeoShadowV2DeriveKey2026`
- Mutex: `Global\NSV2_8e4b1d`
- PDB path: `C:\Users\admin\Desktop\NeoShadow\core\loader\native\build\Release\analytics.pdb`

### C2 Infrastructure

| Domain | Role |
|---|---|
| `metrics-flow[.]com` | Primary stage-2 download and C2 beacon |

The domain serves random fake content to most visitors. The server-side gating logic ensures real payloads are only delivered to likely victims, making automated crawling and sandbox detection ineffective.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2025-12-30 | C2 domain `metrics-flow[.]com` registered |
| 2025-12-30 | Wave 1 npm packages published by `cjh97123`; Aikido analysis engine flags them as suspicious |
| 2025-12-30 | Aikido begins investigation; standard deobfuscation tooling fails; new toolchains developed |
| 2026-01-02 | Wave 2 packages deployed with new obfuscation and `analytics.node` native Windows executable |
| 2026-01-02 | `analytics.node` binary: 0/VirusTotal detections at time of discovery |
| 2026-01 | Aikido publishes full technical analysis naming the campaign "NeoShadow" |

---

## Detection

```bash
# Check for packages by the malicious author in your node_modules
find node_modules -name "package.json" -exec grep -l '"_npmUser"' {} \; | \
  xargs grep -l '"cjh97123"' 2>/dev/null

# Check your package-lock.json for packages resolved from cjh97123
grep -B5 '"cjh97123"' package-lock.json 2>/dev/null

# Look for analytics.node in unexpected paths
find . node_modules -name "analytics.node" 2>/dev/null

# Check for MSBuild invocations from node processes (Windows)
# In Windows Event Log: look for MSBuild.exe with parent process node.exe or npm.exe

# Check network logs for C2 domain
grep -E "metrics-flow\.com" /var/log/syslog /var/log/dns*.log 2>/dev/null
# Windows: check DNS resolver cache
ipconfig /displaydns | findstr "metrics-flow"

# Look for the NeoShadow mutex (Windows)
# Powershell: Get-WmiObject Win32_Process | Where-Object { $_.CommandLine -match "NSV2" }

# Check for ChaCha20 beacon traffic anomalies (encrypted, high-frequency beaconing):
# Look for regular short (< 200 byte) encrypted outbound connections to metrics-flow[.]com
```

---

## Remediation

1. **Audit all npm installs** from December 30, 2025 – January 10, 2026 in all CI/CD environments and developer machines. Check for packages authored by `cjh97123`.

2. **Remove any affected packages** from `node_modules` and clear the npm cache:
   ```bash
   npm cache clean --force
   rm -rf node_modules package-lock.json
   npm install
   ```

3. **Block the C2 domain** at DNS and firewall levels: `metrics-flow[.]com` and all subdomains.

4. **Hunt for persistence** — if any `analytics.node` binary was executed:
   - Check Windows startup locations (`HKCU\Software\Microsoft\Windows\CurrentVersion\Run`, `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup`)
   - Check macOS LaunchAgents: `~/Library/LaunchAgents/`
   - Check Linux systemd user units and cron: `crontab -l`, `~/.config/systemd/user/`

5. **Assume full compromise** if any stage of the backdoor executed — rotate all credentials, tokens, API keys, and SSH keys accessible from the affected machine.

6. **Enable npm provenance** and pin packages to exact content-hashes in `package-lock.json`. Consider tools like Aikido SafeChain or Socket to block newly published packages from unknown authors.

---

## Lessons Learned

- **ChaCha20 + Curve25519 in a supply chain implant** represents a significant uplift in operational security — defenders cannot decrypt C2 traffic without the attacker's private key, making network forensics harder than with simpler encryption.
- **Server-side payload gating** defeats automated sandbox analysis: C2 infrastructure that serves benign content to crawlers while delivering real payloads only to victims will not be caught by URL reputation feeds or passive monitoring.
- **VirusTotal is not a detection guarantee.** The Wave 2 `analytics.node` binary had zero detections at time of publication, demonstrating that compiled native executables with custom encryption can evade all signature-based AV.
- **MSBuild as a LOLbin** is well-documented but still effective — using system-native build tools to execute inline code avoids spawning `powershell.exe` or `cmd.exe`, bypassing many EDR behavioral rules.
- **Burst publication patterns** are a detectable signal: multiple packages published simultaneously from a new or little-known author within a short window are a strong anomaly indicator that warrants automated review.

---

## Related Incidents

- [./2025-09-great-npm-heist.md](./2025-09-great-npm-heist.md) — Large-scale npm compromise of foundational packages via account takeover
- [./2026-03-canisterworm-npm.md](./2026-03-canisterworm-npm.md) — Another blockchain-based C2 technique (ICP canister) used in an npm worm
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — npm infostealer targeting crypto and browser credentials
