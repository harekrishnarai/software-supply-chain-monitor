# Gone Phishin' — npm Packages Used as CDN for Targeted Spear-Phishing Against Industrial Firms

**Date:** January 2026
**Ecosystem:** npm / jsDelivr
**Severity:** High
**Type:** Spear-phishing / Credential harvester / CDN abuse
**Sources:**
- [Aikido Security — Gone Phishin': npm Packages Serving Custom Credential Harvesting Pages](https://www.aikido.dev/blog/npm-supply-chain-phishing-campaigns)

---

## Summary

On January 20, 2026 at 18:03 UTC, Aikido's malware detection pipeline flagged an npm package called `flockiali`. Over the next two days, the same attacker published four additional packages — `opresc`, `prndn`, `oprnm`, and `operni`. None of these packages contained code meant to run on a developer's machine via a postinstall hook. Instead, each package was a single JavaScript file designed to completely replace any webpage it was loaded on with a phishing kit — delivered via **jsDelivr**, npm's official public CDN.

The attack targeted employees at industrial and energy companies across Europe, the Middle East, and the United States. Each package version was tailored to a specific named individual, delivering a custom phishing page matching their employer's branding and listing files likely to be familiar to that person. This is a novel supply chain technique: rather than attacking developers through malicious packages, the attacker weaponized the npm registry and jsDelivr CDN as free, trusted phishing delivery infrastructure, bypassing the need for dedicated attacker-hosted phishing servers.

---

## Compromised Artifacts

| Package | Ecosystem | Published | Targeting |
|---|---|---|---|
| `flockiali` | npm | 2026-01-20 18:03 UTC | CQFD Composites (aerospace, France) |
| `opresc` | npm | 2026-01-22 | Energy sector targets (Siemens Energy typosquat) |
| `prndn` | npm | 2026-01-22 | Industrial sector targets |
| `oprnm` | npm | 2026-01-22 | Industrial sector targets |
| `operni` | npm | 2026-01-22 | Industrial sector targets |

All packages were removed from npm registry following disclosure.

---

## How It Worked

### Entry Point — npm as Phishing CDN

The attack did not rely on developer machines installing and running the malicious packages. Instead, it exploited the fact that **jsDelivr** (`cdn.jsdelivr.net`) automatically mirrors every npm package and serves its contents over HTTPS from a globally distributed CDN. This allowed the attacker to:

1. Publish a JavaScript file to npm (free, no verification required)
2. Immediately serve it via `https://cdn.jsdelivr.net/npm/<package>@<version>/index.js`
3. Embed this CDN URL as a `<script>` tag in phishing emails or compromised pages

Because jsDelivr is a well-known, trusted CDN, the JavaScript URL would pass through email security filters and corporate firewalls that block unknown domains.

### Payload Mechanics

Each package contained a single JavaScript file. When loaded by a browser, it:

1. **Completely replaced the current webpage** with a tailored phishing kit using `document.write()` or DOM manipulation
2. **Rendered per-victim branding**: The `v1.2.5` variant targeting CQFD Composites used a "MicroSecure Pro" product brand with aerospace CAD file lures
3. **Anti-analysis measures**:
   - Checked for WebDriver indicators (`navigator.webdriver`) to detect automated analysis environments
   - Filtered bot-like user agents
   - Honeypot form fields to detect automated form submission
   - **Mouse trajectory analysis**: Kept the download/submit button disabled until detecting legitimate human mouse movement or touch input
4. **Credential exfiltration**: Harvested entered credentials and POSTed them to attacker-controlled endpoints

### C2 Infrastructure

| Endpoint | Target | Notes |
|---|---|---|
| `oprsys.deno[.]dev` | CQFD Composites variant | Deno Deploy (serverless) |
| Siemens Energy typosquat domains | Energy sector variants | Multiple typosquat domains mimicking siemensenergy.com |

Using serverless compute platforms (Deno Deploy) and typosquat domains as credential collection endpoints is consistent with an attacker deliberately avoiding dedicated infrastructure that could be traced.

### Per-Victim Targeting

Each package version was custom-built for a specific named individual. The phishing pages included:
- Job-relevant file names (RFQs, project specs, CAD files) matching the target's known work
- Company branding matching the target's employer
- Language localized to the target's region

This level of precision (one npm package version per target) indicates a targeted, resource-intensive spear-phishing operation rather than opportunistic mass phishing.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2026-01-20 18:03 UTC | `flockiali` published to npm; Aikido detection fires |
| 2026-01-20 18:03–18:29 UTC | Four versions of `flockiali` published within 26 minutes (rapid iteration) |
| 2026-01-22 | `opresc`, `prndn`, `oprnm`, `operni` published (additional targets) |
| 2026-01-23 | Aikido publishes full disclosure; packages removed from npm |

---

## Detection

```bash
# Check if any of the malicious packages are installed as dependencies
npm ls flockiali opresc prndn oprnm operni 2>/dev/null
grep -E '"(flockiali|opresc|prndn|oprnm|operni)"' package-lock.json 2>/dev/null

# Check for jsDelivr CDN calls to these packages in network logs
grep -i "jsdelivr.*flockiali\|jsdelivr.*opresc\|jsdelivr.*prndn\|jsdelivr.*oprnm\|jsdelivr.*operni" \
  /var/log/nginx/access.log /var/log/apache2/access.log 2>/dev/null

# Check for credential exfiltration endpoints in network logs
grep -i "oprsys\.deno\.dev" /var/log/syslog 2>/dev/null
# macOS:
log show --last 7d | grep -i "oprsys.deno.dev" 2>/dev/null

# Check browser history / network captures for jsDelivr requests to the malicious package URLs
# Pattern: https://cdn.jsdelivr.net/npm/(flockiali|opresc|prndn|oprnm|operni)@*/
# This can be searched in browser developer tools network logs or proxy logs

# Email gateway: check for links matching the pattern
# https://cdn.jsdelivr.net/npm/(flockiali|opresc|prndn|oprnm|operni)
```

---

## Remediation

1. **Remove any accidental dependency**: Run `npm uninstall flockiali opresc prndn oprnm operni` to ensure none of the packages are in your dependency tree.

2. **Block jsDelivr CDN requests to unknown packages** at the corporate proxy or email gateway level. Consider allowlisting known legitimate packages served via jsDelivr rather than allowing all `cdn.jsdelivr.net` traffic.

3. **If a user was phished** (entered credentials on a spoofed page):
   - Immediately revoke and rotate the compromised credentials
   - Notify the relevant service (corporate VPN, email, SaaS tool) of the potential account compromise
   - Review audit logs for the compromised account for any unauthorized access

4. **Report phishing infrastructure**: Submit the malicious jsDelivr URLs and the credential collection domains to jsDelivr's abuse team, Deno Deploy abuse, and any relevant domain registrars.

5. **Employee awareness**: Remind staff that legitimate corporate tools and vendor portals do NOT load from `cdn.jsdelivr.net`. Any page that unexpectedly transforms its appearance after loading an npm CDN script should be treated as phishing.

---

## Lessons Learned

- **Trusted CDNs are weaponizable phishing infrastructure**: jsDelivr automatically mirrors all npm packages. Publishing a 1KB phishing kit to npm instantly provides a globally distributed, HTTPS-enabled delivery URL at no cost.
- **Per-victim package versioning enables highly targeted spear-phishing**: Publishing a new npm version for each target allows the attacker to maintain separate credential collection and make forensic attribution harder.
- **Anti-analysis in JavaScript phishing kits is maturing**: Mouse trajectory analysis, WebDriver detection, honeypot fields, and bot filtering mean that automated scanners increasingly fail to trigger the phishing payload, letting malicious packages pass registry checks.
- **npm is not just a developer attack surface**: This attack used npm as a logistics platform rather than targeting developers' machines. Defenders should consider npm-as-CDN abuse as a distinct threat vector separate from traditional malicious postinstall hooks.
- **Serverless exfiltration (Deno Deploy, Cloudflare Workers) is increasingly common**: Free serverless platforms absorb the infrastructure cost for attackers and blend credential collection traffic in with legitimate platform usage.

---

## Related Incidents

- [./2026-02-npm-gambling-backdoor.md](./2026-02-npm-gambling-backdoor.md) — Targeted npm supply chain attack using env-gated activation and a remote risk code panel
- [./2026-01-gwagon-npm-infostealer.md](./2026-01-gwagon-npm-infostealer.md) — Concurrent npm infostealer campaign (same week, January 2026)
- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — GitHub Actions supply chain attack abusing trusted CI/CD infrastructure
