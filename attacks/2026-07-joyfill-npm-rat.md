# @joyfill npm Supply Chain Attack — 5-Stage Blockchain-C2 RAT via Compromised Beta Releases

**Date:** July 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Compromised Package / Import-time Execution / Blockchain C2 / Socket.IO RAT / Worm
**Sources:**
- [StepSecurity — Compromised npm Packages: @joyfill/components and @joyfill/layouts Ship an Obfuscated Remote Access Trojan](https://www.stepsecurity.io/blog/joyfill-npm-supply-chain-compromise)

---

## Summary

On July 28, 2026, malicious prerelease versions of `@joyfill/components` and `@joyfill/layouts` — two packages from the Joyfill form-builder ecosystem — were published to npm carrying an identical, heavily obfuscated five-stage malware chain. Both packages are legitimate projects that were hijacked; the malicious code lives only in the published tarballs and does not appear in the source repositories.

The payload executes **at import time, not at install time** — `package.json` contains no lifecycle hooks, so `npm install --ignore-scripts` provides zero protection. The moment an application imports the package (a unit test, a bundler run, a dev server, or a production deploy), the loader fires. It resolves its live C2 server through a chain of **Tron → BNB Smart Chain → Aptos** blockchain transactions, keeping all attacker infrastructure off static indicator lists. The final payload is a full **Socket.IO remote access trojan** with arbitrary JavaScript execution capability, plus a staged Python credential stealer targeting browser wallets, Git credentials, and OS keychains. A worm propagation mechanism infects `npm/lib/cli.js` globally, ensuring every subsequent npm invocation re-runs the malware.

The campaign was tracked under tag `A9-0135-3`. StepSecurity confirmed the compromise by installing each affected version alongside a clean sibling and diffing the compiled entry bundles on a hardened runner.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Notes |
|---------|---------------------|-------|
| `@joyfill/components` | `4.0.0-rc24-2773-beta.4`, `4.0.0-rc24-2773-beta.5`, `4.0.0-rc24-2773-beta.6` | Obfuscated payload appended after ~3.2 MB of legitimate React code in `dist/index.js`, `dist/index.esm.js`, `dist/joyfill.min.js` |
| `@joyfill/layouts` | `0.1.2-2773.beta.0`, `0.1.2-2773.beta.1`, `0.1.2-2773.beta.2` | Obfuscated payload spliced into `dist/index.cjs.js` and `dist/index.es.js:862`; malicious tarball is ~45 KB vs ~322 KB for clean sibling |

All 6 malicious versions share an identical obfuscated payload block — strong evidence of a single injection actor.

---

## How It Worked

### Stage 1: Import-time Obfuscated Loader

The payload is compiled into the package entry bundles. On import, a seeded character-shuffle decoder (`_$_1e42`) decrypts a string array and reconstructs two global assignments:

```javascript
global["!"] = "9-0135-3";  // campaign fragment → _V = "A9-0135-3"
global["r"] = require;     // stashes require under innocuous name
global["m"] = module;      // stashes module
```

The campaign tag `_V = "A9-0135-3"` identifies this as the npm distribution vector. A two-step Function-constructor ladder then bootstraps Stage 1 without `Function` or `eval` ever appearing as literals — the word `constructor` itself is recovered via the shuffle decoder. An LZ77-style dictionary decompressor decompresses the ~3.8 KB stage-1 script entirely in memory.

### Stage 2: Blockchain-based C2 Resolution

Rather than a hardcoded domain, Stage 1 reads live C2 coordinates from public blockchain infrastructure:

1. Fetches the latest outbound transaction from a **Tron address** (`TMfKQEd7TJJa5xNZJZ2Lep838vrzrs7mAP`) via `api.trongrid.io`; the transaction's `raw_data.data` field (hex-encoded, reversed) is a BNB Smart Chain transaction hash
2. Fetches that **BNB Smart Chain transaction** via `bsc-dataseed.binance.org` or `bsc-rpc.publicnode.com`; the transaction `input` field (hex-decoded, split on `?.?`) is the XOR-encrypted next stage
3. An **Aptos account** (`0xbe037400…80811e`) serves as a fallback pointer if Tron resolution fails
4. Decrypts using repeating-key XOR and `eval()`s the result

This runs for two independent branches (A and B) with different XOR keys and Tron/Aptos addresses. Branch A executes in-process; Branch B launches a detached child that survives the parent.

### Stage 3: Campaign-Gated C2 Selection

The decrypted Stage 2 reads `_V` and selects per-campaign C2 infrastructure:

```javascript
if (_V[0] == "A" || _V == "0") {   // npm campaign "A9-0135-3"
    global["_t_s"] = "http://166.88.134.62:443";  // Socket.IO C2
    global["_t_u"] = "http://166.88.134.62";       // Upload C2
} else if (!isNaN(parseInt(_V))) {
    global["_t_s"] = "http://198.105.127.210:443";
    // ...
} else {
    global["_t_s"] = "http://23.27.202.27:443";
    global["_t_u"] = "http://23.27.202.27:27017";
}
```

Stage 2 also stores its own source in `global._t_c` and records `__dirname`/`__filename` for reuse by the worm stage.

### Stage 4: Socket.IO Remote Access Trojan (~77 KB)

The resolved stage configures a full Socket.IO RAT connecting to the campaign's SOCKET_URL. On connect it fingerprints the host (OS, hostname, username, CI/build environment detection) and self-installs missing dependencies (`socket.io-client`, `axios`, `form-data`) via `npm install`. Supported command verbs include:

| Command | Behavior |
|---------|---------|
| `ss_info` | Full host report: OS, campaign tag, session ID, Node version, path info |
| `ss_ip` | Geolocation via `ip-api.com/json` |
| `ss_cb` | Clipboard theft (`pbpaste` / `xclip` / `powershell Get-Clipboard`) |
| `ss_upf` / `ss_upd` | File and directory upload to C2 (multipart POST to `/u/f`) |
| `ss_dir` | Directory listing |
| `ss_eval:` / `ss_eval64:` | Arbitrary JavaScript execution (live RAT capability) |
| `ss_inz:` / `ss_inzx:` | Inject loader into local applications (worm) |
| `~py` | Spawn detached Python stealer process |
| `ss_connect:` | Re-point agent at a new C2 |

### Stage 5: Persistence and Python Credential Stealer

The RAT persists by injecting a self-reloading stub (tagged `/*C250617A*/…/*RS260605*/`) into files developer tools load automatically:

- VS Code, Cursor, Antigravity: `resources/app/node_modules/@vscode/deviceid/dist/index.js`
- Discord: `modules/discord_desktop_core[-1]/discord_desktop_core/index.js`
- GitHub Desktop: `resources/app/main.js`
- **npm CLI** (`<npm root -g>/npm/lib/cli.js`): the worm amplifier — once patched, every subsequent `npm` invocation re-executes the malware and can re-infect packages built or published from that machine

The Python stealer (staged under `%USERPROFILE%\.npm` or `/tmp/.npm`, encrypted archive) targets browser password databases and extension wallets, Git and GitHub CLI credentials, OS keychains, and password managers.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| 2026-07-28 | Malicious `2773 beta` versions of both packages published to npm within hours of each other |
| 2026-07-28 | StepSecurity OSS AI scan flags `@joyfill/layouts@0.1.2-2773.beta.0` as CRITICAL (score 0, REJECTED verdict) |
| 2026-07-28–29 | StepSecurity detonates all 6 versions under Harden-Runner; operator C2 appears to have been taken down within hours of disclosure |
| 2026-07-30 | StepSecurity publishes full analysis |

---

## Detection

```bash
# Check lockfiles for any 2773 prerelease
grep -rEn 'joyfill.*2773' package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null

# Check installed node_modules directly
ls node_modules/@joyfill/ 2>/dev/null | grep -E "components|layouts"
node -e "const v=require('@joyfill/components/package.json').version; console.log('joyfill/components:', v)"
node -e "const v=require('@joyfill/layouts/package.json').version; console.log('joyfill/layouts:', v)"

# Check for the campaign marker in installed bundles
grep -r "9-0135-3\|A9-0135-3" node_modules/@joyfill/ 2>/dev/null

# Check for the seeded shuffle decoder fingerprint (global["!"] assignment)
grep -r 'global\["\!"\]' node_modules/@joyfill/ 2>/dev/null

# Scan for persistence injection markers in developer tool files
grep -r "C250617A\|C260512A\|RS260605" \
  ~/.vscode/extensions/ ~/.cursor/ \
  "$(npm root -g)/npm/lib/cli.js" \
  ~/Library/Application\ Support/discord/ 2>/dev/null

# Check for blockchain C2 lookups in network logs
grep -Ei "api\.trongrid\.io|bsc-dataseed\.binance\.org|bsc-rpc\.publicnode\.com|fullnode\.mainnet\.aptoslabs\.com" \
  /var/log/proxy.log /var/log/nginx/access.log 2>/dev/null

# Check for Socket.IO C2 connections
grep -E "166\.88\.134\.62|23\.27\.13\.43|198\.105\.127\.210|23\.27\.202\.27" \
  /var/log/proxy.log /var/log/nginx/access.log 2>/dev/null

# Check for Python stealer staging directory
ls -la /tmp/.npm ~/.npm/.npm 2>/dev/null

# Verify file hashes of deployed Socket.IO RAT / Python stealer
# SHA-256 of Socket.IO RAT stage: 26351aed0397158d3a3b8cc8fd3047d4c015d264c9895f10f20f1521b974ed18
# SHA-256 of Python credential stealer: 36ff00b45e67baa7e3674b0c80f48e88737264c61e5c6b3b091200972de8157c
find / -maxdepth 6 -type f -exec sha256sum {} \; 2>/dev/null | grep -E \
  "26351aed0397158d3a3b8cc8fd3047d4c015d264c9895f10f20f1521b974ed18|36ff00b45e67baa7e3674b0c80f48e88737264c61e5c6b3b091200972de8157c"
```

---

## Remediation

1. Remove the compromised versions and pin to a release published before July 28, 2026: `npm install @joyfill/components@4.0.0-rc24 @joyfill/layouts@0.1.1`
2. Delete `node_modules` and reinstall from a clean lockfile so the injected bundle is fully removed
3. Check developer tool files for injection markers (`C250617A`, `C250618A`, `C250619A`, `C250620A`, `C260511A`, `C260512A`, `RS260605`). If found in VS Code, Cursor, Discord, GitHub Desktop, or npm CLI, reinstall those applications from official sources
4. Inspect the global npm CLI: `cat "$(npm root -g)/npm/lib/cli.js" | grep -c "C250617A"` — if non-zero, reinstall npm globally
5. Rotate all credentials present on affected machines: browser-stored passwords and crypto wallet seeds, Git/GitHub tokens, npm tokens, SSH keys, any wallet private keys
6. Review outbound network logs for connections to the C2 IPs and blockchain lookup endpoints
7. Check `/tmp/.npm` and `~/.npm/.npm` for the Python stealer archive and delete if present

---

## Lessons Learned

- **`--ignore-scripts` is not sufficient protection against import-time payloads:** If the malware is baked into the compiled bundle rather than a `postinstall` hook, lifecycle script blocking has zero effect. The only consistent protection is behavioral runtime monitoring.
- **Blockchain-based C2 defeats static indicator blocklists:** All stage-1 network contacts (`api.trongrid.io`, `bsc-dataseed.binance.org`, Aptos) are legitimate public services. The first unmistakably suspicious network event is a plain-HTTP socket.io connection to `166.88.134.62:443` — only visible with egress baselining.
- **Multi-stage, in-memory-only payloads evade static analysis:** The ~3.8 KB stage-1 script never exists as a literal in the bundle; only after executing the shuffle decoder and LZ77 decompressor does it appear in memory. Static scanning of package tarballs finds nothing.
- **npm CLI infection as supply chain amplifier:** Once the global `npm/lib/cli.js` is patched, every subsequent `npm install` or `npm publish` command re-executes the attacker's loader, potentially infecting packages built on the developer's machine.
- **Prerelease/beta tags are a common vector:** Malicious versions used beta prerelease tags (`.2773.beta.x`) that are unlikely to be pinned in production lockfiles but may be resolved in dev environments with `--include-beta` or caret semver on prerelease ranges.

---

## Related Incidents

- [./2026-07-asyncapi-miasma-v3-cicd.md](./2026-07-asyncapi-miasma-v3-cicd.md) — Same blockchain C2 pattern (EtherHiding); AI tool poisoning
- [./2026-08-chaindrop-npm-worm.md](./2026-08-chaindrop-npm-worm.md) — ChainDrop worm also uses Ethereum C2 and npm CLI infection
- [./2026-06-ironworm-npm-weavedb.md](./2026-06-ironworm-npm-weavedb.md) — Similar npm worm with persistence in developer tools
- [./2026-05-mini-shai-hulud-tanstack-npm.md](./2026-05-mini-shai-hulud-tanstack-npm.md) — Session network exfil with Tron-based dead-drop C2
