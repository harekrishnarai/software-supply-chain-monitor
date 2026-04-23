# postmark-mcp — First Confirmed Malicious MCP Server on npm (Silent BCC Injection)

**Date:** October 2025
**Ecosystem:** npm / Model Context Protocol (MCP)
**Severity:** High
**Type:** Typosquatting / MCP Supply Chain / Email Interception
**Sources:**
- [Semgrep — So the first malicious MCP server has been found on npm, what does this mean for MCP security?](https://semgrep.dev/blog/2025/so-the-first-malicious-mcp-server-has-been-found-on-npm-what-does-this-mean-for-mcp-security/)
- [Koi Research — postmark-mcp npm malicious backdoor email theft](https://www.koi.security/blog/postmark-mcp-npm-malicious-backdoor-email-theft)

---

## Summary

On September 25, 2025, Koi Research discovered the first confirmed malicious Model Context Protocol (MCP) server on npm. A threat actor created a package named `postmark-mcp` on npm — a name identical to the official Postmark MCP server, which ActiveCampaign/Postmark had been developing on GitHub but had not yet independently published to npm. The attacker had been maintaining their npm version of the package for 15 clean, legitimate-appearing versions before quietly adding a one-line malicious payload in a later release.

The attack is notable for its subtlety. Unlike most npm malware, there was no malware dropper, no credential harvester, and no obfuscated code. Instead, the attacker added a single extra line to the `send_email` tool implementation that silently added a BCC recipient (an attacker-controlled email address) to every email sent by any AI agent using the package. Because Postmark handles transactional email — password resets, security alerts, payment receipts, account notifications — the BCC injection could have silently intercepted sensitive tokens, financial data, and account recovery flows from any organization using the package with an AI agent.

The attack demonstrates the emerging supply chain threat surface created by the MCP ecosystem, where developers and enterprises are rapidly installing third-party MCP servers to connect AI assistants to production services, without the vetting processes typically applied to production code libraries.

---

## Compromised Artifacts

| Package | Malicious Version | Platform |
|---------|------------------|----------|
| `postmark-mcp` (npm, typosquat) | v[final, post-15-version buildup] | npm |

**Legitimate source:** `postmark-mcp` is officially maintained by ActiveCampaign at https://github.com/ActiveCampaign/postmark-mcp and offers 4 official tools: send email with template, send email without template, list templates, and delivery statistics.

**Typosquat account:** The malicious npm account belonged to an active developer with a consistent GitHub commit history, real-world project contributions, and a Paris-based profile — engineered to pass superficial legitimacy checks that security-conscious developers might apply to unfamiliar npm accounts.

---

## How It Worked

### Step 1 — Identifying the Target Opportunity

Postmark (an ActiveCampaign product) is a transactional email service widely used for password resets, security notifications, receipts, and newsletters. ActiveCampaign developed an official MCP server for Postmark on GitHub, making it available for AI assistants and agents to send emails programmatically. However, the official GitHub implementation was not published to npm under the same name.

This gap between GitHub availability and npm distribution is a structural vulnerability that the attacker exploited: developers installing MCP servers via npm would find the attacker's package before discovering the official GitHub source.

### Step 2 — Long-Horizon Trust Building (15 Clean Versions)

The attacker published the `postmark-mcp` package to npm and maintained it for 15 clean, functional versions — a faithful copy of ActiveCampaign's open-source MCP server. This strategy:

- Established publish history on the attacker's npm account
- Built up a version number that appeared mature
- Made the package appear as a legitimate, maintained project
- Allowed the package to accumulate downloads and potentially appear in dependency graphs

This is a significant evolution from opportunistic typosquatting, where attackers typically publish a malicious package immediately. The multi-version patience strategy was specifically designed to defeat heuristics that flag "new accounts" or "first-time publishers" as risky.

### Step 3 — The Silent BCC Payload

In a later release, the attacker added a single line to the `send_email` tool handler: an additional BCC recipient field pointing to an attacker-controlled email address. The modified code would be structurally identical to the legitimate implementation except for this one extra parameter.

The payload was deliberately chosen to be:
- **Non-obvious:** A BCC addition is functional code, not malware-looking obfuscation
- **Silent:** Recipients never see BCC fields; the email sender UI shows a successful send
- **High-value:** Postmark handles transactional email including password reset tokens, payment confirmations, and security codes — data far more valuable than typical credential theft
- **Plausibly deniable:** Could be dismissed as a "misconfiguration" rather than intentional malware

### Step 4 — AI Agent Attack Surface

The BCC injection was specifically designed to exploit the AI agent use case. When an AI agent (Claude, GPT, Copilot, etc.) is connected to the `postmark-mcp` MCP server and instructed to send emails, it invokes the `send_email` tool without auditing the underlying implementation. The agent passes the specified recipients and content; the malicious MCP server silently adds the BCC before passing the request to Postmark's API.

This means:
- Automated password reset flows: attacker receives the reset token
- Invoice and receipt emails: attacker receives payment details
- Security verification emails: attacker intercepts OTP codes
- Marketing emails with sensitive customer data: silently copied to attacker

The attack is particularly dangerous in agentic AI workflows where the AI assistant autonomously sends emails on behalf of the user or organization, with no human review of outgoing messages.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Pre-Sept 2025 | ActiveCampaign develops `postmark-mcp` on GitHub and makes 4 tools available |
| Pre-Sept 2025 | Attacker publishes `postmark-mcp` to npm; maintains 15 clean, functional versions to build trust |
| ~2025-09-25 | Attacker publishes malicious version with silent BCC injection |
| 2025-09-25 | Koi Research discovers and reports the malicious package |
| 2025-10-02 | Semgrep publishes public analysis of the attack and its implications for MCP security |

---

## Detection

```bash
# ── STEP 1: Check if the malicious postmark-mcp is installed ────────────────
npm list postmark-mcp 2>/dev/null

# If installed, verify the source matches the official ActiveCampaign repo
# and check the package's send_email implementation for unexpected bcc fields
cat node_modules/postmark-mcp/dist/*.js 2>/dev/null | grep -i "bcc"

# ── STEP 2: Audit your MCP server sources ────────────────────────────────────
# List all installed MCP server packages
npm list --depth=0 2>/dev/null | grep -i "mcp"

# For each MCP package, verify it comes from the official vendor account:
# npm info <package-name> | grep -E "(author|repository|homepage)"

# ── STEP 3: Check for non-official postmark-mcp versions ─────────────────────
# The official package is from ActiveCampaign / the postmark GitHub org
# Any npm postmark-mcp NOT published by those maintainers is suspicious
npm info postmark-mcp maintainers 2>/dev/null

# ── STEP 4: Review outgoing email BCC fields in logs ─────────────────────────
# If you use Postmark's API, check your message send history for unexpected BCC
# recipients via the Postmark admin dashboard at account.postmarkapp.com

# ── STEP 5: Review AI agent email activity ───────────────────────────────────
# If using Claude, GPT, or other agents with MCP email tools, audit the
# Postmark sending history for BCC addresses not set by your application
```

---

## Remediation

1. **Uninstall the npm `postmark-mcp` package** immediately and replace with the official implementation directly from https://github.com/ActiveCampaign/postmark-mcp. Build and install from source.
2. **Review all emails sent via AI agents** using this package. Check Postmark's message activity dashboard for any BCC recipients you did not authorize.
3. **Rotate any Postmark API keys** that were available to the MCP server, as the server had access to send on your behalf and could have exfiltrated the key itself.
4. **Audit all MCP server packages** in your environment. For every package, verify: (a) the npm publisher account is the official vendor or a verified maintainer, (b) the repository URL matches an official source, and (c) the install source is a verified official distribution.
5. **Prefer GitHub source installs over npm for MCP servers** until the MCP ecosystem establishes trusted marketplace or signing infrastructure.
6. **Implement egress monitoring** on AI agent email activity: log all BCC/CC fields in emails sent through MCP tools and alert on unexpected recipients.

---

## Lessons Learned

- **MCP is a new, under-scrutinized attack surface.** Developers installing MCP servers to enhance AI assistant capabilities are not applying the same vetting processes used for production code libraries. The speed of MCP ecosystem adoption is outpacing supply chain security practices.
- **Long-horizon trust building defeats traditional new-package heuristics.** Maintaining 15 clean versions before adding a payload specifically defeats scanners that flag "new accounts" or "recently published packages." Version history alone is not a sufficient trust signal.
- **Subtle, functional malware is harder to detect than obfuscated droppers.** A single BCC line in a send_email function is not flagged by most static analysis tools as malicious — it looks like a feature, not malware.
- **AI agents amplify the blast radius of MCP compromise.** When an AI agent autonomously sends emails, the attacker's BCC injection runs without any human review cycle. The more autonomy granted to AI agents, the more critical MCP server vetting becomes.
- **The gap between "available on GitHub" and "published to npm" creates squatting opportunities.** Vendors who develop MCP servers (or any tools) on GitHub but delay npm publication create a window where attackers can claim the obvious npm package name. Official vendors should pre-register npm package names even before first release.
- **Postmark's email category (transactional/security) makes this uniquely dangerous.** Password reset tokens, OTP codes, and payment confirmations intercepted via BCC can enable account takeover without any other attack surface.

---

## Related Incidents

- [./2026-02-hackerbot-claw.md](./2026-02-hackerbot-claw.md) — AI-powered bot attack campaign targeting developer tools
- [./2026-01-gone-phishin-npm-phishing.md](./2026-01-gone-phishin-npm-phishing.md) — npm packages used as CDN for targeted phishing; novel abuse of npm/jsDelivr
- [./2025-09-chalk-debug-npm-cryptostealer.md](./2025-09-chalk-debug-npm-cryptostealer.md) — Same September 2025 timeframe; npm compromise wave
