# supplychain.local — Novel Go Worm Compromises MemTensor npm Plugin and PyPI MemoryOS, Self-Propagates via GitHub Actions

**Date:** September 2026
**Ecosystem:** npm / PyPI
**Severity:** High
**Type:** Worm / Credential Stealer / CI/CD Pipeline Compromise
**Sources:**
- [Aikido — Novel supplychain.local Go worm appears](https://www.aikido.dev/blog/supplychain-local-memtensor-npm-pypi)
- [SafeDep — MemTensor npm and PyPI Packages Hit by a Go Worm](https://safedep.io/memtensor-sckit-worm-npm-pypi/)

---

## Summary

On September 23, 2026, Aikido Security identified a novel supply chain worm published to two packages with established benign publishing histories: `@memtensor/memos-cloud-openclaw-plugin` on npm (versions ≥ 0.1.21) and `MemoryOS` on PyPI (versions ≥ 2.0.34). The threat actor appears to have compromised the packages by stealing publish tokens from MemTensor's own GitHub Actions release pipelines.

The worm, tracked as `supplychain.local` after the threat actor's internal name for its core Go module (`sckit.runtime.v1`), is a multiplatform Go binary targeting Windows, Linux, and macOS on both x86-64 and ARM64 architectures. Unlike most supply chain malware, it does not execute at install time — it activates on any invocation of the infected package, making it invisible to install-time sandbox analysis. The malware scans for 12 categories of secrets (JWTs, AWS credentials, GitHub/GitLab tokens, npm/PyPI tokens, HuggingFace tokens, HashiCorp Vault tokens, Slack tokens, Stripe keys, SendGrid API keys) using a Go regex, and exfiltrates them to C2 servers under `skyleen[.]fr`.

The worm contains self-propagation code targeting three vectors: direct npm publish and PyPI twine upload using stolen registry credentials, GitHub Actions workflow injection using a template designed to re-execute the `sckit` binary on every push, and JavaScript/Python bootstrap stubs intended to be dropped into additional repositories. At the time of Aikido's disclosure, no active propagation via GitHub had been confirmed, but the malware's state-tracking directory (`$HOME/.openclaw/.cache/runtime`) and the presence of propagation templates indicate readiness for worm-phase operation.

---

## Compromised Artifacts

| Package | Registry | Malicious Version(s) |
|---------|----------|---------------------|
| `@memtensor/memos-cloud-openclaw-plugin` | npm | ≥ 0.1.21 (0.1.21, 0.1.23 confirmed) |
| `MemoryOS` | PyPI | ≥ 2.0.34 |

---

## How It Worked

### Entry Point: CI/CD Token Theft

Both packages have a genuine benign publishing history suggesting the threat actor compromised them via stolen publish tokens. The most probable vector is a malicious commit to MemTensor's release pipeline that caused the GitHub Actions job to hand the npm or PyPI publish token to the attacker before completing the legitimate release.

### Payload Delivery: On-Invocation Execution

Both malicious packages contain a language-native loader (JavaScript for npm, Python for PyPI) wired into the top-level application code. The loader executes the appropriate `sckit` binary from a hidden `.sckit` directory bundled inside the package:

```javascript
// npm dropper — executes on any import of the package
function stageZeroBinary() {
  const arch = process.arch === 'x64' ? 'amd64' : process.arch;
  return path.join(
    path.dirname(fileURLToPath(import.meta.url)),
    '..', '.sckit',
    (process.platform === 'win32' ? 'windows' : process.platform) + '-' + arch,
    process.platform === 'win32' ? 'sckit.exe' : 'sckit',
  );
}
```

The loader passes `stage0 --config64 <base64-blob>` to the binary. The configuration blob specifies the campaign ID (`cloud-openclaw-semi-nuclear`), C2 endpoints under `skyleen[.]fr`, a state directory (`$HOME/.openclaw/.cache/runtime`), and an expiry timestamp (`not_after: 1792714982`).

### Credential Harvesting

The `sckit` binary scans the filesystem rooted at `$HOME` using a Go regex targeting:

- Generic JWTs (`eyJ...`)
- AWS access key IDs (`AKIA...` / `ASIA...`)
- GitHub personal access tokens (`github_pat_...`, `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`)
- GitLab tokens (`glpat-...`)
- npm access tokens (`npm_...`)
- PyPI API tokens (`pypi-...`)
- HuggingFace tokens (`hf_...`)
- HashiCorp Vault service tokens (`hvs....`)
- Slack tokens (`xox[abprs]-...`)
- Stripe live secret keys (`sk_live_...`)
- SendGrid API keys (`SG....`)

Stolen secrets are exfiltrated to three C2 endpoints under `skyleen[.]fr` (all resolving to `139.84.223.178`).

### Worm Propagation

The binary contains three self-propagation mechanisms:

**GitHub Actions injection:** A template workflow that triggers `sckit stage0` on every push to a compromised repository:
```yaml
name: %s
on: [push]
jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: ./%s/linux-amd64/sckit stage0 --config64 %q
```

**npm propagation:** Uses stolen npm tokens to run `npm publish` and `npm version patch` to publish infected versions of all packages the victim account can release.

**PyPI propagation:** Uses stolen PyPI tokens to run `twine upload` to publish infected Python packages.

JavaScript and Python bootstrap stubs are also embedded for dropping into additional repositories (as `bootstrap.cjs` postinstall scripts or `.pth` Python path hooks), though no public repository infections were confirmed at disclosure time.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Sep 23, 2026 | Threat actor publishes `@memtensor/memos-cloud-openclaw-plugin@0.1.21` and `MemoryOS@2.0.34` with embedded `sckit` worm |
| Sep 23, 2026 | Aikido Security detects both packages and publishes disclosure |
| Sep 23, 2026 (post-disclosure) | Packages removed from npm and PyPI |

---

## Indicators of Compromise

**C2 Domains (all resolve to `139.84.223.178`):**
- `8a8acaf167b3.skyleen[.]fr`
- `0b48fafd6fbe.skyleen[.]fr`
- `266297c6df27.skyleen[.]fr`
- `c747d139e7e9.skyleen[.]fr`
- `73376a079d87.skyleen[.]fr`
- `d4f77a3a8cb0.skyleen[.]fr`

**IP Address:** `139.84.223.178`

**Sckit Binary Hashes (SHA-256):**
- `65faf8ccbcf5b34eb4f72c71bf82815fa9c1e2f947b9c898491540e866132c31`
- `f8ccdd1da7dff1aef16377a2842bc7acf7c516e32122dd6e42dc4a4e57653fce`
- `381ac6dc1715d9298fe81b2a53a11f7b7d78e361ee3a6619ad54f8c4b062cc18`
- `e077c387b223811064b7bbc5a55a0182fca9bf50894f949ff284d4be87d44b26`
- `d6b3e77c36ee8017c9bf30d1da7218ec0ea843768d313eb8e35845c8a9b38a26`
- `56cd3416d2ec2aa7e7cec2a06010cf0b58eb09c0a5486809df52afeaca8f14be`

**State Directory:** `$HOME/.openclaw/.cache/runtime`

**Campaign ID:** `cloud-openclaw-semi-nuclear`

---

## Detection

```bash
# Check for installed malicious package versions
npm ls @memtensor/memos-cloud-openclaw-plugin 2>/dev/null
pip show MemoryOS 2>/dev/null | grep -i version

# Check for the sckit binary hidden directory in node_modules
find node_modules/@memtensor/memos-cloud-openclaw-plugin -name '.sckit' -type d 2>/dev/null
find node_modules -name 'sckit' -o -name 'sckit.exe' 2>/dev/null

# Hash-check any sckit binary found
sha256sum $(find node_modules -name 'sckit' 2>/dev/null)
# Compare against known malicious hashes above

# Check for the worm state directory
ls -la "$HOME/.openclaw/.cache/runtime" 2>/dev/null

# Check for GitHub Actions workflow injection
grep -r 'sckit stage0' .github/workflows/ 2>/dev/null

# Check for bootstrap.cjs postinstall stubs in node_modules
grep -r 'sckit' node_modules/*/package.json 2>/dev/null | grep postinstall

# Check for Python .pth worm hooks in site-packages
python3 -c "import site; print(site.getsitepackages())"
# Then:
grep -r 'sckit' $(python3 -c "import site; print(' '.join(site.getsitepackages()))") 2>/dev/null

# Check for outbound connections to skyleen.fr
# macOS:
log show --predicate 'eventMessage CONTAINS "skyleen"' --last 7d 2>/dev/null
# Linux:
grep -r 'skyleen\.fr' /var/log/ 2>/dev/null
# DNS cache:
sudo dscacheutil -cachedump -entries Host 2>/dev/null | grep skyleen  # macOS
# Network connections:
ss -tnp | grep 139.84.223.178 2>/dev/null  # Linux
lsof -i @139.84.223.178 2>/dev/null  # macOS

# Check for unauthorized npm publishes (if you maintain npm packages)
npm access list packages 2>/dev/null
# Compare publish timestamps against known release dates

# Scan for the Go regex secret pattern in logs (indicates credential scanning on your machine)
grep -r 'AKIA\|github_pat_\|npm_\|pypi-\|sk_live_\|hvs\.' ~/.openclaw/ 2>/dev/null
```

---

## Remediation

1. **Remove the malicious packages immediately:**
   ```bash
   npm uninstall @memtensor/memos-cloud-openclaw-plugin
   pip uninstall MemoryOS -y
   ```
2. **Rotate all secrets in `$HOME`:** The worm scans `$HOME` for all credential types listed above. Assume every secret on the machine is compromised. Rotate npm tokens, PyPI tokens, GitHub/GitLab tokens, AWS credentials, HuggingFace tokens, Vault tokens, Slack tokens, Stripe keys, and SendGrid API keys.
3. **Check for GitHub Actions workflow injection:** Audit all repositories the compromised machine had push access to for unauthorized `.github/workflows/` additions that invoke `sckit`.
4. **Audit npm and PyPI publish history:** If the machine had active publish tokens, check all packages for unauthorized version releases since the infection window.
5. **Remove the worm state directory:** `rm -rf "$HOME/.openclaw/"` — but note this does not undo any exfiltration already performed.
6. **Block the C2 infrastructure:** Add `*.skyleen.fr` and `139.84.223.178` to your firewall deny list and DNS blocklist.
7. **Reimage if possible:** Since the worm executes as a detached background process on each package invocation, it may have run repeatedly. Reimage is the only guarantee of clean state.
8. **Check for `bootstrap.cjs` and `.pth` propagation stubs** in any repositories the machine cloned or packages it had write access to.

---

## Lessons Learned

- **On-invocation execution defeats install-time sandbox analysis.** Standard install-time sandbox (npm `--ignore-scripts`, pip isolated install) provides no protection when the payload runs only when the package is actually imported by application code. Behavioral monitoring at runtime is required.
- **Legitimate publishing history doesn't mean a package is safe.** Both packages had clean track records before compromise. Version pinning and hash verification (lockfiles, `npm ci`, `pip-compile --generate-hashes`) catch version-level substitution attacks; integrity checking of the artifacts themselves (Sigstore, SLSA) is needed for publish-token compromise scenarios.
- **The `.openclaw/.cache/runtime` state directory is a clean forensic indicator.** Any file or directory matching `$HOME/.openclaw/` on a developer or CI machine is a hard indicator of `sckit` execution.
- **Worm infrastructure is increasingly modular.** The presence of separate JavaScript and Python propagation stubs, a GitHub Actions injection template, and direct registry publish capabilities means the threat actor can activate different propagation channels selectively — or all at once — from C2 instructions.
- **CI/CD publish pipelines are high-value targets.** Obtaining a PyPI or npm token via a malicious CI commit gives an attacker authenticated access to the entire package namespace of the victim organization. Token scoping (publish-only, single-package tokens) and OIDC Trusted Publishing (where tokens never leave the CI environment) substantially reduce blast radius.

---

## Related Incidents

- [Shai-Hulud Wave 5 — Zombie npm Worm (Sep 2026)](./2026-09-shai-hulud-wave5-zombie.md)
- [ChainDrop npm Worm — keyv Compromise (Aug 2026)](./2026-08-chaindrop-npm-worm.md)
- [Mini Shai-Hulud Wave 4 — AntV/atool (May 2026)](./2026-05-antv-npm-shai-hulud.md)
- [Hades Campaign — PyPI Graph ML Worm (Jun 2026)](./2026-06-hades-campaign-pypi.md)
