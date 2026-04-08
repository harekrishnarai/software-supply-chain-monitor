# The Great npm Heist

**Date:** September 8, 2025
**Ecosystem:** npm
**Severity:** Critical
**Type:** Foundational package compromise / browser-based crypto hijacking / phishing-enabled maintainer account takeover
**Sources:**
- [aikido.dev — npm debug and chalk packages compromised](https://www.aikido.dev/blog/npm-debug-and-chalk-packages-compromised)

---

## Summary

Starting at **September 8, 2025, 13:16 UTC**, Aikido Intel detected a coordinated campaign that compromised **18 foundational npm packages** collectively downloaded **over 2 billion times per week**. The attacker gained access by phishing the primary maintainer (Josh Junon / Qix-) using a spoofed `support@npmjs.help` domain registered just three days earlier.

The injected payload is a **browser-based interceptor** that hooks into the JavaScript runtime to silently hijack cryptocurrency transactions — rewriting wallet addresses, manipulating Ethereum and Solana approvals, and redirecting funds without any visible indication to the user. Because these are low-level utility packages (color libraries, string formatters, debug tools) that sit at the base of nearly every JavaScript project, the blast radius was enormous.

---

## Compromised Packages

All 18 packages had new malicious versions pushed beginning September 8, 2025:

| Package | Malicious Version | Weekly Downloads (at time of attack) |
|---------|------------------|--------------------------------------|
| `debug` | — | 357.6M |
| `ansi-styles` | — | 371.41M |
| `chalk` | — | 299.99M |
| `supports-color` | 10.2.1 | 287.1M |
| `ansi-regex` | — | 243.64M |
| `strip-ansi` | — | 261.17M |
| `wrap-ansi` | — | 197.99M |
| `color-convert` | — | 193.5M |
| `color-name` | — | 191.71M |
| `is-arrayish` | — | 73.8M |
| `slice-ansi` | — | 59.8M |
| `error-ex` | — | 47.17M |
| `color-string` | — | 27.48M |
| `simple-swizzle` | — | 26.26M |
| `has-ansi` | — | 12.1M |
| `supports-hyperlinks` | — | 19.2M |
| `chalk-template` | — | 3.9M |
| `backslash` | — | 0.26M |

An additional package `proto-tinker-wc@0.1.87` was detected at 16:58 UTC, likely compromised by the same attackers.

> All packages shared the same underlying attacker: the compromise was enabled by one phished maintainer who maintained many of these packages.

---

## How It Worked

### Entry: Phishing the Maintainer

The attack began with a targeted phishing email to **Josh Junon (Qix-)**, the npm maintainer responsible for many of these packages. The email came from `support@npmjs[.]help` — a domain that mimics the official `npmjs.com` helpdesk. The phishing domain was registered on **September 5, 2025**, just three days before the attack.

With the stolen credentials, the attacker published malicious patch/minor versions of each package the maintainer controlled.

### Payload: Browser-Based Transaction Interceptor

The malicious `index.js` in each package contained heavily **obfuscated code**. After deobfuscation, the payload is a sophisticated browser-based interceptor that operates at multiple layers:

**Layer 1 — Runtime API Hooking**

The malware injects itself into the browser environment by hooking core web APIs and wallet interfaces:
- `fetch` and `XMLHttpRequest` — intercepts all HTTP traffic
- `window.ethereum` and other crypto wallet APIs — hooks MetaMask and equivalent injected providers
- Common payment and transaction interfaces used by DeFi applications

**Layer 2 — Transaction Rewriting**

Before a transaction is signed, the payload:
- Scans for cryptocurrency wallet address patterns in request payloads and API responses
- Replaces legitimate destination addresses with attacker-controlled addresses using **lookalike string substitution** — making swaps visually indistinguishable at a glance
- Rewrites Ethereum transaction parameters: `to`, `value`, token approval `spender` fields
- Rewrites Solana transaction parameters: recipients and approval targets

**Layer 3 — Stealth Behavior**

- If a hardware wallet or browser wallet extension is detected, the payload **avoids obvious UI-level changes** to reduce the chance of detection
- Silent hooks remain active in the background; the visible interface continues to show the original (correct) address
- Even if the UI appears correct, the underlying signed transaction routes funds to the attacker

**Step-by-Step Attack Flow:**

1. Developer installs any of the 18 compromised packages (typically as a transitive dependency)
2. Malicious `index.js` is included in the JavaScript bundle served to end-users' browsers
3. On page load, the payload hooks `fetch`, `XHR`, and wallet APIs
4. When a user initiates a crypto transaction on any site using the compromised bundle, the payload intercepts it
5. Destination address is silently replaced with a lookalike attacker address
6. The transaction is signed and broadcast — funds go to the attacker

### Why Foundational Packages?

Targeting utilities like `debug`, `chalk`, and `ansi-styles` — which have no direct relationship to cryptocurrency — ensures maximum installation surface. Nearly every Node.js web application uses these packages as transitive dependencies. Any application that happens to handle wallet addresses or web3 interactions would become an unwitting delivery vehicle.

---

## Timeline

| Time (UTC) | Event |
|-----------|-------|
| Sep 5, 2025 | `npmjs.help` phishing domain registered |
| Sep 8, 13:16 UTC | Aikido Intel detects malicious packages being published to npm |
| Sep 8, 15:15 UTC | Maintainer Josh Junon notified via Bluesky; confirms compromise and begins cleanup |
| Sep 8, 16:58 UTC | `proto-tinker-wc@0.1.87` detected — second maintainer targeted by same attackers |
| Sep 8 (ongoing) | Maintainer deletes most compromised packages before losing account access; `simple-swizzle` remains compromised at time of initial disclosure |

---

## Detection

```bash
# Check for malicious versions of the 18 packages
npm ls debug chalk ansi-styles strip-ansi ansi-regex \
    color-convert wrap-ansi slice-ansi supports-color \
    is-arrayish color-name error-ex color-string simple-swizzle \
    has-ansi supports-hyperlinks chalk-template backslash

# Cross-reference versions in your lockfile
grep -E '"(debug|chalk|ansi-styles|strip-ansi|supports-color)":' package-lock.json

# Check for proto-tinker-wc
npm ls proto-tinker-wc
```

If any package was installed during September 8, 2025, treat your environment as potentially compromised.

---

## Remediation

1. **Update all affected packages** to the latest clean versions immediately
2. **Clear npm cache:** `npm cache clean --force`
3. **Audit recent transactions** in any web application that processes crypto wallet addresses — look for unexpected destination addresses in transaction logs
4. **Rotate all credentials** stored in the development environment
5. **Rebuild and redeploy** any web applications that may have served a compromised bundle to end-users' browsers

---

## Indicators of Compromise

| Type | Value |
|------|-------|
| Phishing domain | `npmjs.help` |
| Phishing email | `support@npmjs[.]help` |
| Attacker account | Compromised Qix- (Josh Junon) npm account |

---

## Lessons Learned

- **Transitive dependencies are a massive attack surface.** Most projects don't directly depend on `chalk` or `debug` — they arrive through dozens of layers. Tools like [Socket.dev](https://socket.dev) or Aikido Intel can surface malicious transitive deps before install.
- **Typosquatting of official support domains is highly effective.** `npmjs.help` is visually convincing. Never trust support emails at non-`npmjs.com` domains.
- **Browser-layer interception is the most dangerous payload model.** Unlike a credential dumper that fires once, a browser interceptor silently taxes every transaction made through any application that includes the compromised bundle — across all users of all affected sites.
- **One compromised maintainer can cascade across dozens of packages.** Josh Junon's account had publish rights across many of the most-downloaded packages in the ecosystem. Account-level compromise is a high-leverage attack.
- **Obfuscation delays detection.** The payload was obfuscated enough that deobfuscation was required to understand it — meaning standard static analysis tools may have missed it until behavioral signals emerged.

---

## Related Incidents

- [Shai-Hulud Worm Wave 1 (Sep 2025)](./2025-late-shai-hulud-worm.md) — concurrent campaign in the same threat period
- [XRP Supply Chain Attack (Apr 2025)](./2025-04-xrp-supply-chain.md) — same target (crypto private keys), different mechanism (server-side SDK vs. browser interceptor)
- [GlassWorm / CanisterWorm (Mar 2026)](./2026-03-glassworm-canisterworm.md) — similar browser credential and wallet harvesting objectives
