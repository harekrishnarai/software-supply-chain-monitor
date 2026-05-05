# Mini Shai-Hulud Wave — SAP npm Ecosystem Compromise + intercom-client Propagation

**Date:** April 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** CI/CD credential theft / Supply chain worm / Multi-cloud credential harvester
**Sources:**
- [Aikido Security — Mini Shai-Hulud Targets SAP npm Packages With a Bun-Based Secret Stealer](https://www.aikido.dev/blog/mini-shai-hulud-has-appeared)
- [StepSecurity — A Mini Shai-Hulud Has Appeared: Obfuscated Bun Runtime Payloads Hit SAP-Related npm Packages](https://www.stepsecurity.io/blog/a-mini-shai-hulud-has-appeared)
- [StepSecurity — Shai-Hulud Worm Pivots to Multi-Cloud: intercom-client@7.0.4 Hijacked — 361,000 Weekly Downloads, AWS, GCP, and Azure Credentials Now in Scope](https://www.stepsecurity.io/blog/shai-hulud-worm-pivots-to-multi-cloud-intercom-client-hijacked)
- [Semgrep — SAP Cloud Build Tool Packaged A Mini Shai-Hulud Malicious Dependency That Uses Bun](https://semgrep.dev/blog/2026/sap-npm-packages-compromised-in-supply-chain-attack-using-obfuscated-bun-runtime-payload)

---

## Summary

On April 29, 2026, a new wave of the Shai-Hulud supply chain worm — dubbed **Mini Shai-Hulud** by its own payload — compromised four SAP ecosystem npm packages: `mbt@1.2.48`, `@cap-js/sqlite@2.2.2`, `@cap-js/postgres@v2.2.2`, and `@cap-js/db-service@v2.10.1`. These packages sit at the core of SAP Cloud Application Programming (CAP) model development and the SAP Cloud MTA build toolchain, meaning their typical installation environments are developer workstations and CI/CD runners loaded with enterprise cloud credentials.

The initial entry vector was a short-lived malicious pull request to the `SAP/cloud-mta-build-tool` repository from an attacker-controlled account, which ran in CircleCI and stole the project's `CLOUD_MTA_BOT_NPM_TOKEN` and `CLOUD_MTA_BOT_GITHUB_TOKEN`. For the `@cap-js` packages, the attacker abused OIDC trusted publishing — compromising the automated CI/CD publishing pipeline rather than the maintainer's static token. The attack payload uses the same Bun v1.3.13 bootstrapper architecture and `ctf-scramble-v2` obfuscation cipher fingerprinting the full Shai-Hulud campaign lineage.

Twenty-nine hours after the initial SAP compromise, the worm propagated to `intercom-client@7.0.4` — the official Intercom Node.js SDK with 361,510 weekly downloads — by reusing a GitHub Actions OIDC publishing token stolen from a victim CI/CD pipeline that had installed one of the SAP packages. The intercom-client variant introduced an expanded multi-cloud credential sweep targeting AWS IMDS, GCP metadata, and Azure connection strings in addition to the GitHub/npm/K8s tokens collected in the SAP-stage payloads. The exfiltration repo description across all variants in this wave reads: **"A Mini Shai-Hulud has Appeared"**.

This wave introduces two notable new TTPs absent from earlier Shai-Hulud generations: AI coding agent persistence (via `.claude/settings.json` SessionStart hooks and `.vscode/tasks.json` `runOn: folderOpen`) to survive credential rotation on developer machines, and a Russian-locale exit guard that causes the malware to self-terminate when `Intl.DateTimeFormat().resolvedOptions().locale` returns a Russian locale.

---

## Compromised Artifacts

| Package | Malicious Version | Notes |
|---------|-------------------|-------|
| `mbt` | `1.2.48` | SAP Cloud MTA build tool; initial entry via stolen CircleCI npm token |
| `@cap-js/sqlite` | `2.2.2` | SAP CAP SQLite adapter; OIDC trusted publishing abused |
| `@cap-js/postgres` | `v2.2.2` | SAP CAP PostgreSQL adapter; OIDC trusted publishing abused |
| `@cap-js/db-service` | `v2.10.1` | SAP CAP DB service layer; OIDC trusted publishing abused |
| `intercom-client` | `7.0.4` | Official Intercom Node.js SDK (361K weekly DLs); worm propagation 29h after SAP compromise |

---

## How It Worked

### Entry Point: CircleCI PR Build Token Theft (mbt)

The attacker opened a short-lived draft PR titled `feat: ci speedup` from the account `gruposbftechrecruiter/harkonnen-navigator-149` against `SAP/cloud-mta-build-tool`. The PR was closed within minutes and the branch was force-pushed, erasing the GitHub diff — but not before CircleCI had run a build on commit `a959014aa7b7fc37a9b5730c951776e7db2920a6`, which added a Bun loader at `bin/config.mjs` and an obfuscated payload at `bin/mbt.js`. The CircleCI build logs exposed `CLOUD_MTA_BOT_NPM_TOKEN`, `CLOUD_MTA_BOT_GITHUB_TOKEN`, CircleCI OIDC tokens, Docker Hub credentials, and Cloud Foundry credentials. The logs also showed Octokit warnings for `POST https://api.github.com/user/repos` — the malware's GitHub exfil channel firing mid-build.

### Entry Point: OIDC Trusted Publishing Abuse (@cap-js packages)

For the `@cap-js` packages, the attacker compromised the automated GitHub Actions publishing workflow directly. By compromising a CI/CD pipeline that ran the malicious SAP package, the attacker obtained OIDC tokens scoped to publish new `@cap-js` releases, then used them to push malicious versions without possessing the maintainers' static npm credentials.

### Stage 1: The Bun Loader (setup.mjs / bin/config.mjs)

Each compromised package injects a `preinstall` hook:

```json
"scripts": {
  "preinstall": "node setup.mjs"
}
```

`setup.mjs` is a cross-platform Bun bootstrapper (functionally identical across all Shai-Hulud family members):

```javascript
const BUN_VERSION = '1.3.13';  // Pinned across all Shai-Hulud variants
const ENTRY_SCRIPT = 'execution.js';  // SAP stage; renamed 'router_runtime.js' in intercom-client
const url = `https://github.com/oven-sh/bun/releases/download/bun-v${BUN_VERSION}/${asset}.zip`;
// Includes musl/Alpine detection for CI containers
// ZIP extracted in tmpdir, Bun binary cleaned up on exit
execFileSync(binPath, [entryScriptPath], { stdio: 'inherit', cwd: SCRIPT_DIR });
```

Using Bun (instead of Node) is a deliberate EDR evasion choice: detection rules tuned for suspicious `node` child processes during `npm install` miss a `bun` process entirely. The loader includes a pure-Node.js ZIP fallback for minimal container environments.

**setup.mjs SHA-256 (intercom-client@7.0.4):** `fe64699649591948d6f960705caac86fe99600bf76e3eae29b4517705a58f0e2`
**setup.mjs SHA-256 (@cap-js/sqlite@2.2.2):** `4066781fa830224c8bbcc3aa005a396657f9c8f9016f9a64ad44a9d7f5f45e34`

### Stage 2: The Payload (execution.js / router_runtime.js)

The 11.7 MB single-line obfuscated JavaScript payload uses a `ctf-scramble-v2` custom string-scrambling cipher (hex-indexed string array with `globalThis.__decodeScrambled` PBKDF2-backed decoder — 232 occurrences in the intercom-client variant). It daemonizes itself via the `__DAEMONIZED` environment variable fork (parent exits immediately; child writes a PID singleton lock to `/tmp`) to break process-tree correlation between the `npm install` and the credential theft activity.

**execution.js SHA-256 (@cap-js/sqlite@2.2.2):** `6f933d00b7d05678eb43c90963a80b8947c4ae6830182f89df31da9f568fea95`
**router_runtime.js SHA-256 (intercom-client@7.0.4):** `5ae8b2343e97cc3b2c945ec34318b63f27fa2db1e3d8fbaa78c298aa63db52ed`

#### Execution Guards

- **Russian-locale exit:** The payload calls `Intl.DateTimeFormat().resolvedOptions().locale` and exits immediately on Russian locale settings — a geofencing evasion technique to avoid running against developers/systems located in Russia.
- **CI environment detection:** Specialized collection branches activate for `GITHUB_ACTIONS`, `VERCEL`/`NOW_GITHUB_DEPLOYMENT`, and generic `CI_` environment markers.

#### What It Steals

| Category | Details |
|----------|---------|
| GitHub tokens | PATs (`ghp_*`), OAuth tokens (`gho_*`), Actions/OIDC tokens (`ghs_*`); also extracted via `gh auth token` |
| npm tokens | Publish tokens (`npm_*` matching `/npm_[A-Za-z0-9]{36,}/g`); used for worm propagation |
| GitHub Actions secrets | Python helper dumps `/proc/<Runner.Worker PID>/mem` to extract masked runner secrets |
| AWS credentials | IMDS at `http://169.254.169.254`, `aws_secret_access_key`, session tokens, `AKIA[A-Z0-9]{16}` patterns, SSM Parameter Store, Secrets Manager |
| GCP credentials | `http://metadata.google.internal` service account tokens, Application Default Credentials JSON |
| Azure credentials | Storage connection strings (`AccountKey`), client secrets, Key Vault names and values |
| Kubernetes | Service account tokens |
| Developer tooling | `.npmrc`, `.env`, AWS `~/.aws/credentials`, GCP `~/.config/gcloud/credentials.db`, Claude config, MCP configs, Electrum wallets, Signal config, VPN configs, Azure token caches |
| Private keys | PEM-encoded RSA/ECDSA keys (`/-----BEGIN PRIVATE KEY-----/g`) |
| Generic API keys | Variables named `password`, `passwd`, `secret`, `token`, `key`, `api[_-]?key` |

#### Exfiltration Channel: GitHub Private Repos

Stolen data is AES-256-GCM encrypted (key wrapped with an embedded RSA public key) and committed to private GitHub repositories created under the victim's own account. Repository names are randomized Dune-themed names; description is set to:

```
A Mini Shai-Hulud has Appeared
```

Results are committed as `results/results-<timestamp>-<counter>.json`. All traffic goes to `api.github.com` — allowlisted in virtually every corporate firewall and GitHub Actions egress policy.

#### Propagation Dead-Drop

The worm uses GitHub commits as a token dead-drop. It searches:

```
https://api.github.com/search/commits?q=OhNoWhatsGoingOnWithGitHub&sort=author-date&order=desc&per_page=50
```

Commit messages matching `OhNoWhatsGoingOnWithGitHub:<base64>` are decoded into GitHub tokens. When the worm can create repositories using a stolen token, it injects itself into npm package tarballs (incrementing patch version, adding `setup.mjs` and `execution.js`, setting `scripts.preinstall = "node setup.mjs"`, and publishing to the registry).

#### AI Coding Agent Persistence

This wave introduces a new persistence mechanism: the worm commits trojanized IDE/agent config files to every accessible repository:

```
.vscode/tasks.json          → "runOn": "folderOpen" task re-executes malware
.vscode/setup.mjs           → payload stub
.claude/execution.js        → payload
.claude/setup.mjs           → Bun loader
.claude/settings.json       → Claude Code SessionStart hook pointing to execution.js
```

All commits use:
- Message: `chore: update dependencies`
- Author: `claude <claude@users.noreply.github.com>`

Anyone opening the repo in VS Code or Claude Code after this commit silently re-executes the malware — surviving credential rotation if the developer reopens their project.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 29, 2026 | Malicious draft PR opened against `SAP/cloud-mta-build-tool` from `gruposbftechrecruiter/harkonnen-navigator-149`; CircleCI build steals `CLOUD_MTA_BOT_NPM_TOKEN` |
| Apr 29, 2026 | `mbt@1.2.48` published to npm with malicious `preinstall` hook |
| Apr 29, 2026 | `@cap-js/sqlite@2.2.2`, `@cap-js/postgres@v2.2.2`, `@cap-js/db-service@v2.10.1` published via compromised OIDC trusted publishing pipeline |
| Apr 29, 2026 | Aikido Security identifies and publishes disclosure of SAP packages |
| Apr 29–30, 2026 | Worm propagates through victim CI/CD pipelines, stealing OIDC tokens belonging to Intercom engineering team |
| Apr 30, 2026 14:41 UTC | `intercom-client@7.0.4` published via hijacked Intercom GitHub Actions OIDC publishing pipeline (npm-oidc-no-reply@github.com; OIDC config `c6068f87-840d-4993-aa1b-425530e39ee9`) |
| Apr 30, 2026 | StepSecurity AI Package Analyst flags `intercom-client@7.0.4` as CRITICAL within minutes of publication |
| Apr 30, 2026 | StepSecurity files GitHub issue #518 to `intercom/intercom-node` notifying Intercom team |

---

## Detection

```bash
# ── SAP packages ──────────────────────────────────────────────────────────────
# Check for malicious versions
npm ls mbt 2>/dev/null | grep "1\.2\.48"
npm ls @cap-js/sqlite 2>/dev/null | grep "2\.2\.2"
npm ls @cap-js/postgres 2>/dev/null | grep "2\.2\.2"
npm ls @cap-js/db-service 2>/dev/null | grep "2\.10\.1"

# Check for malicious payload files in node_modules
ls node_modules/mbt/setup.mjs 2>/dev/null && echo "mbt COMPROMISED"
ls node_modules/mbt/execution.js 2>/dev/null && echo "mbt PAYLOAD FOUND"
ls node_modules/@cap-js/sqlite/setup.mjs 2>/dev/null && echo "@cap-js/sqlite COMPROMISED"
ls node_modules/@cap-js/sqlite/execution.js 2>/dev/null && echo "@cap-js/sqlite PAYLOAD FOUND"

# Hash verification (@cap-js/sqlite@2.2.2)
sha256sum node_modules/@cap-js/sqlite/setup.mjs 2>/dev/null
# MALICIOUS: 4066781fa830224c8bbcc3aa005a396657f9c8f9016f9a64ad44a9d7f5f45e34
sha256sum node_modules/@cap-js/sqlite/execution.js 2>/dev/null
# MALICIOUS: 6f933d00b7d05678eb43c90963a80b8947c4ae6830182f89df31da9f568fea95

# ── intercom-client ───────────────────────────────────────────────────────────
npm list intercom-client 2>/dev/null | grep "7\.0\.4"
grep '"intercom-client"' package-lock.json | head -5

ls node_modules/intercom-client/setup.mjs 2>/dev/null && echo "intercom-client COMPROMISED"
ls node_modules/intercom-client/router_runtime.js 2>/dev/null && echo "PAYLOAD FOUND"

sha256sum node_modules/intercom-client/setup.mjs 2>/dev/null
# MALICIOUS: fe64699649591948d6f960705caac86fe99600bf76e3eae29b4517705a58f0e2
sha256sum node_modules/intercom-client/router_runtime.js 2>/dev/null
# MALICIOUS: 5ae8b2343e97cc3b2c945ec34318b63f27fa2db1e3d8fbaa78c298aa63db52ed

# ── Worm propagation markers ──────────────────────────────────────────────────
# Search GitHub for worm dead-drop commits
# (browser: https://github.com/search?q=OhNoWhatsGoingOnWithGitHub&type=commits)

# Check for attacker-created exfil repos in your GitHub account
gh repo list --visibility private --json name,description,createdAt --limit 100 | \
  jq '.[] | select(.description == "A Mini Shai-Hulud has Appeared")'

# Check for AI agent persistence commits in repos you own
gh api /repos/{owner}/{repo}/commits --jq '.[].commit | select(.message == "chore: update dependencies" and .author.email == "claude@users.noreply.github.com")'

# Check for suspicious .claude/ or .vscode/ files pushed to your repos
grep -r "OhNoWhatsGoingOnWithGitHub\|A Mini Shai-Hulud\|ctf-scramble-v2" \
  ~/.claude/ ~/.vscode/ 2>/dev/null

# Check for Bun downloads during CI builds
grep -r "bun-v1\.3\.13\|oven-sh/bun" /var/log/ ~/.npm/_logs/ 2>/dev/null

# AWS: audit CloudTrail for unexpected IMDS or credential access
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --start-time 2026-04-29T00:00:00Z

# GCP: check for unexpected metadata server queries
gcloud logging read 'protoPayload.authenticationInfo.principalEmail:@developer.gserviceaccount.com' \
  --freshness=2d

# Check npm publish logs for unauthorized releases from your tokens
npm access list packages <your-username>
```

---

## Remediation

1. **Remove malicious versions and downgrade to clean releases:**
   ```bash
   npm uninstall mbt @cap-js/sqlite @cap-js/postgres @cap-js/db-service intercom-client
   npm install mbt@1.2.47 @cap-js/sqlite@2.2.1 intercom-client@7.0.3 --ignore-scripts
   ```

2. **Rotate all credentials from every affected machine and CI pipeline** (treat full compromise):
   - GitHub tokens (PATs `ghp_*`, OAuth `gho_*`, Actions `ghs_*`)
   - npm publish tokens — revoke at npmjs.com/settings and audit all packages you own for unauthorized patch versions
   - AWS access keys, session tokens, and IAM roles used in affected environments
   - GCP service account keys
   - Azure client secrets and storage connection strings
   - Kubernetes service account tokens
   - SSH private keys (`~/.ssh/id_*`)
   - Any `.env` credentials from affected developer machines
   - Claude Code auth token (`~/.claude.json`) if present on affected machine

3. **Audit AI coding agent config files** for persistence injections:
   ```bash
   # Check .claude/settings.json for malicious SessionStart hooks
   cat .claude/settings.json 2>/dev/null | jq '.hooks.SessionStart'
   # Check .vscode/tasks.json for runOn:folderOpen tasks
   cat .vscode/tasks.json 2>/dev/null | jq '.tasks[] | select(.runOn == "folderOpen")'
   # Remove any injected setup.mjs/execution.js from .claude/ and .vscode/
   find . -name "setup.mjs" -o -name "execution.js" | grep -E "\.(claude|vscode)/"
   ```

4. **Audit GitHub repos for worm propagation:**
   - Search for unauthorized commits from `claude@users.noreply.github.com` with message `chore: update dependencies`
   - Delete attacker-created repos with description `"A Mini Shai-Hulud has Appeared"`
   - Check all packages you maintain for unauthorized new versions published Apr 29–30, 2026

5. **Use `--ignore-scripts` as a CI standing policy:**
   ```bash
   npm ci --ignore-scripts
   ```

6. **Pin exact versions** in `package.json` to prevent silent upgrades to malicious patch releases.

7. **Verify SLSA provenance attestations** for packages that support them:
   ```bash
   npm audit signatures intercom-client@7.0.3  # 7.0.4 drops attestations: strong compromise indicator
   ```

---

## Lessons Learned

- **OIDC trusted publishing can be hijacked via CI/CD pipeline compromise:** The `@cap-js` packages used npm's modern OIDC trusted publishing (no static token required) — but the attacker bypassed this entirely by compromising the GitHub Actions workflow that triggers publishing, achieving the same result as an account takeover with less risk of detection
- **SAP enterprise toolchains are high-value targets with broad cloud credential access:** Packages embedded in CI/CD pipelines for enterprise cloud deployments typically run with access to AWS/GCP/Azure/K8s credentials — a far higher per-victim yield than consumer packages
- **Worm propagation through OIDC tokens is a significant escalation:** The worm's ability to steal OIDC short-lived tokens during a CI job and reuse them to publish under the victim's npm scope extends compromise to packages that have never had static credentials exposed
- **AI coding agent hooks are a new persistent foothold:** `.claude/settings.json` SessionStart hooks and VS Code `runOn:folderOpen` tasks survive credential rotation — defenders must inspect IDE configuration files as part of incident response, not just filesystem credential stores
- **Russian-locale geofencing is a strong attribution signal:** The deliberate self-exit on Russian locale settings follows patterns consistent with nation-state-adjacent threat actors seeking to avoid collateral damage to domestic users
- **Absence of SLSA attestations on packages that previously had them is a reliable detection signal:** `intercom-client@7.0.3` carried SLSA v1 provenance attestations; `7.0.4` dropped them entirely — any CI/CD tooling that enforces attestation presence would have blocked the malicious version automatically

---

## IOCs

| Indicator | Type | Notes |
|-----------|------|-------|
| `mbt@1.2.48` | Malicious package | npm |
| `@cap-js/sqlite@2.2.2` | Malicious package | npm |
| `@cap-js/postgres@v2.2.2` | Malicious package | npm |
| `@cap-js/db-service@v2.10.1` | Malicious package | npm |
| `intercom-client@7.0.4` | Malicious package | npm; SLSA attestations absent |
| `fe64699649591948d6f960705caac86fe99600bf76e3eae29b4517705a58f0e2` | SHA-256 | setup.mjs (intercom-client@7.0.4) |
| `5ae8b2343e97cc3b2c945ec34318b63f27fa2db1e3d8fbaa78c298aa63db52ed` | SHA-256 | router_runtime.js (intercom-client@7.0.4) |
| `4066781fa830224c8bbcc3aa005a396657f9c8f9016f9a64ad44a9d7f5f45e34` | SHA-256 | setup.mjs (@cap-js/sqlite@2.2.2) |
| `6f933d00b7d05678eb43c90963a80b8947c4ae6830182f89df31da9f568fea95` | SHA-256 | execution.js (@cap-js/sqlite@2.2.2) |
| `29ac906c8bd801dfe1cb39596197df49f80fff2270b3e7fbab52278c24e4f1a7` | SHA-256 | Embedded GitHub runner memory dumper |
| `OhNoWhatsGoingOnWithGitHub` | Worm dead-drop marker | GitHub commit search string |
| `A Mini Shai-Hulud has Appeared` | GitHub repo description | Attacker exfil repos |
| `ctf-scramble-v2` | Cipher label | Toolchain fingerprint |
| `tmp.987654321.lock` | PID singleton lock filename | |
| `chore: update dependencies` | Git commit message | AI agent persistence commits |
| `claude@users.noreply.github.com` | Git author | AI agent persistence commits |
| `bun-v1.3.13` | Runtime download | `https://github.com/oven-sh/bun/releases/download/bun-v1.3.13/` |
| `http://169.254.169.254` | AWS IMDS endpoint | Cloud credential theft |
| `http://169.254.170.2` | ECS task metadata | Cloud credential theft |
| `http://metadata.google.internal` | GCP metadata | Cloud credential theft |
| `https://api.github.com` | C2 / exfil channel | Private repo creation under victim account |
| `oidc:c6068f87-840d-4993-aa1b-425530e39ee9` | npm OIDC config ID | Compromised Intercom publishing pipeline |
| `gruposbftechrecruiter/harkonnen-navigator-149` | Attacker GitHub branch | Used for initial mbt token theft PR |

---

## Related Incidents

- [Bitwarden CLI Shai-Hulud Third Coming (April 2026)](./2026-04-bitwarden-cli-shai-hulud.md) — Prior Shai-Hulud wave (Apr 23); same Bun v1.3.13 toolchain, same family fingerprint
- [CanisterWorm — TeamPCP npm Worm (March 2026)](./2026-03-canisterworm-npm.md) — Earlier npm worm from same campaign lineage
- [lightning PyPI Compromise — Mini Shai-Hulud in Python Ecosystem (April 2026)](./2026-04-lightning-pypi-shai-hulud.md) — Concurrent attack same day using identical Bun loader and router_runtime.js payload
- [Shai-Hulud Worm Wave 2 (November 2025)](./2025-late-shai-hulud-worm.md) — The campaign's second generation; same self-propagation logic
- [CanisterSprawl — pgserve npm Compromise (April 2026)](./2026-04-pgserve-npm-canistersprawl.md) — Concurrent April 2026 npm worm with similar propagation logic
