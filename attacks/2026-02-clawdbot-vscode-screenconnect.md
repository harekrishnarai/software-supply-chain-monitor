# Fake ClawdBot Agent VS Code Extension — ScreenConnect RAT via Brand Impersonation

**Date:** February 2026
**Ecosystem:** VS Code Marketplace
**Severity:** High
**Type:** Brand impersonation / Trojan / RAT
**Sources:**
- [Aikido Security — Fake Clawdbot VS Code Extension Installs ScreenConnect RAT](https://www.aikido.dev/blog/fake-clawdbot-vscode-extension-malware)

---

## Summary

On January 27, 2026, Aikido's malware detection system flagged a VS Code extension called "ClawdBot Agent" in the Visual Studio Code Marketplace. The extension was a fully functional trojan: it presented a working AI coding assistant on the surface while silently deploying ScreenConnect (ConnectWise), a legitimate remote desktop administration tool, onto Windows machines the moment VS Code launched.

The real Clawdbot team had never published an official VS Code extension — the attackers simply claimed the name first, capitalizing on developer searches for AI coding tools. The technique exploits developer trust in both the VS Code Marketplace and well-known AI assistant brands. Microsoft moved quickly to remove the extension after the disclosure.

The attack is notable for weaponizing a legitimate remote administration tool (RAT-as-a-service) rather than custom malware, allowing ScreenConnect's trusted reputation to evade security tooling. This follows a broader pattern of supply chain attackers using IT management software — ScreenConnect, AnyDesk, and TeamViewer — to establish persistent access that endpoint detection frequently allows through.

---

## Compromised Artifacts

| Artifact | Type | Details |
|---|---|---|
| `ClawdBot Agent` | VS Code Marketplace Extension | Malicious impersonation of the Clawdbot AI assistant; deployed ScreenConnect RAT |

---

## How It Worked

### Entry Point — Marketplace Squatting via Brand Impersonation

The attackers registered a VS Code Marketplace extension under the name "ClawdBot Agent" before the legitimate Clawdbot team could do so. The extension was fully functional as an AI coding assistant, meaning users had no immediate reason to be suspicious. A developer searching for "Clawdbot VS Code" would find this malicious extension first.

### Payload Mechanics

On every VS Code startup, the extension executed its malicious routine:

1. **Remote configuration fetch**: The extension contacted `clawdbot.getintwopc[.]site` to download a `config.json` file containing the payload URL and execution parameters.
2. **Binary download and execution**: The config.json directed the extension to download and run a binary named `Code.exe`.
3. **ScreenConnect installation**: `Code.exe` silently installed a customised ScreenConnect (ConnectWise Remote Access) client, establishing a persistent remote desktop channel back to the attacker's infrastructure.

The attack targeted **Windows only**. The AI assistant functionality was preserved so infected users saw no error or unusual behavior.

### C2 / Persistence

ScreenConnect is a legitimate remote administration platform. Once installed, it communicated to the attacker's ConnectWise tenant, giving the attacker persistent interactive remote access to the developer's machine — including access to source code, credentials, CI/CD tokens, cloud configuration files, and anything else visible on the workstation.

The persistence mechanism was ScreenConnect's own startup routine (registered as a Windows service), making it appear like legitimate IT support tooling.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2026-01-27 08:46 UTC | Aikido malware detection system flags "ClawdBot Agent" extension |
| 2026-01-27 | Extension confirmed as trojan; payload analysis completed |
| 2026-02-02 | Aikido publishes full disclosure blog post |
| 2026-02-02 | Microsoft removes "ClawdBot Agent" extension from VS Code Marketplace |

---

## Detection

```bash
# Check for installed ScreenConnect RAT client (Windows)
# Malicious install path
ls "C:\Program Files (x86)\ScreenConnect Client (083e4d30c7ea44f7)\"

# Check for running ScreenConnect processes
tasklist | findstr -i "screenconnect"

# Check Windows services for ScreenConnect
sc query type= all | findstr -i "screenconnect"
Get-Service | Where-Object {$_.DisplayName -like "*ScreenConnect*"}

# Check VS Code extensions for the malicious ClawdBot extension
code --list-extensions | grep -i "clawdbot"
# Or on Windows:
code --list-extensions | findstr /i "clawdbot"

# Check extension directories for the malicious extension
ls ~/.vscode/extensions/ | grep -i "clawdbot"
# Windows:
dir "%USERPROFILE%\.vscode\extensions\" | findstr /i "clawdbot"

# Check for C2 domain in DNS cache or network logs
ipconfig /displaydns | findstr "getintwopc"
# macOS/Linux DNS lookup evidence
grep "getintwopc" /var/log/syslog 2>/dev/null

# Check for config.json fetch evidence in browser/extension logs
grep -r "clawdbot.getintwopc" ~/.vscode/ 2>/dev/null
```

---

## Remediation

1. **Remove the malicious extension** from VS Code immediately:
   ```bash
   code --uninstall-extension <clawdbot-agent-extension-id>
   ```
   Or navigate to Extensions panel → find "ClawdBot Agent" → Uninstall.

2. **Remove the ScreenConnect RAT client** installed by the malware:
   - Navigate to `C:\Program Files (x86)\ScreenConnect Client (083e4d30c7ea44f7)\`
   - Run the uninstaller if present, or delete the directory
   - Remove any associated Windows services: `sc delete ScreenConnect*`

3. **Revoke and rotate all credentials** accessible from the compromised machine: cloud provider keys, SSH keys, npm/PyPI tokens, GitHub tokens, API keys stored in `.env` files or IDE settings.

4. **Audit source code** in repositories accessible from the affected workstation for any unauthorized commits or changes made during the attacker's ScreenConnect session.

5. **Check for lateral movement**: ScreenConnect access gives the attacker interactive access; review other systems on the same network for signs of access from the compromised developer machine.

6. **Block the C2 domain** at DNS/firewall level: `clawdbot.getintwopc[.]site` and `*.getintwopc.site`.

7. **Report to Microsoft**: If you encountered this extension, report it via the VS Code Marketplace abuse reporting mechanism.

---

## Lessons Learned

- **AI tool brand impersonation is a high-value attack vector**: Developers eagerly search for AI coding assistants, creating a large attack surface for extension squatting. The VS Code Marketplace lacks strong identity verification for publishers.
- **Functional malware avoids suspicion**: Because the AI features worked correctly, users had no reason to investigate further or leave negative reviews.
- **Legitimate RAT tools bypass endpoint security**: ScreenConnect, AnyDesk, and TeamViewer are trusted by many endpoint security products. Attackers increasingly use these instead of custom C2 to avoid detection.
- **Marketplace squatting requires proactive brand protection**: Teams building products with developer tooling integrations should pre-register their brand name in all major marketplaces (VS Code, OpenVSX, Chrome Web Store) even before publishing official extensions.
- **Single-step persistence**: Activation occurs on every VS Code launch without requiring user interaction — no `postinstall` hook that might be audited, just a startup side effect of opening the IDE.

---

## Related Incidents

- [./2026-03-iolitelabs-vscode-backdoor.md](./2026-03-iolitelabs-vscode-backdoor.md) — Dormant VS Code publisher account hijack targeting Solidity developers
- [./2026-03-fast-draft-bloktrooper.md](./2026-03-fast-draft-bloktrooper.md) — OpenVSX extension delivering BlokTrooper RAT
- [./2026-02-cline-openclaw.md](./2026-02-cline-openclaw.md) — Cline npm package silently deploying OpenClaw AI agent
