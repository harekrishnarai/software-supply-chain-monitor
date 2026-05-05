# GenAI Chrome Extension RAT Campaign — 18 Malicious AI Extensions Exfiltrate Developer Secrets

**Date:** April 2026
**Ecosystem:** Chrome Web Store
**Severity:** High
**Type:** Infostealer / RAT / Spyware / Adversary-in-the-Browser (AitB)
**Sources:**
- [Unit 42 — That AI Extension Helping You Write Emails? It's Reading Them First](https://unit42.paloaltonetworks.com/high-risk-gen-ai-browser-extensions/)

---

## Summary

Unit 42 researchers identified 18 malicious Chrome extensions disguised as AI productivity tools and reported them to Google. The extensions use the Chrome Web Store as a trusted distribution channel to deliver RATs, infostealers, spyware, and search hijackers targeting developers, AI users, and enterprise employees.

The campaign is notable for its deliberate exploitation of the GenAI productivity tool trust model. Attackers position the extensions as enhancers for workflows involving ChatGPT, Claude, Gemini, and Gmail, then leverage the broad browser permissions these tools request to exfiltrate AI API keys, email content, passwords, and browsing sessions. At least one extension — *Chrome MCP Server - AI Browser Control* — specifically impersonates an AI developer tool using the Model Context Protocol (MCP) framework, delivering a full RAT capable of remote JavaScript execution, form filling, screenshot capture, and Chrome Debugger Protocol attachment.

Multiple samples across the campaign show AI-generated code fingerprints — identical section divider comments, template-based scaffolding, and formulaic code structure — indicating that threat actors are using LLMs to accelerate malware production at scale. Google removed or warned the owners of all 18 reported extensions. The total affected user count across the campaign exceeds 95,000.

---

## Compromised Artifacts

| Extension Name | Extension ID | Version | Users | Primary Threat |
|---------------|--------------|---------|-------|----------------|
| Chrome MCP Server - AI Browser Control | fpeabamapgecnidibdmjoepaiehokgda | 1.0.1 | ~1,000 | RAT via WebSocket C2 |
| browser cash | oaldjcdohhhibelagdhoahbedekfjjjf | 1.0.3 | ~30,000 | Unknown |
| Anker AIME Copilot | nbflcljmdbibeoaipongjgfmbapanipm | 1.0.2 | ~7,000 | C2 at 172.16.18.184:5443 |
| Nano Banana | ffocfibjgakneigiajpccfcdmomlbapo | 1.3.0 | ~4,000 | Remote quota tracking |
| Text Summarizer | npifianbfjhobabjjpfdjjihgbdnbojh | 1.1.0 | ~5,000 | WebSocket C2 |
| Google AI | pfdmleklaejjccgfhoeafapbhkjipcnj | 1.2 | ~2,000 | Malicious impersonation |
| 会译 AI Translation Agent | dgeiaiglmhdhajbpfbmajaajdlfdinpi | 1.6.16 | ~20,000 | Proxy hijacker / spyware |
| AI Agent | hnppehcgmflfkcdkbkaeemjfngffmeag | 1.9 | ~1,000 | C2 at 199.80.55.27:3130 |
| Notion中文版 | ljlhpcabhpjdlcjhbmgjigfceppgabmk | 1.1.0 | ~3,000 | Sends data to notionapp.cn |
| Notion中文版 | pdahnbohfcekobflehebdkoemnmmempk | 1.0.6 | ~1,511 | Malicious impersonation |
| NotionAI插件 | jndldoeopjgmpakgmieaeeelhnjnfgkj | 1.1.4 | ~192 | Malicious impersonation |
| Agent Risk Reminder Remover | bonhfflnjgdbnhcpjemkknlhimceckgb | 1.0.1 | ~563 | Unknown |
| Reverse Recruiting - AI Job Application Assistant | iefpkdilnfhogjbkhgnliaomoldgkdlj | 0.3.0 | ~1 | OpenAI/Gemini/Claude API key theft |
| Chat AI for Chrome | jhhjbaicgmecddbaobeobkikgmfffaeg | 1.1.2 | ~2,000 | Search hijacker / persistent tracker |
| [Redacted brand]: AI Photo, Video | hmkcidjcpomiegnklmplkimmbcbklglb | 1.0 | ~579 | Brand impersonator / install-time payload |
| Ask AI - GPT chat | cjmhegifablecgkkncjddcgkjmgoacfd | 1.1 | ~1,000 | C2 at vomet.ru |
| Picsart: AI Photo Video Editor | dcjfbgppfdokmjgajnnkgdmkdeiloigh | 1.1 | ~608 | Brand impersonator |
| Supersonic AI | eebihieclccoidddmjcencomodomdoei | 1.0.6 | ~17 | Gmail email exfiltration |

---

## How It Worked

### Malware Category 1 — RAT: Chrome MCP Server - AI Browser Control

The highest-severity extension impersonates an AI browser automation tool using the Model Context Protocol (MCP) framework. Its Chrome Web Store listing falsely claims "100% local processing — your data never leaves your browser" and "No external servers required for core functionality." In practice, the extension hardcodes a WebSocket connection to `wss://mcp-browser.qubecare.ai/chrome` and accepts over 30 remote commands:

- Executing arbitrary JavaScript via `new Function()` in the context of any authenticated browser tab
- Attaching the Chrome Debugger Protocol to intercept decrypted HTTPS traffic
- Filling out forms, capturing screenshots, and reading browsing history

The WebSocket connection reestablishes automatically across network disruptions and browser restarts via the service worker. The `new Function()` execution pattern means an attacker can run arbitrary code inside the victim's active banking, corporate VPN, or email session.

### Malware Category 2 — Adversary-in-the-Browser (AitB): Supersonic AI

Supersonic AI presents as an AI-powered email assistant for Gmail and Outlook. Its content script passively monitors DOM changes to collect every displayed email (read, sent, or in-compose) and exfiltrates the full text to an external server (`gosupersonic.email`). Because the data is read from the rendered DOM rather than from intercepted network traffic, the attack bypasses network-layer security controls entirely. Sandbox analysis confirmed OTP codes included in emails were captured and transmitted in plaintext.

### Malware Category 3 — Infostealer: Reverse Recruiting - AI Job Application Assistant

This extension requests `<all_urls>` permissions for legitimate form autofill purposes. In addition to its stated functionality, `optimized-api.js` reads the user's OpenAI, Gemini, and Claude API keys from `chrome.storage.sync` and forwards them to `api.reverserecruiting.io` in custom HTTP headers on every request. The extension also transmits name, email, phone, LinkedIn URL, salary expectations, education, and resume to the same backend via `profile-sync.js`.

### Malware Category 4 — Search Hijacker: Chat AI for Chrome

On installation, the extension generates a unique tracking identifier and stores it in three redundant persistence layers: `chrome.cookies`, `window.localStorage`, and `chrome.storage.sync` (which syncs across all Chrome instances signed into the same Google account). A listener on `chrome.cookies.onChanged` detects and recreates the cookie if deleted. The extension then silently replaces the victim's default search engine via `chrome_settings_overrides`, routing all searches through `chatgptforchrome.com`. Because the ID is synced across devices, clearing cookies on one machine does not eliminate tracking.

### Malware Category 5 — Spyware: 会译 AI Translation Agent

The Chinese-language translation extension requests permissions far exceeding what translation requires: `chrome.webRequest.onCompleted` listeners for all HTTP requests, `chrome.proxy.settings.set()` access, and a fetched PAC (proxy auto-configuration) script. On startup it downloads `proxy.pac` from `yiban.io/extension/proxy.pac?t=huiyi` and applies it via `chrome.proxy.settings.set()`. The publisher can modify the PAC script at any time — with no extension store update required — to selectively route any subset of user traffic through attacker-controlled infrastructure.

### AI-Generated Malware

A campaign by the same threat actors (the "10xprofit" affiliate hijacking group) documented by Socket Research runs six additional extensions that inject affiliate tags across e-commerce platforms. Unit 42's analysis of these extensions found AI-generated code fingerprints across all six: identical section divider comments, template-based scaffolding, and formulaic code structure. This points to LLM-assisted malware production enabling rapid campaign scaling.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| August 2025 | Early campaign activity: AI summary extensions exfiltrating data to low-reputation domains; adware via hidden iframes; cursor extensions delivering PUPs |
| September 2025 | Prompt and search hijackers redirecting queries to attacker-controlled domains |
| February 2026 | MCP-themed RAT (Chrome MCP Server) deployed targeting AI developers |
| 2026-04-30 | Unit 42 publishes full analysis covering 18 high-risk extensions; all 18 reported to Google |
| 2026-04-30 | Google removes or issues policy warnings to owners of all 18 reported extensions |

---

## Detection

```bash
# Check for known malicious extension IDs in Chrome profile
# (Linux/macOS path — adjust for your OS and Chrome variant)
for EXT_ID in \
  fpeabamapgecnidibdmjoepaiehokgda \
  oaldjcdohhhibelagdhoahbedekfjjjf \
  nbflcljmdbibeoaipongjgfmbapanipm \
  ffocfibjgakneigiajpccfcdmomlbapo \
  npifianbfjhobabjjpfdjjihgbdnbojh \
  pfdmleklaejjccgfhoeafapbhkjipcnj \
  dgeiaiglmhdhajbpfbmajaajdlfdinpi \
  hnppehcgmflfkcdkbkaeemjfngffmeag \
  ljlhpcabhpjdlcjhbmgjigfceppgabmk \
  pdahnbohfcekobflehebdkoemnmmempk \
  jndldoeopjgmpakgmieaeeelhnjnfgkj \
  bonhfflnjgdbnhcpjemkknlhimceckgb \
  iefpkdilnfhogjbkhgnliaomoldgkdlj \
  jhhjbaicgmecddbaobeobkikgmfffaeg \
  hmkcidjcpomiegnklmplkimmbcbklglb \
  cjmhegifablecgkkncjddcgkjmgoacfd \
  dcjfbgppfdokmjgajnnkgdmkdeiloigh \
  eebihieclccoidddmjcencomodomdoei; do
  CHROME_EXT_DIR="$HOME/Library/Application Support/Google/Chrome/Default/Extensions"
  if [ -d "$CHROME_EXT_DIR/$EXT_ID" ]; then
    echo "FOUND malicious extension: $EXT_ID"
  fi
done

# macOS: check enterprise Chrome extensions directory
ls ~/Library/Application\ Support/Google/Chrome/Default/Extensions/ 2>/dev/null \
  | grep -E "fpeabamapgecnidibdmjoepaiehokgda|eebihieclccoidddmjcencomodomdoei|iefpkdilnfhogjbkhgnliaomoldgkdlj|dgeiaiglmhdhajbpfbmajaajdlfdinpi|jhhjbaicgmecddbaobeobkikgmfffaeg"

# Check network logs for C2 domains
grep -iE "mcp-browser\.qubecare\.ai|gosupersonic\.email|api\.reverserecruiting\.io|chatgptforchrome\.com|yiban\.io|vomet\.ru|xuix\.top|banana\.summarizer\.one" \
  /var/log/*.log 2>/dev/null

# Check for C2 IPs in network logs
grep -E "172\.16\.18\.184|199\.80\.55\.27|158\.160\.66\.115" \
  /var/log/firewall.log /var/log/pf.log 2>/dev/null

# Check Chrome proxy settings (detects PAC script hijacking)
# Run in Chrome DevTools console or chrome://net-internals/#proxy
# Look for proxy PAC URL pointing to yiban.io

# Enterprise: query Chrome Browser Cloud Management or endpoint DLP
# for extensions making outbound requests to the IOC domains listed above
```

---

## Remediation

1. **Remove all identified malicious extensions** from all Chrome instances (including synced devices) using the extension IDs in the Compromised Artifacts table.
2. **Rotate all AI API keys** (OpenAI, Gemini, Claude, and others) if the Reverse Recruiting extension or any extension requesting access to API key storage was installed. Treat all stored keys as compromised.
3. **Rotate Gmail and Outlook credentials** if Supersonic AI was installed — all emails displayed during the installation window should be considered exfiltrated.
4. **Remove the persistent tracking cookie** created by Chat AI for Chrome: uninstall the extension, then clear cookies AND `chrome.storage.sync` data (standard cookie clearing alone is insufficient due to cross-device sync).
5. **Audit Chrome proxy settings** (`chrome://net-internals/#proxy`) to confirm no malicious PAC script from `yiban.io` is still active.
6. **Review browser extension inventory** across the organization using Chrome Browser Cloud Management or an equivalent MDM solution; enforce an extension allowlist.
7. **Rotate any credentials accessible during authenticated browser sessions** on affected machines — this includes corporate SSO tokens, banking sessions, and any secrets visible in-browser during the installation period.
8. **File a Google Security Incident** for confirmed enterprise breaches and engage Unit 42 Incident Response if a RAT (Chrome MCP Server) was installed.

---

## Lessons Learned

- **AI productivity lures lower security vigilance**: Users who believe an extension needs broad permissions "for AI" are less likely to challenge excessive `<all_urls>`, `debugger`, or `webRequest` grants. Threat actors are deliberately exploiting the AI permission ambiguity pattern.
- **DOM-based exfiltration bypasses network DLP**: Extensions that read email and form data directly from the rendered DOM generate no suspicious network requests for data collection — only outbound POST requests that look like legitimate API calls. Network-layer controls alone are insufficient.
- **PAC-script hijacking is a persistent, updatable backdoor**: A proxy auto-configuration script loaded from a remote server can route traffic to new destinations at any time, without a Chrome Web Store update. This is a hard-to-detect persistent control channel.
- **Chrome storage sync enables cross-device persistence**: Tracking identifiers stored in `chrome.storage.sync` survive cookie clears and persist across all signed-in devices, making full remediation harder than it appears.
- **AI-accelerated malware production scales campaigns**: Evidence of LLM-generated code across six related extensions suggests threat actors can now produce and deploy multiple extension variants rapidly, saturating the Chrome Web Store review process.
- **MCP developer tooling is a high-value target**: Extensions that impersonate MCP framework tools specifically target AI developers with access to AI API keys, proprietary prompts, and cloud infrastructure — the exact credentials most valuable for lateral movement.

---

## Related Incidents

- [./2026-02-clawdbot-vscode-screenconnect.md](./2026-02-clawdbot-vscode-screenconnect.md) — RAT disguised as AI assistant; VS Code Marketplace distribution
- [./2026-03-iolitelabs-vscode-backdoor.md](./2026-03-iolitelabs-vscode-backdoor.md) — Dormant publisher account hijack; VS Code extension backdoor
- [./2026-03-fast-draft-bloktrooper.md](./2026-03-fast-draft-bloktrooper.md) — OpenVSX/VS Code RAT with clipboard monitor and wallet stealer
- [./2026-04-glassworm-zig-dropper.md](./2026-04-glassworm-zig-dropper.md) — Mass IDE infection via trojanized extension
