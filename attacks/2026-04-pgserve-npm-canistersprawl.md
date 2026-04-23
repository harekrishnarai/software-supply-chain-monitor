# CanisterSprawl — pgserve npm Compromise: ICP Canister Exfil + Self-Propagating Worm

**Date:** April 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Credential Stealer / Supply chain worm / ICP canister exfiltration
**Sources:**
- [StepSecurity — CanisterSprawl: pgserve Compromised on npm: Malicious Versions Harvest Credentials and Exfiltrate to a Decentralized ICP Canister](https://www.stepsecurity.io/blog/pgserve-compromised-on-npm-malicious-versions-harvest-credentials)
- [Semgrep — Security Advisory: $foo compromised on $packagemanager](https://semgrep.dev/blog/2026/security-advisory-pgserve-xinference-kube-health)

---

## Summary

On April 21, 2026, an attacker published three malicious versions of `pgserve` to npm — versions 1.1.11, 1.1.12, and 1.1.13 — introducing a 1,143-line credential-harvesting and worm-propagation script that runs automatically on `npm install` via a `postinstall` hook. The last legitimate release was v1.1.10, tagged on April 17, 2026. None of the three compromised versions have a corresponding git tag in the upstream repository.

`pgserve` is an embedded PostgreSQL server for development environments: zero config, auto-provisioned databases, designed to be dropped into any Node.js project. Its developer-facing role makes it a high-value target — machines running it typically have npm publish tokens, cloud credentials, and browser-stored passwords that the malware actively targets.

The attack, dubbed **CanisterSprawl** by StepSecurity, is notable for two distinguishing features. First, it uses an **Internet Computer Protocol (ICP) blockchain canister** as the primary exfiltration endpoint — a censorship-resistant compute contract that cannot be taken down by domain seizure or hosting provider requests. Second, the malware is a **self-propagating supply chain worm**: if it finds an npm publish token on the victim's machine, it re-injects itself into every package that token can publish, automatically spreading the compromise further downstream. It also crosses ecosystems, using `.pth` file injection to spread to PyPI packages if a PyPI token is found.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Safe Version | Trigger |
|---------|---------------------|--------------|---------|
| `pgserve` | 1.1.11, 1.1.12, 1.1.13 | ≤ 1.1.10 | `postinstall` hook → `scripts/check-env.cjs` |

---

## How It Worked

### Entry Point

The `postinstall` hook in `package.json` was changed to:
```json
"postinstall": "node scripts/check-env.cjs || true"
```
The `|| true` silences any errors so the install appears to complete cleanly. Two files were injected into the `scripts/` directory: `check-env.js` (the 1,143-line malware) and `public.pem` (the attacker's RSA-4096 public key for payload encryption).

### 1. Environment Variable Harvesting

The `harvest()` function scans every environment variable against ~40 regex patterns:

```javascript
const sensitivePatterns = [
  /TOKEN/i, /SECRET/i, /KEY/i, /PASSWORD/i, /CREDENTIAL/i,
  /^AWS_/i, /^AZURE_/i, /^GCP_/i, /^GOOGLE_/i,
  /^NPM_/i, /^GITHUB_/i, /^GITLAB_/i, /^DOCKER_/i,
  /^DATABASE/i, /^DB_/i, /^REDIS/i, /^MONGO/i,
  /^OPENAI/i, /^ANTHROPIC/i, /^COHERE/i,
  /^STRIPE/i, /^TWILIO_/i, /^SENDGRID_/i, ...
];
```

In a controlled analysis run on a GitHub Actions runner, the malware harvested 38 matching environment variables.

### 2. Filesystem Secret Collection

The script reads credential files from the developer's home directory, including: npm and git tokens (`~/.npmrc`, `~/.netrc`), SSH keys (all files under `~/.ssh/`), cloud credentials (`~/.aws/credentials`, `~/.azure/accessTokens.json`, GCP application default credentials), crypto wallets (Solana keypair, Ethereum keystore, MetaMask, Phantom, Exodus, Atomic Wallet), and **Chrome browser passwords** — it copies Chrome's SQLite `Login Data` database and decrypts saved passwords using the known Linux Chrome key derivation (`peanuts + saltysalt`, AES-128-CBC).

### 3. Payload Encryption

Before transmission, all stolen data is encrypted with a hybrid scheme that prevents interception-based recovery:
- A random AES-256-CBC session key is generated
- Stolen data is encrypted with the session key
- The session key is then encrypted with the attacker's RSA-4096 public key (bundled in `scripts/public.pem`)

### 4. Dual-Channel Exfiltration

The encrypted payload is sent to two endpoints:

| Channel | Endpoint | Condition |
|---------|----------|-----------|
| ICP canister (primary) | `https://cjn37-uyaaa-aaaac-qgnva-cai.raw.icp0.io/drop` | Always called |
| Webhook (secondary) | `https://telemetry.api-monitor.com/v1/telemetry` | Only if `TEL_SIGN_KEY` env var is set |

The ICP canister is a smart contract on the Internet Computer blockchain — permanent for the lifetime of the blockchain and immune to domain seizure or takedown requests. In the controlled analysis run, the canister confirmed receipt: `{"success":true,"id":10,"size":4468}`.

`telemetry.api-monitor.com` was a fresh domain with zero prior threat intelligence, registered with privacy protection via Bluehost specifically for this campaign.

### 5. Supply Chain Worm Propagation

After exfiltration, the malware searches for npm publish tokens in `process.env.NPM_TOKEN`, `~/.npmrc`, project `.npmrc`, `/etc/npmrc`, and npm config. If a token is found, it enumerates all packages the victim can publish, then for each:
- Bumps the patch version
- Copies `check-env.js` and `public.pem` into a `scripts/` subdirectory
- Adds the `postinstall` hook to `package.json`
- Publishes the infected version with `npm publish`

A single compromised developer account can cascade into dozens of infected downstream packages.

### 6. PyPI Cross-Ecosystem Spreading

If a PyPI token is found, the malware also spreads to Python packages using `.pth` file injection — placing a `.pth` file in Python's `site-packages` directory that executes on every Python interpreter invocation. This is the same cross-ecosystem propagation technique used in the Shai-Hulud worm campaign.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 17, 2026 21:57 | `pgserve@1.1.10` published with git tag `v1.1.10` (last legitimate release) |
| Apr 21, 2026 22:14 | `pgserve@1.1.11` published to npm (no git tag) |
| Apr 21, 2026 22:26 | `pgserve@1.1.12` published to npm (no git tag; identical payload to 1.1.11) |
| Apr 21, 2026 | `pgserve@1.1.13` published to npm (no git tag) |
| Apr 22, 2026 | StepSecurity AI Package Analyst flags all three versions as Critical / Rejected |
| Apr 22, 2026 | Harden-Runner confirms live exfil during controlled analysis; both IOC domains added to global block list |
| Apr 22, 2026 | StepSecurity discloses to maintainer via GitHub issue #25 |
| Apr 22, 2026 | Semgrep publishes advisory covering pgserve alongside xinference and kube-health packages |

---

## Detection

```bash
# Check installed pgserve version
npm ls pgserve

# If version is 1.1.11, 1.1.12, or 1.1.13, the system is compromised

# Check for the injected postinstall script
ls node_modules/pgserve/scripts/check-env.js 2>/dev/null
ls node_modules/pgserve/scripts/public.pem 2>/dev/null

# Check postinstall hook in package.json
cat node_modules/pgserve/package.json | grep -A2 '"postinstall"'
# MALICIOUS: "node scripts/check-env.cjs || true"

# Check for outbound connections to ICP canister or webhook domain
# in network logs:
grep -r "cjn37-uyaaa-aaaac-qgnva-cai\|api-monitor\.com\|telemetry\.api-monitor" /var/log/ 2>/dev/null

# Check npm tokens that may have been harvested
cat ~/.npmrc | grep "_authToken"

# Check for unauthorized recently published npm versions on your packages
npm view <your-package> versions --json 2>/dev/null | tail -5

# Check package.json of your packages for unauthorized postinstall hooks
find . -name "package.json" -not -path "*/node_modules/*" -exec grep -l "check-env" {} \;
```

---

## Remediation

1. **Remove or downgrade the compromised package:**
   ```bash
   npm uninstall pgserve
   # or pin to the safe version:
   npm install pgserve@1.1.10
   ```

2. **Check whether your npm token was exfiltrated.** Assume yes if `NPM_TOKEN`, `~/.npmrc`, or project `.npmrc` was present on the affected machine. Revoke and regenerate your npm token immediately at npmjs.com → Account Settings → Access Tokens.

3. **Audit all packages you can publish** for unauthorized recent patch versions. Check timestamps and compare against your own publish history:
   ```bash
   npm view <your-package> time --json 2>/dev/null
   ```
   If any unexpected versions appear, unpublish them immediately.

4. **Rotate all credentials** accessible on the affected machine: AWS keys, GCP service account keys, Azure credentials, SSH keys, Docker registry tokens, GitHub/GitLab tokens, database passwords, API keys matching `OPENAI_*`, `ANTHROPIC_*`, `STRIPE_*`, etc.

5. **Rotate browser-saved passwords** — the malware decrypts Chrome's saved passwords on Linux. Invalidate sessions and rotate passwords for any accounts stored in the browser.

6. **Check for PyPI token** and audit PyPI packages you publish for unauthorized versions if a PyPI token was present.

7. **Block the IOC domains** at the firewall/DNS level:
   - `cjn37-uyaaa-aaaac-qgnva-cai.raw.icp0.io`
   - `telemetry.api-monitor.com`

---

## Lessons Learned

- ICP blockchain canisters as exfiltration endpoints are operationally immune to takedown — defenders cannot remove the receiving infrastructure the way they can seize a domain or contact a hosting provider
- Self-propagating worms that use npm publish tokens can turn a single compromised developer account into a cascading supply chain compromise affecting dozens of downstream packages, all within a single `npm install`
- The `|| true` postinstall hook silencing pattern means the malware leaves no visible install error — victim machines appear to have installed the package cleanly
- Chrome password decryption on Linux (via known `peanuts + saltysalt` AES-128-CBC derivation) is a mature, well-documented technique now appearing routinely in supply chain malware — browser password managers on developer machines are not safe from postinstall scripts
- The cross-ecosystem PyPI `.pth` injection shows attackers are now thinking about multi-registry propagation as a standard worm capability, not an advanced feature

---

## Related Incidents

- [CanisterWorm — TeamPCP npm Worm & Kubernetes Wiper](./2026-03-canisterworm-npm.md) — Earlier ICP canister C2 abuse and self-propagating npm worm with the same dual-channel exfil model
- [Shai-Hulud Worm Wave 1](./2025-late-shai-hulud-worm.md) — Prior self-propagating npm → PyPI cross-ecosystem worm using the same `.pth` injection technique
- [TeamPCP xinference PyPI Compromise](./2026-04-xinference-pypi-teampcp.md) — Concurrent April 2026 PyPI attack reported the same day
- [GlassWorm / CanisterWorm](./2026-03-glassworm-canisterworm.md) — CanisterWorm origin with ICP C2 infrastructure
