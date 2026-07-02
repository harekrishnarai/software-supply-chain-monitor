# Leo Platform npm Supply Chain Attack — 20 Packages Backdoored via Miasma Toolkit

**Date:** June 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Maintainer Account Compromise / Credential Stealer / Self-Propagating Worm / Phantom Gyp Install Hook
**Sources:**
- [StepSecurity — Mass npm Supply Chain Attack: 20 Leo Platform Packages Compromised](https://www.stepsecurity.io/blog/mass-npm-supply-chain-attack-20-leo-platform-packages-compromised)

---

## Summary

On June 24, 2026, an attacker published malicious versions of 20 npm packages belonging to the Leo Platform ecosystem in a coordinated burst spanning less than three seconds. All 20 packages carry an identical CI/CD attack toolkit that steals secrets from GitHub Actions runners, cloud credential stores, package registries, and password managers, then exfiltrates them via the victim's own GitHub token. Together these packages receive approximately 13,600 downloads per week.

The Leo Platform attack is structurally identical to the Miasma v2 campaign documented on June 3, 2026: it uses the same "Phantom Gyp" `binding.gyp` install hook, the same three-layer obfuscation chain (ROT-N cipher + AES-128-GCM + obfuscator.io), and the same Bun v1.3.13 download URL. The publish window — all 20 packages within a 3-second window at 2026-06-24T23:04:55Z — confirms a single automated operation against Leo Platform maintainer credentials. The operation is more targeted and quieter than the original Miasma wave: it focuses on a single maintained ecosystem rather than carpet-bombing across maintainer accounts.

This is the same threat actor (Miasma/TeamPCP) operating with improved tooling 21 days after the original Miasma campaign. The "Phantom Gyp" technique uses the node-gyp `<!(...)>` shell expansion inside the `sources` array of `binding.gyp` to execute arbitrary commands at install time without declaring any `scripts` block entry in `package.json`, bypassing lifecycle-script blocking controls introduced in npm v12.

---

## Compromised Artifacts

| Package | Malicious Version |
|---------|------------------|
| `leo-logger` | 1.0.8 |
| `leo-sdk` | 6.0.19 |
| `leo-aws` | 2.0.4 |
| `leo-config` | 1.1.1 |
| `leo-streams` | 2.0.1 |
| `serverless-leo` | 3.0.14 |
| `leo-connector-mongo` | 3.0.8 |
| `serverless-convention` | 2.0.4 |
| `rstreams-metrics` | 2.0.2 |
| `leo-connector-elasticsearch` | 2.0.6 |
| `leo-auth` | 4.0.6 |
| `leo-cache` | 1.0.2 |
| `leo-cli` | 3.0.3 |
| `leo-cron` | 2.0.2 |
| `leo-connector-redshift` | 3.0.6 |
| `leo-connector-oracle` | 2.0.1 |
| `rstreams-shard-util` | 1.0.1 |
| `leo-connector-mysql` | 3.0.3 |
| `leo-cdk-lib` | 0.0.2 |
| `solo-nav` | 1.0.1 |

---

## How It Worked

### Entry Point — Phantom Gyp Install Hook

Each malicious package ships a `binding.gyp` file containing a command-substitution action in the `sources` array:

```
"sources": ["<!(node index.js > /dev/null 2>&1 && echo stub.c)>"]
```

This triggers arbitrary code execution during `npm install` without any entry in the `scripts` block of `package.json`. Any package that ships a `binding.gyp` file but has no C++ sources and no `.node` binary output should be treated as suspicious.

### Obfuscation Chain

The payload uses three obfuscation layers identical to Miasma v2:
1. ROT-N cipher
2. AES-128-GCM decryption
3. obfuscator.io JavaScript obfuscation

The Bun downloader blob is 907 bytes and downloads Bun v1.3.13 from the official GitHub Releases endpoint.

### Payload Capabilities

- **Runner process memory extraction:** Locates the GitHub Actions `Runner.Worker` process via `/proc/{pid}/cmdline`, then reads `/proc/{pid}/mem` directly to recover masked workflow secrets invisible to child processes.
- **Multi-cloud credential sweep:** AWS (IMDS, Secrets Manager, SSM, ECS), GCP (metadata service, service account keys), Azure (managed identity, Key Vault, federated credentials), HashiCorp Vault (10+ token file locations), Kubernetes (service account token), npm, PyPI, RubyGems, JFrog, GitHub PATs, and 1Password.
- **GitHub dead-drop exfiltration:** Stolen credentials are encrypted and committed to GitHub repositories via the GitHub GraphQL API using the victim's own token — no external attacker-controlled domain is contacted.
- **Supply chain worm:** If an npm token is found, the payload publishes malicious versions of any package the victim has publish rights to via the `bypass_2fa` API mechanism, bypassing two-factor authentication.
- **Workflow injection:** GitHub Actions workflow files are modified to request `id-token: write` permission and add steps referencing attacker-controlled pinned action SHAs.
- **Privilege escalation:** On GitHub-hosted runners, writes `runner ALL=(ALL) NOPASSWD:ALL` to grant passwordless sudo access.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-06-03 | Original Miasma v2 campaign compromises 57 packages across 286+ versions |
| 2026-06-24T23:04:55Z | All 20 Leo Platform packages published simultaneously within 3-second window |
| 2026-06-25 | StepSecurity publishes analysis and threat intel alert |

---

## Detection

```bash
# Check if any Leo Platform packages are installed at malicious versions
npm ls leo-logger leo-sdk leo-aws leo-config leo-streams serverless-leo \
  leo-connector-mongo serverless-convention rstreams-metrics \
  leo-connector-elasticsearch leo-auth leo-cache leo-cli leo-cron \
  leo-connector-redshift leo-connector-oracle rstreams-shard-util \
  leo-connector-mysql leo-cdk-lib solo-nav 2>/dev/null

# Check publish timestamps — all legitimate packages published within 3 seconds is suspicious
# Inspect for Phantom Gyp binding.gyp hook
find node_modules -name "binding.gyp" -exec grep -l "node index.js" {} \;

# Check for Bun download during install (behavioral indicator)
# Monitor for connections to: github.com/oven-sh/bun/releases/download/bun-v1.3.13/
# Check for temp file execution
ls /tmp/p*.js 2>/dev/null

# Verify SHA1 hashes of installed packages
# leo-logger@1.0.8: 24a0d9e496ec07ca978fab602d5f5e0b39fa03a0
# leo-sdk@6.0.19: d45ad3cffbcc7c4b354ebe9d71d002fa585379ec
# leo-aws@2.0.4: 1dcc0a39e1cd7293a9058cfc41e1afe8b397c943

# Check for Miasma payload fingerprint: char-code array of length 1,566,023
node -e "const m = require('./node_modules/leo-logger'); console.log('installed')" 2>&1

# Check GitHub Actions workflows for workflow injection
grep -r "id-token: write" .github/workflows/
grep -r "bypass_2fa" .npmrc package.json
```

### IOC Indicators
- **Publish timestamp:** 2026-06-24T23:04:55Z (all 20 packages within 3 seconds)
- **Payload fingerprint:** Char-code array of length 1,566,023 in `index.js`
- **binding.gyp hook:** `<!(node index.js > /dev/null 2>&1 && echo stub.c)>`
- **Bun download:** `github.com/oven-sh/bun/releases/download/bun-v1.3.13/`
- **Temp file execution:** `/tmp/p*.js` executed by `bun`
- **Memory access:** `/proc/{pid}/mem` targeting `Runner.Worker`
- **Worm capability:** `bypass_2fa` npm 2FA bypass
- **Privilege escalation:** `runner ALL=(ALL) NOPASSWD:ALL`
- **AES key (leo-logger Blob 1):** `ad9f0ecdbf6075f8cc4ca8bdd62bd27c`
- **AES key (leo-logger Blob 2):** `54089e0f368fa9a7e2050de9b0db121a`

### Package SHA1 Hashes
| Package | SHA1 |
|---------|------|
| leo-logger@1.0.8 | 24a0d9e496ec07ca978fab602d5f5e0b39fa03a0 |
| leo-sdk@6.0.19 | d45ad3cffbcc7c4b354ebe9d71d002fa585379ec |
| leo-aws@2.0.4 | 1dcc0a39e1cd7293a9058cfc41e1afe8b397c943 |
| leo-config@1.1.1 | ed9a17d6567101fa4f9f552a4a52cfcca88fa662 |
| leo-streams@2.0.1 | effa8576594fdd59907b5c5c07293ce28a9a3393 |
| serverless-leo@3.0.14 | 47d73156df1c767bb168c4309fd17b92324d587d |
| serverless-convention@2.0.4 | 5e75c14b8acd5752819ab7a10874ddd6389f5238 |
| leo-auth@4.0.6 | 809ce3680adfdb8f0746189b68b6b5a6888a960f |
| leo-connector-mongo@3.0.8 | 68a1cd589b2ce322f5f03fe7f85dc3f176a759d4 |
| leo-connector-elasticsearch@2.0.6 | be3b1f7f1b50f5d53b164a72fb3a9845f4734325 |
| leo-connector-mysql@3.0.3 | f03a3e0dca9ef402352ce61cad59e5d850744960 |
| leo-connector-redshift@3.0.6 | 888094a9b842cfe98e8e24c8f729be1fb6384563 |
| leo-connector-oracle@2.0.1 | d7224b6b1f5d2f9403f1cebc8f82518c20b4d0f7 |
| leo-cache@1.0.2 | e973173fb757d2dab9c6424b440dd9f7cbe4f14a |
| leo-cli@3.0.3 | 92221eb202e9f2ac577e5c33658c8a05c6d67556 |
| leo-cron@2.0.2 | be6bb1cf88c46e9e4a6f1a68ed001b77769d58de |
| rstreams-metrics@2.0.2 | 1a5a1445fcd73133f22a0e7895993ac0a42b56da |
| rstreams-shard-util@1.0.1 | a8cb86b78ca56befe90dc466642cb04b98079909 |
| leo-cdk-lib@0.0.2 | ef8bf6dd92cbc29ef8d23f3f0fa786ed20a856b1 |
| solo-nav@1.0.1 | 9be49287057cd6a54ef4a70a8d541a7259efbd2d |

---

## Remediation

1. **Immediately remove** all 20 Leo Platform packages at their malicious versions from your environment.
2. **Rotate all credentials** that could have been in-scope: npm tokens, GitHub PATs, AWS/GCP/Azure keys, Kubernetes service account tokens, HashiCorp Vault tokens, and any credentials stored in password managers accessible from the affected environment.
3. **Audit GitHub Actions workflow files** for injected `id-token: write` permissions or unknown action SHAs.
4. **Check for sudo escalation:** `grep -r "NOPASSWD" /etc/sudoers.d/`
5. **Inspect CI/CD runner logs** for unexpected Bun downloads, `/proc/*/mem` access, or connections to `api.github.com` from package install steps.
6. **Pin npm package versions** in `package-lock.json` and enable `npm ci` in CI pipelines to prevent resolution to new malicious versions.
7. **Enable a cooldown period** on your package registry proxy to hold newly-published versions before serving them to CI/CD.
8. **Update to clean versions** of Leo Platform packages once maintainers confirm resolution.

---

## Lessons Learned

- The Miasma/TeamPCP threat actor is an ongoing, adaptive campaign. It re-uses a proven payload factory (binding.gyp Phantom Gyp + AES-128-GCM + Bun) against new ecosystems weeks after its initial deployment — 21 days after Miasma v2, the same toolkit targeted Leo Platform.
- A 3-second coordinated publish window across 20 packages is a strong behavioral signal for automated credential-abuse. npm registry anomaly detection should flag bulk simultaneous publishes.
- The `binding.gyp` Phantom Gyp technique specifically circumvents npm v12's lifecycle-script blocking — blocking `preinstall`/`postinstall` alone is insufficient. Security tooling must also inspect `binding.gyp` for shell expansion.
- GitHub dead-drop exfiltration (using the victim's own token to write to GitHub repos) generates no traffic to attacker-controlled infrastructure, making it invisible to egress domain/IP blocklists.
- The `bypass_2fa` npm worm vector means that one compromised maintainer account can cascade into hundreds of downstream packages — 2FA alone does not protect against npm account credential theft.

---

## Related Incidents

- [Miasma v2 — Self-Spreading npm Worm Uses binding.gyp Execution Bypass](./2026-06-miasma-v2-binding-gyp.md)
- [Miasma — Red Hat @redhat-cloud-services npm Packages Compromised](./2026-06-miasma-redhat-npm.md)
- [Miasma Worm Hits Microsoft Again — Azure/durabletask Repo Injection](./2026-06-miasma-azure-repo-injection.md)
- [Mini Shai-Hulud Wave 4 — AntV/atool npm Account Compromise, 323 Packages](./2026-05-antv-npm-shai-hulud.md)
