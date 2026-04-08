# npm 'is' Package Compromise — Phishing-Driven Account Takeover

**Date:** July 2025
**Ecosystem:** npm
**Severity:** High
**Type:** Infostealer / Account takeover / Social engineering
**Sources:**
- [StepSecurity — Another npm Supply Chain Attack: The 'is' Package Compromise](https://www.stepsecurity.io/blog/another-npm-supply-chain-attack-the-is-package-compromise)

---

## Summary

On July 19, 2025, malicious versions of the `is` npm package (versions 3.3.1 and 5.0.0) were published to the registry after attackers compromised an old maintainer's account through an ongoing phishing campaign targeting npm package maintainers. The `is` package is a fundamental type-checking utility with millions of weekly downloads, used as a transitive dependency across a vast swath of the JavaScript ecosystem.

The attack was notable for its multi-stage social engineering: the threat actors first hijacked an old (former) maintainer's account via phishing, then used that account to re-add itself to the `is` package using a convincing cover story about npm removing the account for lacking 2FA. Once restored, the account published malicious versions the following morning. The malicious code went undetected for approximately six hours — longer than typical for high-profile packages — and the attackers managed to publish a second malicious version (5.0.0) using what appeared to be a pre-existing authenticated session.

This incident was part of the same sustained phishing campaign that previously compromised `eslint-config-prettier` and several other widely-used npm packages in mid-2025, demonstrating an attacker methodology of systematically targeting package maintainers rather than any technical vulnerability in the registry.

---

## Compromised Artifacts

| Package | Malicious Versions | Safe Versions |
|---|---|---|
| `is` (npm) | 3.3.1, 5.0.0 | 3.3.0, 3.3.2+ |

Attack window: July 19, 2025 (malicious versions published) → ~6 hours later (detected and remediated)

---

## How It Worked

### Entry Point — Phishing the Original Maintainer

The attack unfolded in three stages:

1. An old (former) maintainer of the `is` package had their npm account compromised through the ongoing phishing campaign targeting npm maintainers.
2. The hijacked account owner contacted the current maintainer team via email, claiming that npm had removed their account because they lacked 2FA. The framing exploited a legitimate, known grievance — npm's notification system for ownership changes is widely criticised as opaque.
3. The current maintainer team re-added the hijacked account to the `is` package in good faith.
4. The next morning, the attackers used the restored access to publish malicious versions 3.3.1 and 5.0.0. Version 5.0.0 appeared to be published using a pre-existing authenticated session before the access was revoked.

### Payload Mechanics

The malicious code embedded in the compromised versions was not detailed in the primary disclosure, but the attack is part of the same campaign responsible for compromising `eslint-config-prettier` and other packages in the same period — a campaign characterised by credential harvesting payloads designed for silent exfiltration.

### Propagation

The `is` package is a foundational type-checking utility with millions of weekly downloads and presence across thousands of projects as a transitive dependency. Its compromise created a large potential blast radius affecting any project that auto-updated during the ~6-hour exposure window.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| Mid-2025 | Ongoing phishing campaign begins targeting npm maintainer accounts (previously compromising `eslint-config-prettier` and others) |
| Pre-July 19 | Old maintainer of `is` package falls victim to phishing; npm account hijacked |
| Pre-July 19 | Hijacked account contacts current maintainer team claiming npm removed it for lacking 2FA |
| Pre-July 19 | Current maintainer team re-adds hijacked account in good faith |
| 2025-07-19 | Malicious versions 3.3.1 and 5.0.0 published to npm |
| 2025-07-19 + ~6h | Attack detected and malicious versions removed; clean version 3.3.2 published |
| 2025-07-22 | StepSecurity publishes incident analysis |

---

## Detection

```bash
# Check installed version of 'is' package
npm ls is

# Check for the specific compromised versions in your dependency tree
npm ls is | grep -E "3\.3\.1|5\.0\.0"

# Check package-lock.json for compromised versions
grep -A2 '"is"' package-lock.json | grep -E '"3\.3\.1|5\.0\.0"'

# Scan all projects for the compromised versions
find . -name "package-lock.json" -exec grep -l '"is"' {} \; | \
  xargs grep -l '"3\.3\.1\|5\.0\.0"'

# Check npm cache for malicious versions
ls ~/.npm/is/ 2>/dev/null

# Check for unexpected network connections (if malicious code ran in CI)
# Look for anomalous outbound connections in Harden-Runner or equivalent CI security tool logs
```

---

## Remediation

1. **Identify exposure**: Run `npm ls is` to check the installed version in all affected projects.

2. **Remove compromised versions**:
   ```bash
   rm -rf node_modules package-lock.json
   npm cache clean --force
   # For yarn users:
   rm -rf node_modules yarn.lock
   ```

3. **Install safe versions**: Reinstall with version `3.3.0` or `3.3.2+`:
   ```bash
   npm install
   ```

4. **Rotate credentials** if a compromised version executed in CI or production:
   - npm tokens and publish credentials
   - Any API keys or secrets accessible in the environment where the package ran
   - Browser credentials if the package ran in a browser context

5. **Review CI logs** for any anomalous network connections during the window July 19, 2025 (the ~6-hour exposure window).

6. **Monitor npm ownership notifications** — use tools like Socket.dev or StepSecurity Artifact Monitor to receive alerts when packages your projects depend on have ownership changes or new unexpected releases.

---

## Lessons Learned

- **Ownership change notifications are a systemic gap in npm.** The attacker exploited the fact that npm does not adequately notify maintainers when package ownership changes. The cover story about "npm removed me for lacking 2FA" was plausible precisely because npm's ownership system lacks transparency.
- **Social engineering requires no technical exploit.** This attack did not rely on any vulnerability in npm's infrastructure — it succeeded entirely through a convincing social engineering scenario exploiting normal maintainer-to-maintainer trust.
- **Old maintainer accounts are a persistent risk.** Former contributors who retain npm publish access, or who can be re-added by current maintainers, represent an attack surface that grows as projects mature and contributor bases change.
- **Six hours is enough exposure.** Projects that pin exact versions or use NPM Cooldown-style policies (blocking dependency updates within 48 hours of release) would have been protected, since most automated updates would have resolved to safe versions before or after the attack window.
- **Part of a sustained campaign.** This attack is one incident in a coordinated, multi-month phishing campaign targeting npm maintainers. Security teams should treat maintainer phishing as an ongoing threat requiring proactive defences, not a one-off event.

---

## Related Incidents

- [./2025-09-great-npm-heist.md](./2025-09-great-npm-heist.md) — September 2025 compromise of chalk, debug, and 18 other foundational npm packages via the same phishing campaign
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — Related npm infostealer campaign targeting crypto and browser credentials
- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — Similar account takeover vector in GitHub Actions ecosystem
