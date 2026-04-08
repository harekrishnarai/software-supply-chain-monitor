# GlassWorm & CanisterWorm

**Date:** October 2025 (first wave) → March 2026 (major escalation)
**Ecosystem:** npm, GitHub repositories, VS Code / OpenVSX, PyPI
**Severity:** Critical
**Type:** Multi-stage RAT / worm / credential theft / hardware wallet phishing
**Threat Actor:** GlassWorm (Unicode campaign) + TeamPCP (CanisterWorm, linked to Trivy attack)
**Sources:**
- [aikido.dev — GlassWorm Returns: Invisible Unicode (Mar 13, 2026)](https://www.aikido.dev/blog/glassworm-returns-unicode-attack-github-npm-vscode)
- [aikido.dev — GlassWorm Strikes React Native Packages (Mar 16, 2026)](https://www.aikido.dev/blog/glassworm-strikes-react-packages-phone-numbers)
- [aikido.dev — GlassWorm Hides a RAT Inside a Malicious Chrome Extension (Mar 18, 2026)](https://www.aikido.dev/blog/glassworm-chrome-extension-rat)
- [aikido.dev — TeamPCP deploys CanisterWorm on NPM following Trivy compromise (Mar 20, 2026)](https://www.aikido.dev/blog/teampcp-deploys-worm-npm-trivy-compromise)
- [thehackernews.com — GlassWorm Supply-Chain Attack Abuses 72 Open VSX Extensions](https://thehackernews.com/2026/03/glassworm-supply-chain-attack-abuses-72.html)

---

## Timeline of the GlassWorm Campaign

| Date | Event |
|------|-------|
| Mar 2025 | Aikido first discovers malicious npm packages using invisible PUA Unicode characters |
| May 2025 | Aikido publishes first public writeup on invisible Unicode supply chain abuse |
| Oct 17, 2025 | 72 compromised Open VSX extensions using the same invisible Unicode technique |
| Oct 31, 2025 | GlassWorm pivots to GitHub repositories |
| Mar 3–9, 2026 | New mass wave: 151+ GitHub repositories compromised |
| Mar 12, 2026 | npm and VS Code packages added: `@aifabrix/miso-client`, `@iflow-mcp/watercrawl-*`, VS Code extension `quartz.quartz-markdown-editor` |
| Mar 13, 2026 | Aikido publishes GlassWorm Returns disclosure |
| Mar 16, 2026 | React Native packages `react-native-country-select` and `react-native-international-phone-number` backdoored |
| Mar 18, 2026 | Full 3-stage payload chain recovered and published, including Chrome Extension RAT |
| Mar 20, 2026 | TeamPCP deploys **CanisterWorm** on npm (45 packages in <60 seconds), linked to same-day Trivy attack |

---

## Part 1: Invisible Unicode Loader (GlassWorm Signature Technique)

GlassWorm's defining technique is hiding executable payloads inside **invisible Unicode characters** — specifically Unicode variation selectors (U+FE00–FE0F) and tag characters (U+E0100–E01EF). These render as nothing in every editor, terminal, and code review interface.

The decoder pattern looks like this in source code — the backtick string appears empty but is packed with invisible characters:

```js
const s = v => [...v].map(w => (
  w = w.codePointAt(0),
  w >= 0xFE00 && w <= 0xFE0F ? w - 0xFE00 :
  w >= 0xE0100 && w <= 0xE01EF ? w - 0xE0100 + 16 : null
)).filter(n => n !== null);
eval(Buffer.from(s(``)).toString('utf-8'));
```

A GitHub code search for this pattern as of March 13, 2026 returned **at least 151 matching repositories**. The true scope was larger — many had already been deleted. Notable compromised repos included:

- `pedronauck/reworm` (1,460 stars)
- `pedronauck/spacefold` (62 stars)
- `anomalyco/opencode-bench` (56 stars)
- `wasmer-examples/hono-wasmer-starter`

---

## Part 2: Full Attack Chain (React Native Packages, March 16, 2026)

### Compromised Packages

| Package | Malicious Version | Weekly Downloads (at time of attack) |
|---------|------------------|--------------------------------------|
| `react-native-international-phone-number` | 0.11.8 | 20,691 |
| `react-native-country-select` | 0.3.91 | 9,072 |

Both published within 5 minutes of each other on March 16, 2026 at ~10:49–10:54 UTC. Both share an identical `install.js` loader (SHA-256: `59221aa9623d86c930357dba7e3f54138c7ccbd0daa9c483d766cd8ce1b6ad26`).

### Stage 1: Preinstall Loader

A `preinstall` hook runs `node install.js` during `npm install`. The loader:
1. Checks for a **Russian locale** — if `ru_RU`, `ru-RU`, `Russian`, or Russian timezone signals are detected, execution stops (CIS-region exclusion)
2. Checks `~/init.json` for a rate-limit timestamp — stops if executed less than 2 hours ago
3. Queries **Solana blockchain** for a C2 URL stored in a transaction memo field:
   - Wallet `6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ` (obfuscated preinstall variant)
   - Wallet `BjVeAjPrSKFiingBn4vZvghsGj9KCE8AJVtbc9S8o8SC` (Unicode loader variant)
4. Uses the Solana memo as a **covert dead-drop** — the C2 URL is permanent, publicly visible on-chain, and cannot be taken down

The Solana memo observed: `{"link":"aHR0cDovLzQ1LjMyLjE1MC4yNTEvM2U0VGc4ViUyRjhhQ21PSktpcEFTQURnJTNEJTNE"}`
Decoded URL: `http://45.32.150.251/3e4Tg8V%2F8aCmOJKipASADg%3D%3D`

### Stage 2: Credential Theft & Dropper

Stage 2 is a full data-theft framework that stages everything under `%TEMP%\hJxPxpHP\`, then POSTs a zipped bundle to `http://217.69.3.152/wall`.

**Targets:**
- 71 crypto wallet browser extension IDs (MetaMask, Phantom, Coinbase, Exodus, Binance, Ronin, Keplr, etc.)
- `.txt` files from Desktop/Documents matching seed phrase keywords
- `~/.npmrc`, `NPM_TOKEN` env var (validated live against npm registry before exfiltration)
- Git credentials via `git credential fill`
- VS Code internal token storage
- AWS (`~/.aws/`), GCP (`~/.config/gcloud/`), Azure (`~/.azure/`), Docker (`~/.docker/config.json`), Kubernetes (`~/.kube/config`), SSH keys, Heroku, Terraform

**Stage 3 preparation:**
- Downloads `Assaac.exe` (Ledger/Trezor phishing binary) → `%TEMP%\SKuyzYcDD.exe`
- Downloads WebSocket RAT → `%APPDATA%\QtCvyfVWKH\index.js`
- Uses **Google Calendar** as an additional indirection layer: fetches `https://calendar.app.google/2NkrcKKj4T6Dn4uK6`, Base64-decodes the invite title to get the next payload URL on `45.32.150.251`

### Stage 3a: Hardware Wallet Phishing (Ledger & Trezor)

On machines where `%APPDATA%\Ledger Live` exists, `Assaac.exe` is dropped and persisted via `HKCU\...\Run\UpdateLedger`. The binary:
- Re-applies the CIS-region exclusion (checks `ipapi.co/xml`)
- Watches for USB connections via WMI (`Win32_PnPEntity` events)
- When a Ledger or Trezor is plugged in, opens a fake "firmware error" window with 24-word recovery phrase fields
- Kills real Ledger Live processes via `Process.GetProcessesByName` and forces the window back open if closed
- Sends submitted seed phrases to `45.150.34.158`

### Stage 3b: Persistent WebSocket RAT

The RAT (`%APPDATA%\QtCvyfVWKH\index.js`) persists via two mechanisms:
- Scheduled task `UpdateApp` (highest privileges, runs at startup)
- `HKCU\...\Run\UpdateApp` PowerShell launcher

**C2 resolution:** Uses DHT lookup for pinned public key `3c90fa0e84dd76c94b1468f38ed640945d72bc12`, bootstrapped via `dht.libtorrent.org`, `router.bittorrent.com`, `router.utorrent.com`. Falls back to Solana memo if DHT fails.

**C2 commands:**
- `start_hvnc` / `stop_hvnc` — Hidden remote desktop
- `start_socks` / `stop_socks` — Turns victim into SOCKS proxy for attacker
- `reget_log` — Full browser credential pipeline
- `get_system_info` — OS, hardware, network profile
- `command` — `eval()` arbitrary JavaScript (full RCE)

**Malicious Chrome extension** (`Google Docs Offline` v1.95.1) is force-installed via `w.node`. It:
- Polls a C2 via Solana memo (wallet `DSRUBTziADDHSik7WQvSMjvwCHFsbsThrbbjWMoJPUiW`) every 5–30 seconds
- Keylogger (`startkeylogger`) — hooks `keydown`, `keyup`, `keypress`, `input`, `change`, `focus`, `blur`
- Cookie theft, localStorage dump, screenshot capture, history/bookmarks exfil
- Watches `.bybit.com` for session cookies; sends auth-detected webhook to `/api/webhook/auth-detected`

---

## Part 3: CanisterWorm (TeamPCP, March 20, 2026)

A direct follow-up to the Trivy GitHub Actions attack (same day, same threat actor: **TeamPCP**). Detected by Aikido at 20:45 UTC on March 20, 2026.

### Compromised Packages

| Scope / Package | Count | Notes |
|----------------|-------|-------|
| `@EmilGroup` scope | 28 packages | Spread in <60 seconds |
| `@opengov` scope | 16 packages | |
| `@teale.io/eslint-config` | v1.8.11, v1.8.12 | Self-propagating upgrade |
| `@airtm/uuid-base32` | flagged | |
| `@pypestream/floating-ui-dom` | flagged | |

### Three-Stage Architecture

**Stage 1 — postinstall loader (`index.js`)**
- Decodes a base64 Python script and writes it to `~/.local/share/pgmon/service.py`
- Creates a `systemd` user service (`pgmon.service`) with `Restart=always`; no root required
- Disguises itself as PostgreSQL tooling: `pgmon`, `pglog`, `.pg_state`
- Silent on all platforms; **only activates on Linux with systemd**

**Stage 2 — Python backdoor (`service.py`)**
- Sleeps **5 minutes** before first beacon (outlasts most sandboxes)
- Polls an **ICP (Internet Computer) canister** at `https://tdtqy-oyaaa-aaaae-af2dq-cai.raw.icp0.io/` every ~50 minutes
- The canister returns a URL to a binary payload (`/tmp/pglog`); if the URL contains `youtube.com`, the implant stays dormant
- Supports remote payload rotation — the operator swaps the URL in the canister, all infected machines pick up the new binary on next poll
- Kill switch: pointing the canister to a YouTube URL disarms all infected hosts instantly

**Stage 3 — Worm (`deploy.js`)**

Initially a manually-run tool (first wave), upgraded to **self-propagating** in `@teale.io/eslint-config` v1.8.11 (one hour after initial wave):

```js
function findNpmTokens() {
  // Scrapes: ~/.npmrc, ./.npmrc, /etc/npmrc
  // Environment: NPM_TOKEN, NPM_TOKENS, *NPM*TOKEN*
  // npm config get //registry.npmjs.org/:_authToken
}
```

Once tokens are found, the worm:
1. Calls `/-/whoami` to identify the token owner
2. Enumerates all packages via `maintainer:<username>` search (batches of 250)
3. Fetches the original README to preserve appearances
4. Bumps the patch version of each target package
5. Rewrites `package.json` and publishes with `--tag latest` in a detached background process

**28 packages infected in under 60 seconds.**

---

## Indicators of Compromise

### Network / C2
| Address | Role |
|---------|------|
| `45.32.150.251` | Stage 2 payload delivery, Stage 3 WebSocket RAT (:4787) |
| `217.69.3.152` | Stage 2 exfiltration (`/wall`), Stage 3 browser creds (`/log`) |
| `217.69.0.159:10000` | DHT bootstrap node |
| `45.150.34.158` | Ledger/Trezor seed phrase exfiltration |
| `tdtqy-oyaaa-aaaae-af2dq-cai.raw.icp0.io` | CanisterWorm ICP C2 dead-drop |

### Solana Wallets
- `BjVeAjPrSKFiingBn4vZvghsGj9KCE8AJVtbc9S8o8SC` — Unicode loader / Stage 3 DHT fallback
- `6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ` — Preinstall loader
- `DSRUBTziADDHSik7WQvSMjvwCHFsbsThrbbjWMoJPUiW` — Chrome extension C2

### File Hashes (SHA-256)
| Hash | File |
|------|------|
| `59221aa9623d86c930357dba7e3f54138c7ccbd0daa9c483d766cd8ce1b6ad26` | Shared loader (`install.js`) |
| `06fab21dc276e3ab9b5d0a1532398979fd377b080c86d74f2c53a04603a43b1d` | `Assaac.exe` (Ledger/Trezor phishing) |
| `f171c383e21243ac85b5ee69821d16f10e8d718089a5c090c41efeaa42e81fca` | `c_x64.node` (browser credential stealer) |
| `43253a888417dfab034f781527e08fb58e929096cb4ef69456c3e13550cb4e9e` | `data` (Chrome App-Bound Encryption bypass) |

### Key File Paths
- `%TEMP%\hJxPxpHP\` — Stage 2 credential staging
- `%APPDATA%\QtCvyfVWKH\index.js` — Stage 3 RAT
- `~/.local/share/pgmon/service.py` — CanisterWorm Python backdoor
- `~/.config/systemd/user/pgmon.service` — CanisterWorm persistence
- `%LOCALAPPDATA%\Google\Chrome\jucku\` — Force-installed malicious extension

---

## Detection

```bash
# Check for compromised npm packages
npm ls react-native-international-phone-number
npm ls react-native-country-select
npm ls @teale.io/eslint-config
npm ls @airtm/uuid-base32

# Check for CanisterWorm systemd persistence
systemctl --user status pgmon.service
ls ~/.local/share/pgmon/
ls ~/.config/systemd/user/pgmon.service

# Check for GlassWorm invisible Unicode in source files
grep -rP '[\xEF\xB8\x80-\xEF\xB8\x8F]' --include='*.js' .
```

Scan repos with [Aikido Safe Chain](https://www.aikido.dev/blog/glassworm-chrome-extension-rat) for invisible Unicode detection.

---

## Remediation

1. Remove all compromised packages immediately
2. **Kill and disable the systemd service:** `systemctl --user stop pgmon && systemctl --user disable pgmon && rm -f ~/.config/systemd/user/pgmon.service ~/.local/share/pgmon/service.py`
3. Rotate all npm tokens found in `~/.npmrc` and CI/CD pipelines
4. Check npm publish history for unauthorized versions on packages you maintain: `npm view <pkg> time --json`
5. Inspect all Chrome extension installations — remove `jucku` (Windows) or `myextension` (macOS) from Chrome profile
6. Rotate all cloud credentials (AWS, GCP, Azure) — assume all were exfiltrated
7. Check for the scheduled task `UpdateApp` and registry persistence keys
8. If `Ledger Live` is installed, rotate seed phrases immediately
