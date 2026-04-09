# GlassWorm Native — Zig Dropper IDE Mass-Infection

**Date:** April 2026
**Ecosystem:** OpenVSX / VS Code / Cursor / Windsurf / VSCodium / Positron
**Severity:** Critical
**Type:** Trojanized Extension / Native Compiled Dropper / IDE Mass-Infection
**Sources:**
- [Aikido Security — GlassWorm goes native: New Zig dropper infects every IDE on your machine](https://www.aikido.dev/blog/glassworm-zig-dropper-infects-every-ide-on-your-machine)

---

## Summary

On April 8, 2026, Aikido Security reported a new evolution of the GlassWorm campaign: a trojanized OpenVSX extension called `code-wakatime-activity-tracker` that ships a Zig-compiled native binary alongside its JavaScript code. The extension impersonates WakaTime, the popular developer time-tracking tool, replicating its command registrations, API key prompts, and status bar icons almost perfectly. The divergence is a single call at activation time that loads the malicious native binary before any WakaTime logic runs.

Rather than using the binary as the payload directly, GlassWorm employs it as a stealthy indirection layer. The binary's sole purpose is to enumerate every IDE installed on the machine that supports the VS Code extension format — covering VS Code, VS Code Insiders, Cursor, Windsurf, VSCodium, and Positron — and silently force-install a malicious `.vsix` into each one. The `.vsix`, disguised as the popular `steoates.autoimport` extension, is fetched from an attacker-controlled GitHub Releases page. After installation, the downloaded file is deleted to remove evidence.

The second-stage implant is the established GlassWorm dropper: it geofences Russian-language systems, beacons to a Solana blockchain-based C2, and installs a persistent RAT including a malicious Chrome extension capable of credential theft, keylogging, and session cookie exfiltration. A developer running only Cursor as their daily editor but with VS Code also installed would find both compromised. The use of Zig — a low-level compiled language that produces compact, hard-to-reverse binaries — and Node.js native addons (`.node` files) represents a meaningful escalation over GlassWorm's previous JavaScript-only payloads.

---

## Compromised Artifacts

| Artifact | Type | Notes |
|----------|------|-------|
| `specstudio/code-wakatime-activity-tracker` | OpenVSX extension | Trojanized WakaTime impersonator; contains Zig-compiled native `.node` binaries |
| `floktokbok.autoimport` (`autoimport-2.7.9.vsix`) | VS Code/compatible extension | Second-stage GlassWorm dropper; impersonates `steoates.autoimport` |

---

## How It Worked

### Step 1 — Trojanized Activation Hook

The malicious extension mimics the WakaTime extension down to its surface API. The attack begins in the `activate()` function, which fires the moment the IDE loads the extension:

```javascript
const bw = process.platform === "win32" ? "./bin/win.node" : "./bin/mac.node";
const { install } = require(bw);
install();
```

Before any legitimate WakaTime logic runs, the extension loads a native binary from the `./bin/` directory bundled with the extension and calls `install()`. This executes outside the JavaScript sandbox with full OS-level access.

### Step 2 — Zig-Compiled Native Addon

The binaries are Node.js native addons — compiled shared libraries loaded directly into Node's runtime:

- **Windows**: `win.node` — a PE32+ DLL compiled in Zig
- **macOS**: `mac.node` — a universal Mach-O binary covering both x86_64 and arm64, compiled in Zig with debug symbols intact (revealing project path `/Users/davidioasd/Downloads/vsx_installer_zig`)

Both binaries operate outside the JavaScript sandbox and bypass JavaScript-level security controls entirely.

### Step 3 — IDE Discovery and Mass Infection

The binary enumerates all installed IDEs that use the VS Code extension format:

**Windows targets:**
```
%LOCALAPPDATA%\Programs\Microsoft VS Code\bin\code.cmd
%LOCALAPPDATA%\Programs\Microsoft VS Code Insiders\bin\code-insiders.cmd
%LOCALAPPDATA%\Programs\cursor\resources\app\bin\cursor.cmd
%LOCALAPPDATA%\Programs\windsurf\resources\app\bin\windsurf.cmd
%LOCALAPPDATA%\Programs\VSCodium\resources\app\bin\codium.cmd
%LOCALAPPDATA%\Programs\Positron\resources\app\bin\positron.cmd
%ProgramFiles%\Microsoft VS Code\bin\code.cmd
%ProgramFiles%\Positron\resources\app\bin\positron.cmd
```

**macOS targets:**
```
/Applications/Visual Studio Code.app/Contents/Resources/app/bin/code
/Applications/Visual Studio Code - Insiders.app/.../code-insiders
/Applications/Cursor.app/Contents/Resources/app/bin/cursor
/Applications/Windsurf.app/Contents/Resources/app/bin/windsurf
/Applications/VSCodium.app/Contents/Resources/app/bin/codium
/Applications/Positron.app/Contents/Resources/app/bin/positron
```

### Step 4 — Fetching and Installing the Second-Stage .vsix

With the IDE list assembled, the binary fetches the malicious extension package from an attacker-controlled GitHub Releases URL:

```
https://github[.]com/ColossusQuailPray/oiegjqde/releases/download/12/autoimport-2.7.9.vsix
```

The package impersonates `steoates.autoimport`, a legitimate and popular VS Code extension with millions of installs. The `.vsix` is written to a temporary path, then silently installed into every discovered IDE using each editor's own CLI installer. On Windows this runs via:

```
cmd.exe /d /e:ON /v:OFF /c "<ide_path> --install-extension <vsix_path>"
```

After installation, `cleanupVsix` deletes the downloaded file to remove forensic evidence.

### Step 5 — Second-Stage GlassWorm Dropper

The force-installed `autoimport-2.7.9.vsix` is the known GlassWorm dropper payload, carrying the full capability suite documented in previous campaign analyses:

- **Geofencing**: Skips execution on Russian-language systems
- **Solana blockchain C2**: Beacons to attacker infrastructure via the Solana blockchain for C2 command retrieval, making the C2 channel highly resilient to domain takedowns
- **Persistent RAT**: Installs a long-lived remote access trojan
- **Malicious Chrome extension**: Force-installs a browser extension for keylogging, session cookie theft, and browser credential harvesting

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-04-08 | Aikido Security discovers `specstudio/code-wakatime-activity-tracker` on OpenVSX with embedded Zig-compiled native dropper |
| 2026-04-08 | Aikido publishes full analysis identifying IDE mass-infection technique, IOCs, and Solana C2 second stage |
| 2026-04-08 | Attacker GitHub Releases page `ColossusQuailPray/oiegjqde` identified as distribution point for second-stage `.vsix` |

---

## Detection

```bash
# Check for the trojanized WakaTime extension in OpenVSX/VS Code extensions
# Check VS Code
code --list-extensions 2>/dev/null | grep -i "wakatime"
# Check Cursor
cursor --list-extensions 2>/dev/null | grep -i "wakatime"
# Check Windsurf
windsurf --list-extensions 2>/dev/null | grep -i "wakatime"

# Check for the force-installed malicious autoimport extension
code --list-extensions 2>/dev/null | grep "floktokbok.autoimport"
cursor --list-extensions 2>/dev/null | grep "floktokbok.autoimport"
windsurf --list-extensions 2>/dev/null | grep "floktokbok.autoimport"
codium --list-extensions 2>/dev/null | grep "floktokbok.autoimport"

# Look for suspicious .node files inside VS Code extension directories
# macOS
find "$HOME/.vscode/extensions" -name "*.node" -newer /tmp 2>/dev/null
find "$HOME/.cursor/extensions" -name "*.node" 2>/dev/null
find "$HOME/.windsurf/extensions" -name "*.node" 2>/dev/null

# Windows (PowerShell)
# Get-ChildItem "$env:USERPROFILE\.vscode\extensions" -Recurse -Filter "*.node"

# Hash check: match known-malicious binaries
# macOS mac.node
find / -name "mac.node" -exec sha256sum {} \; 2>/dev/null | grep "112d1b33dd9b0244525f51e59e6a79ac5ae452bf6e98c310e7b4fa7902e4db44"
# Windows win.node
find / -name "win.node" -exec sha256sum {} \; 2>/dev/null | grep "2819ea44e22b9c47049e86894e544f3fd0de1d8afc7b545314bd3bc718bf2e02"

# Check for known attacker GitHub repository in network/proxy logs
grep -r "ColossusQuailPray" /var/log/ 2>/dev/null

# Search for Zig project artefacts that reveal attacker infrastructure
strings "$HOME/.vscode/extensions"/**/*.node 2>/dev/null | grep -E "vsx_installer_zig|davidioasd"

# Scan for suspicious autoimport VSIX extension directory
ls "$HOME/.vscode/extensions/" | grep -i "autoimport"
ls "$HOME/.cursor/extensions/" | grep -i "autoimport"
```

---

## Remediation

1. **Remove the trojanized WakaTime extension** from all affected IDEs:
   ```bash
   code --uninstall-extension specstudio.code-wakatime-activity-tracker
   cursor --uninstall-extension specstudio.code-wakatime-activity-tracker
   windsurf --uninstall-extension specstudio.code-wakatime-activity-tracker
   codium --uninstall-extension specstudio.code-wakatime-activity-tracker
   ```
2. **Remove the force-installed malicious autoimport extension** from all IDEs:
   ```bash
   code --uninstall-extension floktokbok.autoimport
   cursor --uninstall-extension floktokbok.autoimport
   windsurf --uninstall-extension floktokbok.autoimport
   codium --uninstall-extension floktokbok.autoimport
   ```
3. **Manually inspect all IDE extension directories** for `*.node` files inside extension folders that are not from known-legitimate native extensions (e.g., language servers). Remove any suspicious entries.
4. **Rotate all credentials** that may have been accessible from any compromised IDE session:
   - SSH keys (`~/.ssh/`)
   - Git credentials and GitHub tokens
   - Cloud provider credentials (AWS, GCP, Azure)
   - API keys stored in `.env` files or IDE workspace settings
   - Browser-stored passwords and cookies (the second-stage installs a browser RAT)
   - Cryptocurrency wallet keys and seed phrases
5. **Audit installed browser extensions** for unexpected entries, especially any extension not intentionally installed by the user — the second stage force-installs a Chrome extension.
6. **Reinstall the legitimate WakaTime extension** from the official publisher if developer time tracking is needed; verify the publisher is `WakaTime` (not `specstudio`).
7. **Block the attacker GitHub repository** at the network level: `github.com/ColossusQuailPray`

---

## Lessons Learned

- **Native compiled addons as dropper indirection**: Embedding a Zig-compiled `.node` native addon lets attackers execute arbitrary OS-level code without triggering JavaScript sandbox protections or any npm lifecycle scripts. Security tooling focused solely on JS code in extensions misses this vector entirely.
- **Multi-IDE blast radius**: Attacks targeting one IDE now routinely expand to all compatible IDEs on the same machine. Developers running multiple editors (VS Code + Cursor, for instance) face compounded risk from a single malicious extension.
- **OpenVSX as an under-scrutinised vector**: OpenVSX marketplace has lower publication scrutiny than the official VS Code Marketplace, making it a preferred entry point for GlassWorm. Extensions published there can still be installed into VS Code-compatible editors.
- **Debug symbols reveal attacker tradecraft**: The macOS Zig binary was compiled with debug symbols intact, exposing the attacker's local project path and username. This is a recurring OPSEC failure in hastily deployed malware.
- **Blockchain C2 raises the stakes for detection**: The Solana-based C2 used in the second stage is highly resistant to domain sinkholing or takedowns. Once the implant is installed, disrupting attacker communication requires blocking at the blockchain protocol layer, which is impractical for most defenders.
- **Extension impersonation succeeds on familiarity**: WakaTime is a widely trusted and commonly installed tool. Developers who see a WakaTime-looking extension are unlikely to scrutinise it further, especially if it surfaces in a search or recommendation feed.

---

## Related Incidents

- [./2026-03-glassworm-chrome-rat.md](./2026-03-glassworm-chrome-rat.md) — Previous GlassWorm campaign: multi-stage RAT + force-installed Chrome extension
- [./2026-03-glassworm-returns.md](./2026-03-glassworm-returns.md) — GlassWorm Unicode mass campaign across 150+ repos
- [./2026-03-glassworm-canisterworm.md](./2026-03-glassworm-canisterworm.md) — GlassWorm + CanisterWorm joint campaign; Solana C2; 45 packages
- [./2026-03-fast-draft-bloktrooper.md](./2026-03-fast-draft-bloktrooper.md) — BlokTrooper RAT via OpenVSX; similar extension-marketplace vector
- [./2026-03-iolitelabs-vscode-backdoor.md](./2026-03-iolitelabs-vscode-backdoor.md) — Dormant VS Code publisher hijack with multi-stage backdoor
