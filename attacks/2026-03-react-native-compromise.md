# React Native Phone Number Packages — Account Takeover & Solana C2 Infostealer

**Date:** March 16–18, 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Account takeover / Multi-wave infostealer / Solana blockchain C2
**Sources:**
- [StepSecurity — Malicious npm Releases Found in Popular React Native Packages](https://www.stepsecurity.io/blog/malicious-npm-releases-found-in-popular-react-native-packages---130k-monthly-downloads-compromised)
- [Aikido — Glassworm Strikes Popular React Native Phone Number Packages](https://www.aikido.dev/blog/glassworm-strikes-react-packages-phone-numbers)

---

## Summary

An attacker compromised the npm accounts behind two popular React Native packages — `react-native-international-phone-number` (100K+ monthly downloads) and `react-native-country-select` (30K+ monthly downloads) — and published malicious versions across three increasingly sophisticated waves over three days. The attack evolved from a direct preinstall hook to a three-layer dependency chain using scoped shell packages, demonstrating real-time evasion adaptation.

The payload uses the Solana blockchain as a censorship-resistant C2 dead-drop, with Russian-timezone geofencing to avoid executing on Russian-locale machines. The attack is attributed to the GlassWorm threat actor based on shared Solana C2 infrastructure, identical geofencing logic, and consistent obfuscation techniques across the campaign.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Wave |
|---------|----------------------|------|
| `react-native-international-phone-number` | 0.11.8 | Wave 1 |
| `react-native-country-select` | 0.4.1 | Wave 1 |
| `react-native-international-phone-number` | 0.12.1 | Wave 2 (infrastructure staging) |
| `react-native-country-select` | 0.4.1 (re-published) | Wave 2 |
| `@usebioerhold8733/s-format` | 2.0.1 → 2.0.12+ | Waves 2–3 (payload carrier) |
| `@agnoliaarisian7180/string-argv` | 0.3.0 | Wave 2 (relay package) |
| `react-native-international-phone-number` | 0.12.2 | Wave 3 (armed payload) |

---

## How It Worked

### Wave 1 — Direct Preinstall Hook (March 16)

The attacker published `react-native-international-phone-number@0.11.8` and `react-native-country-select@0.4.1` with a direct `preinstall: "node install.js"` script in `package.json`. The obfuscated `install.js` contained the full GlassWorm payload. Both packages were published without corresponding GitHub Actions workflow runs, confirming the attacker used stolen npm credentials rather than the legitimate release process. The `gitHead` in version 0.11.8 matched v0.11.7, proving no new source commit was built.

StepSecurity detected both packages within 5 minutes. The maintainer confirmed the compromise and deprecated the versions within ~2 hours.

### Wave 2 — Infrastructure Staging (March 17)

Despite the maintainer's cooperation, the attacker returned the next morning with a more evasive technique — a three-layer dependency chain:

1. **`@usebioerhold8733/s-format@2.0.1`** — Scoped package with a `postinstall` hook; initially a dry run with no active payload (`init.js` absent)
2. **`@agnoliaarisian7180/string-argv@0.3.0`** — Hollow relay package depending on `s-format@2.0.1`; maintained by Proton Mail address
3. **`react-native-international-phone-number@0.12.1`** — Legitimate-looking version depending on `@agnoliaarisian7180/string-argv@0.3.0`

The payload was not yet active — the infrastructure was fully in place, waiting to be turned on. By Wave 3, the attacker had changed the npm maintainer email to `voughoeveryc05de@proton.me`, confirming ongoing account control.

### Wave 3 — Payload Activation (March 18)

The `@usebioerhold8733/s-format` package was updated rapidly (versions 2.0.2 through 2.0.12+), with each iteration refining the `child.js` loader and evasion techniques. The `postinstall` hook now spawned a detached child process (`child.js`) that performed the actual malicious work, making the installation appear to complete normally.

### Payload: Solana Blockchain C2

The malware uses the Solana blockchain as a dead-drop C2 resolver, polling a hardcoded wallet address:

**Wallet:** `6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ`

The loader calls `getSignaturesForAddress` to retrieve the latest transaction memo, which contains a base64-encoded URL pointing to the current payload host. This allows the attacker to rotate C2 servers without modifying any published packages.

### Geofencing — Russian-Locale Avoidance

All three waves include checks that exit silently if a Russian locale is detected:
- Language environment (`LANG`, `LANGUAGE`, `LC_ALL`)
- Timezone strings (`Europe/Moscow`, `Asia/Krasnoyarsk`, `Asia/Vladivostok`, etc.)
- UTC offset (exits if between +2 and +12)

This is a classic nation-state avoidance pattern used by Russian-origin malware.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 16, 2026 ~morning | Wave 1: `react-native-international-phone-number@0.11.8` published with direct preinstall hook |
| Mar 16, 2026 ~morning | Wave 1: `react-native-country-select@0.4.1` published with same payload |
| Mar 16, 2026 | StepSecurity AI flags both packages within 5 minutes |
| Mar 16, 2026 | StepSecurity files issues #165 and #11; maintainer confirms compromise |
| Mar 16, 2026 ~2h later | Malicious versions deprecated by maintainer |
| Mar 17, 2026 8:23 AM | Wave 2: `@usebioerhold8733/s-format@2.0.1` published (dry run) |
| Mar 17, 2026 8:44 AM | Wave 2: `@agnoliaarisian7180/string-argv@0.3.0` published (relay) |
| Mar 17, 2026 9:36 PM | Wave 2: `react-native-international-phone-number@0.12.1` published |
| Mar 17, 2026 9:39 PM | Wave 2: `react-native-country-select@0.4.1` re-published |
| Mar 18, 2026 | Wave 3: `@usebioerhold8733/s-format` rapidly iterated (2.0.2–2.0.12+) |
| Mar 18, 2026 | Wave 3: Payload activated; `child.js` loader operational |
| Mar 18, 2026 | npm maintainer email changed to attacker-controlled Proton Mail |
| Mar 18, 2026 | StepSecurity reports all packages to npm for removal |

---

## Detection

```bash
# Check if compromised packages are installed
npm ls react-native-international-phone-number 2>/dev/null
npm ls react-native-country-select 2>/dev/null

# Check for malicious scoped dependencies
npm ls @usebioerhold8733/s-format 2>/dev/null
npm ls @agnoliaarisian7180/string-argv 2>/dev/null

# Check installed versions against known-malicious
npm view react-native-international-phone-number version
# Safe versions: <= 0.11.7 (before compromise)

npm view react-native-country-select version
# Safe versions: <= 0.4.0 (before compromise)

# Search for the Solana wallet address in node_modules
grep -r "6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ" node_modules/ 2>/dev/null

# Search for the attacker's Proton Mail addresses
grep -r "voughoeveryc05de@proton.me\|AgnoliaArisian7180@proton.me" node_modules/ 2>/dev/null

# Check for child.js loader pattern
find node_modules/ -name "child.js" -exec grep -l "solana\|getSignaturesForAddress" {} \;

# Check network logs for Solana RPC calls from non-crypto applications
# Look for: api.mainnet-beta.solana.com connections during npm install
```

---

## Remediation

1. **Remove compromised packages immediately:**
   ```bash
   npm uninstall react-native-international-phone-number
   npm uninstall react-native-country-select
   ```
2. **Pin to safe versions** if these packages are required (check GitHub releases for legitimate versions)
3. **Audit `package-lock.json`** for any references to `@usebioerhold8733` or `@agnoliaarisian7180` scoped packages
4. **Rotate all credentials** present on any machine where compromised versions were installed
5. **Use `--ignore-scripts`** when installing packages from untrusted sources: `npm install --ignore-scripts`
6. **Enable npm 2FA** on all maintainer accounts and review active tokens

---

## Lessons Learned

- Account takeover of popular packages with 130K+ monthly downloads provides massive blast radius
- Three-layer dependency chains (compromised package → relay → payload carrier) evade simple package audits
- Attackers iterate in real-time, evolving evasion techniques within hours of detection
- Solana blockchain C2 is being standardized as a technique by the GlassWorm threat actor across campaigns
- Russian-locale geofencing is a consistent fingerprint linking campaigns to the same actor
- Even after a maintainer cooperates to deprecate versions, attackers with persistent account access can continue publishing

---

## Related Incidents

- [GlassWorm Chrome Extension RAT](./2026-03-glassworm-chrome-rat.md) — Full GlassWorm payload analysis
- [GlassWorm Returns — Unicode Mass Campaign](./2026-03-glassworm-returns.md)
- [GlassWorm / CanisterWorm](./2026-03-glassworm-canisterworm.md)
- [ForceMemo](./2026-03-forcememo.md) — Same Solana C2 infrastructure
