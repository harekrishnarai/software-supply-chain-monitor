# chalk, debug, color npm Packages Compromised — Single Contributor Phish, 2.6B Weekly Downloads at Risk

**Date:** September 2025
**Ecosystem:** npm
**Severity:** Critical
**Type:** Account Takeover / Cryptostealer / Browser Crypto Hijacking
**Sources:**
- [Aikido Security — npm packages debug and chalk are compromised](https://www.aikido.dev/blog/npm-debug-and-chalk-packages-compromised)
- [Semgrep — Security Alert: chalk, debug and color on npm compromised in new supply chain attack](https://semgrep.dev/blog/2025/chalk-debug-and-color-on-npm-compromised-in-new-supply-chain-attack/)
- [Aikido Security — We Got Lucky: The Supply Chain Disaster That Almost Happened](https://www.aikido.dev/blog/we-got-lucky-the-supply-chain-disaster-that-almost-happened)
- [Aikido Security — DuckDB npm packages compromised (follow-up)](https://www.aikido.dev/blog/duckdb-npm-packages-compromised)

---

## Summary

On September 8, 2025, beginning at 13:16 UTC, attackers pushed malicious versions of 18 widely-used npm packages — including `debug` (357M weekly downloads) and `chalk` (299M weekly downloads) — after phishing the credentials of a single contributor who maintained multiple core JavaScript utility packages. Together the 18 packages account for over 2 billion downloads per week, making this one of the most broadly scoped npm supply chain attacks ever executed against foundational developer tooling.

The payload is a browser-focused cryptostealer that injects obfuscated JavaScript into each package's `index.js`. Once executed in a browser context, the malware intercepts Ethereum wallet interactions, rewrites token approval and transfer destinations using a fuzzy address-matching algorithm, and silently redirects cryptocurrency transactions to attacker-controlled wallets across Bitcoin, Ethereum, Solana, TRON, and Bitcoin Cash. The attacker maintained 67 Ethereum, 40 Bitcoin, 40 Solana, 40 TRON, and 40 Bitcoin Cash wallet addresses, rotating them to maximize stealth by selecting addresses visually similar to the victim's real address.

The community reacted quickly: most malicious packages were deprecated and removed from npm within one hour of detection, limiting real-world financial damage. A follow-up compromise of a separate contributor account was detected the next day (September 9), affecting the `duckdb` npm packages, which had a 7–9 hour exposure window before deprecation. The npm registrar's response to Josh Junon (Qix), the maintainer locked out of his account for nearly 12 hours, revealed severe deficiencies in maintainer incident-response tooling.

---

## Compromised Artifacts

### Wave 1 — September 8, 2025 (initial 18 packages)

| Package | Malicious Version | Weekly Downloads |
|---------|------------------|-----------------|
| `ansi-styles` | 6.2.2 | 371M |
| `debug` | 4.4.2 | 357M |
| `strip-ansi` | 7.1.1 | 261M |
| `supports-color` | 10.2.1 | 287M |
| `chalk` | 5.6.1 | 299M |
| `ansi-regex` | 6.2.1 | 243M |
| `wrap-ansi` | 9.0.1 | 197M |
| `color-convert` | 3.1.1 | 193M |
| `color-name` | 2.0.1 | 191M |
| `is-arrayish` | 0.3.3 | 73M |
| `slice-ansi` | 7.1.1 | 59M |
| `error-ex` | 1.3.3 | 47M |
| `simple-swizzle` | 0.2.3 | 26M |
| `color-string` | 2.1.1 | 27M |
| `supports-hyperlinks` | 4.1.1 | 19M |
| `has-ansi` | 6.0.1 | 12M |
| `chalk-template` | 1.1.1 | 3.9M |
| `backslash` | 0.2.1 | 0.26M |

### Wave 2 — September 9, 2025 (second compromised account)

| Package | Malicious Version | Notes |
|---------|------------------|-------|
| `duckdb` | 1.3.3 | 7–9 hour exposure |
| `@duckdb/node-api` | 1.3.3 | DuckDB statement that 1.3.3 is not a real release |
| `@duckdb/node-bindings` | 1.3.3 | |
| `@duckdb/duckdb-wasm` | 1.29.2 | |
| `proto-tinker-wc` | 0.1.87 | |

---

## How It Worked

### Step 1 — Phishing a Shared Contributor

The attacker compromised the npm publishing credentials of Josh Junon (Qix), a contributor who maintained multiple foundational JavaScript utility packages (`debug`, `chalk`, `color`, and their dependency trees). A single successful phishing email unlocked publish access to all packages the account maintained, enabling a cascade compromise of packages that are near-universal dependencies in JavaScript projects.

### Step 2 — Injected Cryptostealer Payload

Each malicious package version had a line of obfuscated JavaScript appended to its `index.js`:

```
const 0x112fa8=0x180f;(function(_0x13c8b9,_0x35f660){...
```

The obfuscated code is a hex-encoded, self-contained cryptostealer. After deobfuscation, the payload:

1. **Polls for Ethereum wallet presence** via `window.ethereum.request({method: "eth_accounts"})` using `checkethereumw()`
2. **Intercepts browser `fetch` and `XMLHttpRequest`** to monitor HTTP traffic for transaction signatures
3. **Intercepts ERC-20 function calls** by monitoring for known 4-byte function selectors:
   - `0x095ea7b3` — `approve()`
   - `0xa9059cbb` — `transfer()`
   - `0x23b872dd` — `transferFrom()`
   - `0xd505accf` — ERC-2612 `permit()`
4. **Rewrites wallet addresses** using a fuzzy matching algorithm (`runmask()`) that replaces the victim's address with the attacker's most visually similar wallet address — maximizing the chance the swap goes unnoticed on a transaction confirmation screen
5. **Redirects approved token amounts and transfer destinations** to attacker-controlled addresses

The payload was designed to be invisible in server-side Node.js contexts (it only activates in browser environments with `window.ethereum`), which is why it evaded many automated CI scanners that only check for network exfiltration or process spawning.

### Step 3 — Fuzzy Wallet Address Matching

Rather than using a single static wallet address, the attacker pre-registered 247 total wallet addresses across five blockchain networks (67 Ethereum, 40 Bitcoin Legacy, 40 Bitcoin Segwit/SegWit, 40 TRON, 40 Bitcoin Cash, 40 Solana). For each intercepted transaction, the malware selected the attacker's address that most closely resembled the victim's original destination — reducing the likelihood that a user reviewing the transaction confirmation would notice the substitution.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2025-09-08 13:16 | Aikido Intel first alerts on 18 packages pushed to npm with injected malicious code |
| 2025-09-08 ~14:00 | User `informatic` opens GitHub issue on `debug-js/debug` noting npm version absent from repository |
| 2025-09-08 ~14:30 | GitHub and security community spread awareness; npm begins deprecating malicious versions |
| 2025-09-08 ~15:00 | Most malicious package versions removed from npm; exposure window under ~1 hour for popular packages |
| 2025-09-08 | Josh Junon (Qix) locked out of npm account for ~12 hours; npm support reachable only via unauthenticated public help form |
| 2025-09-09 | Second compromised npm account detected; `duckdb@1.3.3` and related packages pushed with same payload |
| 2025-09-09 ~09:43 | Aikido publishes follow-up; `duckdb` packages deprecated after 7–9 hour exposure window |

---

## Detection

```bash
# ── STEP 1: Check for compromised package versions ──────────────────────────
# npm installed packages
npm list --depth=0 2>/dev/null | grep -E \
  "(debug@4\.4\.2|chalk@5\.6\.1|ansi-styles@6\.2\.2|strip-ansi@7\.1\.1|\
color@5\.0\.1|color-name@2\.0\.1|color-convert@3\.1\.1|color-string@2\.1\.1|\
ansi-regex@6\.2\.1|supports-color@10\.2\.1|wrap-ansi@9\.0\.1|\
has-ansi@6\.0\.1|slice-ansi@7\.1\.1|is-arrayish@0\.3\.3|\
error-ex@1\.3\.3|simple-swizzle@0\.2\.3|chalk-template@1\.1\.1|\
supports-hyperlinks@4\.1\.1|backslash@0\.2\.1)"

# Deep tree scan (catches transitive deps)
npm ls --all 2>/dev/null | grep -E \
  "debug@4\.4\.2|chalk@5\.6\.1|duckdb@1\.3\.3"

# ── STEP 2: Check lock files for malicious versions ──────────────────────────
grep -E '"(debug|chalk|ansi-styles|strip-ansi|color|color-name|color-convert|color-string|ansi-regex|supports-color|wrap-ansi|has-ansi|slice-ansi|is-arrayish|error-ex|simple-swizzle|chalk-template|supports-hyperlinks|backslash|duckdb)"' \
  package-lock.json 2>/dev/null | grep -E \
  "(4\.4\.2|5\.6\.1|6\.2\.2|7\.1\.1|5\.0\.1|2\.0\.1|3\.1\.1|2\.1\.1|6\.2\.1|10\.2\.1|9\.0\.1|6\.0\.1|7\.1\.1|0\.3\.3|1\.3\.3|0\.2\.3|1\.1\.1|4\.1\.1|0\.2\.1)"

# ── STEP 3: Check index.js files for injected payload ────────────────────────
# Look for the characteristic hex obfuscation pattern
find node_modules -name "index.js" -exec grep -l "0x[0-9a-f]\{5,\}" {} \; 2>/dev/null | \
  grep -E "(debug|chalk|color|ansi)" | head -20

# Check for the specific payload signature
grep -r "0x112fa8\|checkethereumw\|runmask\|0x095ea7b3" node_modules/ 2>/dev/null

# ── STEP 4: Run Semgrep open-source detection rule ───────────────────────────
# Requires semgrep installed: pip install semgrep
semgrep --config r/kxUgZJg/semgrep.ssc-mal-deps-mit-2025-09-chalk-debug-color .

# ── STEP 5: DuckDB-specific checks ───────────────────────────────────────────
npm list duckdb @duckdb/node-api @duckdb/node-bindings 2>/dev/null | \
  grep "1\.3\.3\|1\.29\.2"
```

---

## Remediation

1. **Upgrade affected packages immediately** to the next clean version (one minor patch above each malicious version — verify against npm registry that the new version has no `trustedPublisher` anomalies).
2. **Run `npm ci --ignore-scripts`** on all CI/CD pipelines to clear contaminated lockfile resolutions and reinstall from a clean manifest.
3. **Purge npm cache** on affected machines:
   ```bash
   npm cache clean --force
   ```
4. **Audit lock files** for the exact malicious version strings and replace with safe versions in both `package.json` and `package-lock.json`.
5. **If the project runs in a browser context** (or builds for the browser), treat any machine that installed the malicious versions as potentially having served crypto-hijacked pages. Review server access logs and user complaint reports for unexpected transaction failures.
6. **Rotate npm tokens** for any account that published packages during the compromise window to rule out lateral spread.
7. **For DuckDB users**: Ensure you are not on `duckdb@1.3.3`, `@duckdb/node-api@1.3.3`, `@duckdb/node-bindings@1.3.3`, or `@duckdb/duckdb-wasm@1.29.2`. DuckDB officially stated it will never release a 1.3.3 and the next legitimate release will be 1.4.0.
8. **Enable phishing-resistant MFA** (hardware token, not TOTP) on all npm accounts with publish rights to high-download packages.

---

## IOCs

### Attacker-Controlled Wallet Addresses (selected)

**Ethereum (67 total — sample):**
- `0x013285c02ab81246F1D68699613447CE4B2B4ACC`
- `0x02004fE6c250F008981d8Fc8F9C408cEfD679Ec3`
- `0x40C351B989113646bc4e9Dfe66AE66D24fE6Da7B`
- `0x97A00E100BA7bA0a006B2A9A40f6A0d80869Ac9e`
- `0x7a250d5630b4cf539739df2c5dacb4c659f2488d`
- `0xe592427a0aece92de3edee1f18e0157c05861564`

**Bitcoin Legacy (40 total — sample):**
- `1GX1FWYttd65J26JULr9HLr98K7VVUE38w`
- `13694eCkAtBRkip8XdPQ8ga99KEzyRnU6a`
- `1JQ15RHehtdnLAzMcVT9kU8qq868xFEUsS`

**Bitcoin SegWit (40 total — sample):**
- `bc1q28ks0u6fhvv7hktsavnfpmu59anastfj5sq8dw`
- `bc1q9ca9ae2cjd3stmr9lc6y527s0x6vvqys6du00u`

**TRON (40 total — sample):**
- `TAzmtmytEibzixFSfNvqqHEKmMKiz9wUA9`
- `TGExvgwAyaqwcaJmtJzErXqfra66YjLThc`

**Bitcoin Cash (40 total — sample):**
- `bitcoincash:qp79qg7np9mvr4mg78vz8vnx0xn8hlkp7sk0g86064`
- `bitcoincash:qqgjn9yqtud5mle3e7zhmagtcap9jdmcg509q56ynt`

### Payload Signatures
- Obfuscation pattern: `const 0x[0-9a-f]{5}=0x[0-9a-f]{4}` at start of injected line
- Function signatures intercepted: `0x095ea7b3`, `0xa9059cbb`, `0x23b872dd`, `0xd505accf`
- Detection function names (deobfuscated): `checkethereumw`, `runmask`, `newdlocal`

---

## Lessons Learned

- **One phished contributor can compromise billions of weekly downloads.** The entire ecosystem of utility packages downstream from a single npm account is only as secure as that account's MFA. Hardware key enforcement for maintainers of >1M weekly download packages would have stopped this attack at the credential layer.
- **Browser-only payloads evade most server-side CI scanners.** Malware that only activates in `window.ethereum` contexts will pass Node.js-only runtime sandbox checks. Supply chain tools must also inspect for known browser API hooks and wallet function signatures.
- **Fuzzy address matching defeats visual inspection.** The attacker's wallet rotation strategy was specifically designed to defeat the human review step in Web3 transaction flows. Downstream web3 dApps and browser wallets should implement automated address change detection alerts.
- **npm maintainer lockout is an emergency with no fast resolution path.** Josh Junon was locked out for 12 hours with no working automated recovery option. Registries need authenticated emergency escalation channels for maintainers of critical packages.
- **Speed of community response was the primary mitigation.** Malicious versions were live for under an hour for most packages. Real-time public disclosure via GitHub issues and social media drove faster registry action than any automated tool.
- **The duckdb follow-up shows the attacker targeted multiple contributor accounts.** The next-day compromise of a separate account with the same payload suggests systematic targeting of contributors to popular JavaScript packages, not opportunistic credential reuse.

---

## Related Incidents

- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — Earlier npm infostealer campaign via fake React/Solana packages
- [./2025-09-great-npm-heist.md](./2025-09-great-npm-heist.md) — The Great npm Heist; 18 foundational packages browser crypto hijacking
- [./2025-07-eslint-config-prettier-phishing.md](./2025-07-eslint-config-prettier-phishing.md) — Phishing-based account takeover of npm maintainer (JounQin campaign)
- [./2025-07-is-package-compromise.md](./2025-07-is-package-compromise.md) — npm `is` package phishing compromise; same maintainer (Josh Junon / Qix)
