# dYdX npm & PyPI Compromise — Wallet Stealer + RAT via Credentials Theft

**Date:** January 2026
**Ecosystem:** npm / PyPI
**Severity:** Critical
**Type:** Infostealer / RAT / Compromised maintainer credentials
**Sources:**
- [Socket — Malicious dYdX Packages Published to npm and PyPI](https://socket.dev/blog/malicious-dydx-packages-published-to-npm-and-pypi)
- [The Hacker News — Compromised dYdX npm and PyPI Packages Deliver Wallet Stealers and RAT Malware](https://thehackernews.com/2026/02/compromised-dydx-npm-and-pypi-packages.html)

---

## Summary

On January 27, 2026, Socket's automated AI scanner detected malicious behavior in the official `@dydxprotocol/v4-client-js` (npm) and `dydx-v4-client` (PyPI) packages — the primary developer SDKs for interacting with the dYdX v4 decentralised exchange protocol. The attacker had obtained legitimate publishing credentials and deployed trojanized versions across both ecosystems simultaneously, embedding malware in core registry files used during normal package operations.

The npm payload focused on **cryptocurrency wallet seed phrase theft** and device fingerprinting. The PyPI payload added a **fully functional Remote Access Trojan (RAT)** that beaconed to an attacker-controlled domain every 10 seconds for command execution. dYdX acknowledged the breach on January 28, 2026, and urged all users who had installed the compromised versions to immediately isolate affected machines, move funds from a clean system, and rotate all API credentials.

This was the second known supply chain attack against the dYdX ecosystem — a similar incident occurred in September 2022 via a compromised npm staff account. The coordinated cross-ecosystem deployment (npm and PyPI simultaneously) suggests the attacker had direct access to publishing credentials rather than exploiting a technical registry vulnerability.

---

## Compromised Artifacts

| Package | Ecosystem | Malicious Versions | Safe Versions |
|---|---|---|---|
| `@dydxprotocol/v4-client-js` | npm | 3.4.1, 1.22.1, 1.15.2, 1.0.31 | 1.22.2+, or pin to pre-attack versions |
| `dydx-v4-client` | PyPI | 1.1.5post1 | 1.1.5 (clean), 1.1.6+ |

**Exposure window:** January 9, 2026 (C2 infrastructure registered) → January 27–28, 2026 (detected and remediated)

The 128 phantom package versions collectively accumulated 121,539 downloads between July 2025 and January 2026 (averaging ~3,900 per week, peak ~4,236 downloads in the final month).

---

## How It Worked

### Entry Point — Compromised Publishing Credentials

The attacker obtained legitimate dYdX developer publishing credentials — granting direct access to publish new versions to both the npm registry and PyPI. No vulnerability in registry infrastructure was exploited. Multiple malicious versions were published simultaneously to both ecosystems in a coordinated operation. Credentials were likely obtained via phishing or credential stuffing of developer accounts.

### Payload Mechanics — npm (JavaScript)

Malicious code was inserted into the core `registry.ts` / `registry.js` files, which execute during **normal package usage** — no unusual import or explicit call required. The tampered `createRegistry()` function was the primary injection point.

The npm payload performs two operations on each invocation:
1. **Wallet seed phrase exfiltration** — intercepts seed phrases processed by the client library and forwards them to the C2
2. **Device fingerprinting** — collects MAC address, hostname, OS version, and machine ID, hashes the combination into a SHA-256 device fingerprint for victim tracking across sessions

All stolen data is sent via HTTPS POST to:
```
https://dydx[.]priceoracle[.]site/v4/price
```

### Payload Mechanics — PyPI (Python)

The PyPI package employed a bootstrap pattern for stealth. A `_bootstrap.py` module auto-executes an obfuscated payload from `config.py` at import time. The Python payload includes:

1. **Wallet stealer** — a `list_prices()` function (mirroring the npm vector) intercepts credentials and posts them to the same `/v4/price` C2 endpoint
2. **Remote Access Trojan (RAT)** — runs as soon as the package is imported; a persistent beacon loop contacts:
   ```
   https://dydx[.]priceoracle[.]site/py
   ```
   Every **10 seconds**, using a hardcoded auth token: `490CD9DAD3FAE1F59521C27A96B32F5D677DD41BF1F706A0BF85E69CA6EBFE75`
3. **100-iteration obfuscation loop** in the PyPI payload (vs. simpler npm version) suggests direct access to the publishing pipeline and deliberate effort to harden the PyPI variant against analysis

### C2 Infrastructure

| Domain | Endpoint | Purpose |
|---|---|---|
| `dydx[.]priceoracle[.]site` | `/v4/price` | Credential + wallet seed phrase exfiltration (npm & PyPI) |
| `dydx[.]priceoracle[.]site` | `/py` | RAT C2 for Python victims (10-second beacon) |

The domain `dydx[.]priceoracle[.]site` was registered on **January 9, 2026** — approximately 18 days before the attack was detected — suggesting a planned, premeditated operation with infrastructure staged in advance. Following abuse reports, the domain was placed on "server transfer prohibited" and "client hold" status, indicating likely registry-level seizure.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2022-09 | Prior dYdX supply chain attack: npm staff account hijacked to publish malicious versions |
| 2026-01-09 | C2 domain `dydx[.]priceoracle[.]site` registered |
| 2026-01-27 | Socket's AI scanner detects malicious behavior in dYdX packages; responsible disclosure initiated |
| 2026-01-28 | dYdX acknowledges incident publicly on X; malicious package versions removed |
| 2026-01-28 | dYdX advises: isolate affected machines, move funds from clean system, rotate all API keys |
| 2026-02 | Aikido Security and The Hacker News publish full technical analyses |

---

## Detection

```bash
# Check installed npm package version
npm ls @dydxprotocol/v4-client-js
# Malicious versions: 3.4.1, 1.22.1, 1.15.2, 1.0.31

# Check installed PyPI package version
pip show dydx-v4-client | grep Version
# Malicious version: 1.1.5post1

# Search dependency trees for compromised npm versions
npm ls @dydxprotocol/v4-client-js 2>/dev/null | grep -E "3\.4\.1|1\.22\.1|1\.15\.2|1\.0\.31"
grep '"@dydxprotocol/v4-client-js"' package-lock.json | grep -E '"3\.4\.1|1\.22\.1|1\.15\.2|1\.0\.31"'

# Search for compromised PyPI version in requirements files and lockfiles
grep -r "dydx-v4-client" . | grep -i "1\.1\.5post1"
pip list | grep dydx

# Check for C2 domain in network logs / DNS cache
grep -E "priceoracle\.site" /var/log/syslog /var/log/dns*.log 2>/dev/null
# macOS: check system.log
grep "priceoracle.site" /var/log/system.log 2>/dev/null
# Windows: DNS cache
ipconfig /displaydns | findstr "priceoracle"

# Check for RAT beacon (Python processes making periodic outbound connections)
# Look for 10-second interval HTTPS to priceoracle.site from python processes
netstat -an | grep ESTABLISHED | grep python
lsof -i | grep python | grep ESTABLISHED  # macOS/Linux

# Inspect installed package files for the malicious bootstrap (PyPI)
python3 -c "import importlib, dydx_v4_client; print(dydx_v4_client.__file__)"
# Then inspect: cat <path>/_bootstrap.py and <path>/config.py

# Inspect registry.js for npm (look for exfiltration code in createRegistry function)
find node_modules/@dydxprotocol/v4-client-js -name "registry.js" \
  -exec grep -l "priceoracle\|v4/price\|exfil" {} \;
```

---

## Remediation

1. **Identify exposure**: Check all environments (CI/CD, developer laptops, production) for the malicious npm and PyPI versions listed above.

2. **Remove compromised npm package and reinstall**:
   ```bash
   npm uninstall @dydxprotocol/v4-client-js
   npm cache clean --force
   npm install @dydxprotocol/v4-client-js@latest  # or pin to a verified safe version
   ```

3. **Remove compromised PyPI package and reinstall**:
   ```bash
   pip uninstall dydx-v4-client -y
   pip install dydx-v4-client==1.1.5  # or latest clean version
   ```

4. **Block the C2 domain** at DNS and firewall levels: `dydx.priceoracle.site` and wildcard `*.priceoracle.site`.

5. **Assume full wallet compromise** — if any of the malicious npm or PyPI versions were installed and executed:
   - Transfer all cryptocurrency funds to new wallets **from a clean, unaffected machine**
   - Treat all seed phrases and private keys stored or processed by the dYdX SDK as compromised
   - Rotate all dYdX API keys, exchange API credentials, and any other secrets accessible from the affected environment

6. **Check for RAT persistence** (PyPI variant): kill any Python processes beaconing to `priceoracle.site`, check for cron jobs, systemd units, or macOS LaunchAgents that might have been installed by the RAT payload.

7. **Audit developer credentials**: If the attack originated from stolen publishing credentials, treat all npm and PyPI tokens associated with the affected developer accounts as compromised and revoke/reissue them.

---

## Lessons Learned

- **Cross-ecosystem simultaneous deployment** (npm + PyPI from the same credentials) is a force-multiplier for attackers and a signal for defenders — any incident where coordinated malicious packages appear in multiple registries at the same time warrants treating both as a single campaign rather than isolated events.
- **Malicious code in core function files executes automatically** — inserting a payload into `createRegistry()` or `list_prices()` means it runs during normal usage without requiring any special import or explicit call. Defenders should audit function bodies of core SDK files, not just installation hooks (`postinstall`).
- **10-second RAT beacons are highly anomalous** — standard application polling intervals are rarely this frequent. Network monitoring rules should flag high-frequency outbound HTTPS connections from package runtime processes.
- **Pre-staged infrastructure** (domain registered 18 days before the attack) indicates premeditation. Infrastructure age and registration timing relative to attack timing can be a useful IOC in retrospective investigations.
- **Ecosystem history is not protection** — this was dYdX's *second* supply chain attack in four years, demonstrating that prior incidents do not prevent recurrence without systemic credential hardening (hardware keys, publishing MFA, repository secrets monitoring).

---

## Related Incidents

- [./2026-03-litellm-pypi-stealer.md](./2026-03-litellm-pypi-stealer.md) — Same TeamPCP operator compromising PyPI packages with multi-stage credential stealers
- [./2026-03-telnyx-pypi-wav.md](./2026-03-telnyx-pypi-wav.md) — WAV steganography credential stealer also targeting PyPI
- [./2025-04-xrp-supply-chain.md](./2025-04-xrp-supply-chain.md) — Similar crypto SDK compromise (xrpl package) with private key exfiltration
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — npm infostealer campaign targeting Chrome credentials and crypto wallets
