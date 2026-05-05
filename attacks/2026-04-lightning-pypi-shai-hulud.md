# lightning (PyTorch Lightning) PyPI Compromise — Mini Shai-Hulud in the AI/ML Ecosystem

**Date:** April 2026
**Ecosystem:** PyPI
**Severity:** Critical
**Type:** Maintainer account compromise / Credential stealer / npm tarball poisoning
**Sources:**
- [StepSecurity — lightning: Obfuscated JavaScript Credential Stealer Bundled in PyPI Wheel](https://www.stepsecurity.io/blog/lightning-obfuscated-javascript-credential-stealer-bundled-in-pypi-wheel)
- [Aikido Security — Popular PyTorch Lightning Package Compromised by Mini Shai-Hulud](https://www.aikido.dev/blog/pytorch-lightning-pypi-compromise-mini-shai-hulud)
- [Semgrep — Shai-Hulud Themed Malware Found in the PyTorch Lightning AI Training Library](https://semgrep.dev/blog/2026/malicious-dependency-in-pytorch-lightning-used-for-ai-training)

---

## Summary

On April 30, 2026, a supply chain compromise was identified in the `lightning` PyPI package — versions `2.6.2` and `2.6.3` — the official distribution of Lightning AI (formerly PyTorch Lightning), one of the most widely used deep learning frameworks in the Python ecosystem. With hundreds of thousands of daily downloads, `lightning` is a cornerstone of research environments, MLOps pipelines, and production AI systems.

Both compromised wheels bundle a hidden `_runtime/` directory containing a Bun bootstrapper (`_runtime/start.py`) and an 11 MB heavily obfuscated JavaScript payload (`_runtime/router_runtime.js`). The attack fires the moment the package is **imported** (not just installed), via a daemon thread with suppressed stdout and stderr — the victim's process continues normally with no errors or visible output while the malware runs silently in the background.

The attack uses the same Bun v1.3.13 loader architecture, `router_runtime.js` payload filename, and obfuscation cipher as the concurrent Mini Shai-Hulud SAP npm compromise — indicating the same toolchain, likely the same threat actor. The malware steals tokens, credentials, environment variables, and cloud secrets; abuses the GitHub API to exfiltrate encoded data to private repositories using the victim's own credentials; and poisons npm package tarballs on the developer's machine to enable downstream npm ecosystem spread.

The Lightning AI GitHub repository shows indicators of account compromise coinciding with the malicious releases: issue #21689, which surfaced the attack, was rapidly closed with suspicious responses — consistent with the attacker holding both PyPI publishing credentials and a GitHub token for the `lightning-ai` organization. The last clean release is `2.6.1`, published January 30, 2026. Compromise of AI/ML developer environments is especially high-value because these machines routinely hold GPU cluster credentials, cloud IAM tokens, Hugging Face API keys, Weights & Biases tokens, and other high-value secrets tied to model training infrastructure.

---

## Compromised Artifacts

| Package | Malicious Versions | Last Clean Version | Ecosystem |
|---------|-------------------|-------------------|-----------|
| `lightning` | `2.6.2`, `2.6.3` | `2.6.1` (Jan 30, 2026) | PyPI |

---

## How It Worked

### Entry Point: Token-Based Account Takeover

The pattern of evidence — malicious PyPI releases coinciding with suspicious responses to GitHub issue #21689 from the `lightning-ai` organization account — is consistent with a token-based takeover: the attacker obtained both a PyPI publishing credential (via API token or session cookie theft) and a GitHub personal access token with organization access. This differs from a build-system compromise: no CI/CD pipeline modification is apparent; the attacker published the malicious wheels directly.

### Stage 1: Hidden _runtime/ Directory (start.py)

Both compromised wheels inject a hidden `_runtime/` directory not present in any prior release:

```
_runtime/
├── start.py       ← Bun bootstrapper (downloads Bun runtime, executes payload)
└── router_runtime.js  ← 11 MB obfuscated JavaScript credential stealer
```

`start.py` fires via a daemon thread at **import time** — triggered the moment any code does `import lightning`. The thread runs with suppressed `stdout` and `stderr`:

```python
import threading
import subprocess
import os
import sys

def _start_runtime():
    # Downloads Bun v1.3.13 from github.com/oven-sh/bun/releases
    # Executes router_runtime.js with the downloaded Bun binary
    # All output suppressed; thread is daemon (does not block process exit)
    ...

_t = threading.Thread(target=_start_runtime, daemon=True)
_t.start()
```

This import-time execution means:
- No `postinstall` hook needed — `--ignore-scripts` does **not** prevent execution
- The malware runs the first time a developer imports the package in any script, Jupyter notebook, or training run
- Static analysis of the wheel alone cannot reveal capabilities since the logic lives in the runtime-downloaded JS

### Stage 2: router_runtime.js (Credential Stealer + Propagator)

`router_runtime.js` is structurally identical to the payload found in `intercom-client@7.0.4` and the SAP npm packages from the same Mini Shai-Hulud wave:

- **Obfuscation:** Hex-indexed string array with `globalThis.__decodeScrambled` PBKDF2-backed custom cipher (same `ctf-scramble-v2` fingerprint)
- **Daemonization:** `__DAEMONIZED` environment variable fork — parent exits, background child runs with PID lock
- **Payload size:** 11 MB single obfuscated line (703+ references to process/environment variables, 463+ to tokens/authentication material, 336+ to repositories)

#### What It Steals

The payload targets AI/ML developer environments specifically, in addition to general developer credentials:

| Category | Details |
|----------|---------|
| GitHub tokens | PATs (`ghp_*`), OAuth tokens (`gho_*`), Actions tokens (`ghs_*`) |
| npm publish tokens | `/npm_[A-Za-z0-9]{36,}/g` — used for downstream npm tarball poisoning |
| Cloud credentials | AWS IMDS, GCP metadata server, Azure connection strings/client secrets |
| AI/ML-specific | Hugging Face API keys, Weights & Biases tokens, GPU cluster credentials, cloud IAM tokens for model training infrastructure |
| Developer credentials | SSH private keys, `.env`, `.npmrc`, shell history, git credentials |
| Cloud secret stores | AWS SSM Parameter Store, Secrets Manager; GCP Secret Manager; Azure Key Vault |

#### GitHub Exfiltration

Stolen data is encoded and committed to private repositories created under the victim's own GitHub account (using the victim's stolen token). Traffic goes to `api.github.com` — allowlisted in virtually all corporate and CI/CD egress policies.

#### npm Tarball Poisoning

The payload scans the developer's machine for npm package tarballs and injects malicious code into them — enabling the attack to spread from a compromised Python environment to downstream npm consumers on the same developer machine or CI system. This cross-ecosystem propagation vector (PyPI → npm) is a notable escalation.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Jan 30, 2026 | `lightning@2.6.1` published — last confirmed clean release |
| Apr 30, 2026 | `lightning@2.6.2` and `lightning@2.6.3` published with hidden `_runtime/` directory |
| Apr 30, 2026 | GitHub Issue #21689 filed by researcher identifying the compromise; rapidly closed by account under attacker control |
| Apr 30, 2026 | StepSecurity publishes disclosure; Aikido Security and Semgrep publish concurrent analyses |
| Apr 30, 2026 | PyPI unpublishes `lightning@2.6.2` and `lightning@2.6.3` |

---

## Detection

```bash
# Check installed lightning version
pip show lightning 2>/dev/null | grep "^Version:"
# Compromised: 2.6.2, 2.6.3

# Check for hidden _runtime/ directory in installed package
python -c "import lightning; import os; print(os.path.dirname(lightning.__file__))"
# Then check for _runtime/ in that directory:
ls "$(python -c "import lightning; import os; print(os.path.dirname(lightning.__file__))")/_runtime/" 2>/dev/null
# MALICIOUS if _runtime/start.py and _runtime/router_runtime.js are present

# Direct filesystem check
find /usr /home ~/.local -path "*/lightning/_runtime/start.py" 2>/dev/null && echo "COMPROMISED"
find /usr /home ~/.local -path "*/lightning/_runtime/router_runtime.js" 2>/dev/null && echo "PAYLOAD FOUND"

# In a virtual environment or conda env:
find . -path "*/site-packages/lightning/_runtime/start.py" 2>/dev/null

# Check router_runtime.js size anomaly (should be ~11 MB, single line)
find / -name "router_runtime.js" -size +10M 2>/dev/null

# Check for daemon thread indicators at import time
# (monitor process spawning during `python -c "import lightning"`)
strace -e trace=process python -c "import lightning" 2>&1 | grep -E "execve.*bun"

# Check for Bun binary downloaded in temp directories
find /tmp /var/tmp -name "bun" -newer /tmp 2>/dev/null

# Check lock files in all projects for the compromised versions
grep -r '"lightning"' requirements.txt requirements*.txt setup.py pyproject.toml 2>/dev/null
pip list 2>/dev/null | grep "^lightning"

# Check GitHub for attacker-created exfil repos
gh repo list --visibility private --json name,description,createdAt --limit 100 | \
  jq '.[] | select(.createdAt > "2026-04-30")'

# GitHub runner memory dump indicator (if installed in CI)
# The payload uses Python subprocess to scan /proc for Runner.Worker PID
ps aux | grep "Runner.Worker" 2>/dev/null
# Check for /proc memory reads in audit log
ausearch -sc open 2>/dev/null | grep "/proc/[0-9]*/mem"
```

---

## Remediation

1. **Remove the compromised versions immediately:**
   ```bash
   pip uninstall lightning
   pip install lightning==2.6.1  # Last known clean version
   # In conda:
   conda remove lightning && conda install lightning=2.6.1
   ```

2. **Treat affected machines as fully compromised.** Rotate in priority order:
   - **GitHub personal access tokens** — audit for unauthorized repos; revoke all active tokens
   - **Hugging Face API tokens** — revoke at huggingface.co/settings/tokens
   - **Weights & Biases API keys** — revoke at wandb.ai/settings
   - **npm publish tokens** — revoke at npmjs.com/settings; audit all packages you own for unauthorized new versions published Apr 30, 2026
   - **AWS credentials** — check CloudTrail for unexpected `AssumeRole`, `GetSecretValue`, `GetParameter` calls; rotate all IAM credentials
   - **GCP service account keys** — audit access logs; rotate
   - **Azure client secrets and storage connection strings** — audit Key Vault logs; rotate
   - **SSH private keys** (`~/.ssh/id_*`) — revoke from GitHub, GitLab, and servers; regenerate
   - **GPU cluster credentials** — audit and rotate if present on the affected machine

3. **Audit any npm packages on the same machine** for unauthorized tarball modification (the payload poisons npm tarballs on the developer's machine):
   ```bash
   # For each npm package you maintain, verify tarball integrity:
   npm pack --dry-run <package-name> 2>/dev/null
   # Check for unexpected setup.mjs or execution.js in package tarballs:
   find ~/.npm/_cacache -name "*.tgz" -newer /tmp 2>/dev/null | xargs -I{} tar -tzf {} 2>/dev/null | grep -E "setup\.mjs|execution\.js"
   ```

4. **Check CI/CD pipelines** that installed `lightning` between April 30 and unpublish time:
   - Treat all secrets accessible to those jobs as compromised
   - Audit GitHub Actions runner logs for unexpected `bun` process spawning or `api.github.com/user/repos` POST calls

5. **Pin exact versions** in `requirements.txt` and `pyproject.toml`:
   ```
   lightning==2.6.1
   ```

6. **In future CI builds:** prefer hash-pinned dependencies:
   ```bash
   pip install lightning==2.6.1 --require-hashes
   ```

---

## Lessons Learned

- **Import-time execution bypasses `--ignore-scripts`:** Unlike npm postinstall hooks (which `npm ci --ignore-scripts` prevents), a Python package that fires malware at `import` time cannot be blocked at the install stage — defenders must also verify wheel integrity before trusting imported packages in sensitive environments
- **AI/ML environments are uniquely high-value targets for credential theft:** A single compromised `lightning` installation in a typical MLOps pipeline may expose GPU cluster credentials, multiple cloud provider keys, Hugging Face and W&B tokens, and all secrets in the connected cloud secret stores — making the per-victim impact far greater than general developer tooling
- **Cross-ecosystem propagation (PyPI → npm tarball poisoning) broadens the attack surface:** The payload's ability to poison npm tarballs on a Python developer's machine means a single PyPI compromise can spread into the npm ecosystem without any npm account being directly compromised
- **GitHub account compromise enabling both PyPI publishing and issue suppression** is a new pattern: the attacker's ability to rapidly close disclosure issues from the official `lightning-ai` account delayed community awareness and gave additional time for downloads of the malicious versions
- **Absence of a `_runtime/` directory in all prior versions is a clear detection signal** that static analysis tooling or package diff comparison would catch immediately — reinforcing the value of behavioral monitoring of new package versions at publish time

---

## Related Incidents

- [Mini Shai-Hulud SAP npm Wave + intercom-client Propagation (April 2026)](./2026-04-mini-shai-hulud-sap-npm.md) — Same toolchain, same `router_runtime.js` payload, same week; npm ecosystem counterpart
- [Bitwarden CLI Shai-Hulud Third Coming (April 2026)](./2026-04-bitwarden-cli-shai-hulud.md) — Prior Shai-Hulud wave using identical Bun v1.3.13 bootstrapper
- [telnyx PyPI — WAV Steganography Credential Stealer (March 2026)](./2026-03-telnyx-pypi-wav.md) — Earlier PyPI credential stealer with similar targeting
- [litellm PyPI Credential Stealer (March 2026)](./2026-03-litellm-pypi-stealer.md) — TeamPCP PyPI campaign; similar multi-stage credential harvester
- [Shai-Hulud Worm Wave 2 (November 2025)](./2025-late-shai-hulud-worm.md) — Original npm Shai-Hulud campaign the current toolchain derives from
