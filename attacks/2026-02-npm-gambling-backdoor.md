# npm Gambling Backdoor — json-bigint Typosquats with Payment & Outcome Manipulation

**Date:** February 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Typosquatting / Backdoor / RAT / Financial fraud
**Sources:**
- [Aikido Security — npm Backdoor Lets Hackers Hijack Gambling Outcomes](https://www.aikido.dev/blog/npm-backdoor-lets-hackers-hijack-gambling-outcomes)

---

## Summary

In February 2026, Aikido Security discovered a set of malicious npm packages typosquatting the popular `json-bigint` library. The packages — `json-bigint-extend`, `jsonfx`, and `jsonfb` — contained a highly targeted backdoor designed for infiltration of online gambling and gaming platform backends. Unlike typical supply chain attacks focused on credential theft, this campaign was engineered specifically to manipulate gambling outcomes and intercept payment flows, representing a novel financial fraud vector via supply chain compromise.

The malware is dormant by default and only activates in environments where a `SERVICE_NAME` environment variable matches specific gambling-related patterns. Once triggered, it injects a rogue Express.js middleware into the victim application's HTTP routing layer, inserting two backdoor capabilities: a payment interception route and a gambling balance manipulation engine (`fixflow`). A remote-controlled risk policy (`riskCode`) is refreshed from the attacker's C2 every 30 seconds, allowing real-time behavioral updates post-deployment. An embedded Chinese-language file manager panel provides the attacker full filesystem access for exfiltration and remote code execution.

---

## Compromised Artifacts

| Package | Ecosystem | Relationship to Legitimate Package |
|---|---|---|
| `json-bigint-extend` | npm | Typosquat of `json-bigint` |
| `jsonfx` | npm | Typosquat of `json-bigint` |
| `jsonfb` | npm | Typosquat of `json-bigint` |

All three packages present as `json-bigint` utility wrappers with identical malicious payloads.

---

## How It Worked

### Entry Point — Typosquatting + Environment Fingerprinting

The packages typosquat the legitimate `json-bigint` npm package. Rather than activating universally, the backdoor performs environment fingerprinting at startup by checking the `SERVICE_NAME` environment variable. Only environments whose service name matches patterns associated with gambling and gaming backends trigger the payload — making the malware completely dormant in unintended environments and drastically reducing detection opportunities during automated sandbox analysis.

### Backdoor 1 — Payment Route Injection

Once activated, the malware injects a rogue Express.js route handler into the running application's HTTP router, targeting the payment endpoint:

```
POST /v1/pay/purchase-goods
```

This intercepts legitimate payment requests, allowing the attacker to redirect, duplicate, or manipulate financial transactions passing through the gambling platform's purchase flow.

### Backdoor 2 — Gambling Outcome Manipulation Engine (`fixflow`)

The `fixflow(backupAmount)` function implements a four-phase balance manipulation engine using two core components:

- **`GameResultAdjuster`** — modifies game outcome calculations in real time, enabling the attacker to control win/loss results for targeted accounts or sessions
- **`getCashFlow()`** — interfaces with the platform's balance management to adjust credit/debit calculations

The `fixflow` engine can operate silently within normal application business logic, making fraudulent adjustments appear as legitimate game results.

### Remote-Controlled Risk Policy (`riskCode`)

A `riskCode(...)` middleware component fetches and executes a remote risk policy from the attacker's C2 infrastructure every **30 seconds**. This allows the operator to:
- Push real-time behavioral updates without re-deploying the package
- Enable or disable specific fraud capabilities on demand
- Adjust targeting parameters (e.g., which user accounts or sessions to target)
- Switch off the payload if detection is suspected

### Embedded Operator Panel (Chinese-language RAT)

The malware includes a full embedded file management panel with a Chinese-language UI (`目录压缩下载服务` — "Directory Compression Download Service"), providing the attacker with:

- **File browser**: Navigate the server filesystem via HTTP
- **`RunFileContent`**: Download arbitrary files from the server
- **`CompressDownload`**: Package and download entire directories as ZIP archives
- **Remote code execution**: Execute arbitrary commands on the compromised server

The Chinese-language UI strongly suggests the operator infrastructure is Chinese-speaking.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| February 2026 | Malicious packages `json-bigint-extend`, `jsonfx`, `jsonfb` published to npm |
| 2026-02-16 | Aikido Security publishes discovery and technical analysis |

---

## Detection

```bash
# Check if any of the malicious packages are installed
npm ls json-bigint-extend jsonfx jsonfb 2>/dev/null

# Search package-lock.json for malicious packages
grep -E '"json-bigint-extend"|"jsonfx"|"jsonfb"' package-lock.json

# Search all node_modules for the packages
find node_modules -maxdepth 1 -type d \( -name "json-bigint-extend" -o -name "jsonfx" -o -name "jsonfb" \)

# Check for injected Express middleware (look for rogue route handler in running process)
# The backdoor targets /v1/pay/purchase-goods — look for unexpected handlers on this route
grep -r "purchase-goods\|fixflow\|GameResultAdjuster\|getCashFlow\|riskCode" node_modules/ 2>/dev/null

# Check for the embedded RAT panel (Chinese UI string is a strong indicator)
grep -r "目录压缩下载服务\|RunFileContent\|CompressDownload" node_modules/ 2>/dev/null

# Check for SERVICE_NAME environment variable usage in node_modules
grep -r "SERVICE_NAME" node_modules/json-bigint-extend node_modules/jsonfx node_modules/jsonfb 2>/dev/null

# Check network logs for periodic outbound connections (30-second riskCode refresh)
# Look for regular short-interval HTTPS polling from node processes
netstat -an | grep ESTABLISHED | grep node
lsof -i | grep node | grep ESTABLISHED  # macOS/Linux

# Check npm audit logs for install events involving these packages
npm audit
```

---

## Remediation

1. **Immediately remove the malicious packages**:
   ```bash
   npm uninstall json-bigint-extend jsonfx jsonfb
   npm cache clean --force
   ```

2. **Audit your `package.json` and `package-lock.json`** for any dependency on these packages and remove all references.

3. **Assume server compromise** if any of these packages were installed and the `SERVICE_NAME` environment variable matched a gambling-related pattern:
   - Treat all secrets on the affected server as compromised (database credentials, API keys, payment processor tokens, session secrets)
   - Rotate all credentials immediately
   - Revoke and reissue all API keys and payment credentials

4. **Audit financial transactions** — if the payment interception route was active, audit all `POST /v1/pay/purchase-goods` transactions for the period the packages were installed for anomalies, duplicates, or redirected payments.

5. **Audit gambling outcome logs** — if `fixflow`/`GameResultAdjuster` was active, audit game results and balance changes for the affected period for evidence of manipulation.

6. **Block C2 traffic**: Review network logs for any periodic outbound HTTP/HTTPS connections from Node.js processes at 30-second intervals and block the destination endpoints at the firewall.

7. **Hunt for filesystem exfiltration**: Check server logs for unexpected file download or directory listing activity consistent with `RunFileContent` / `CompressDownload` RAT operations.

8. **Use exact package names and version pinning**: Verify all `json-bigint` usage refers to the correct package name (`json-bigint`, not any variant). Pin to a known-good version hash in `package-lock.json`.

---

## Lessons Learned

- **Environment-gated backdoors defeat most automated analysis.** By checking `SERVICE_NAME` before activating, the malware remains completely inert in sandbox environments, generic CI runners, and non-gambling backends — allowing it to pass routine security scans undetected.
- **Supply chain attacks can target business logic, not just credentials.** This campaign demonstrates that attackers are moving beyond credential theft toward direct manipulation of application outcomes — a much harder-to-detect and more financially damaging class of attack for gambling operators.
- **30-second remote policy refresh creates a living backdoor.** Traditional IOC-based detection fails against malware that updates its behavior in real time from a remote C2 — defenders need behavioral baselines, not just static signatures.
- **Chinese-language embedded operator panels are an attribution signal.** The embedded `目录压缩下载服务` UI and the functional operator-facing panel suggest a well-resourced, operationally mature threat actor targeting gambling operators specifically.
- **Legitimate-looking package names with minor variations remain effective.** `json-bigint-extend` and `jsonfx`/`jsonfb` are plausible utility package names that could appear in a `package.json` without immediately triggering suspicion.

---

## Related Incidents

- [./2025-12-neoshadow-npm.md](./2025-12-neoshadow-npm.md) — npm typosquatting campaign with multi-stage modular backdoor
- [./2026-01-dydx-npm-pypi.md](./2026-01-dydx-npm-pypi.md) — Crypto SDK compromise with RAT and wallet stealer
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — npm infostealer campaign targeting Chrome credentials and crypto wallets
- [./2025-09-great-npm-heist.md](./2025-09-great-npm-heist.md) — Large-scale npm account takeover of foundational packages
