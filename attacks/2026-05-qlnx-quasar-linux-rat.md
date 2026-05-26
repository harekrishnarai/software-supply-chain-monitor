# QLNX — Quasar Linux RAT Targets Developer Workstations to Enable Supply Chain Compromise

**Date:** May 2026
**Ecosystem:** Linux Developer Environments (credential theft targeting npm, PyPI, GitHub, AWS, GCP, Kubernetes, Docker)
**Severity:** High
**Type:** Linux RAT / Infostealer / Rootkit
**Sources:**
- [Trend Micro — Quasar Linux (QLNX): A Silent Foothold in the Supply Chain — Inside a Full-Featured Linux RAT With Rootkit, PAM Backdoor, Credential Harvesting Capabilities](https://www.trendmicro.com/en_us/research/26/e/quasar-linux-qlnx-a-silent-foothold-in-the-software-chain.html)
- [BleepingComputer — New stealthy Quasar Linux malware targets software developers](https://www.bleepingcomputer.com/news/security/new-stealthy-quasar-linux-malware-targets-software-developers/)
- [SecurityWeek — Sophisticated Quasar Linux RAT Targets Software Developers](https://www.securityweek.com/sophisticated-quasar-linux-rat-targets-software-developers/)
- [SOC Prime — Quasar Linux (QLNX): A Supply Chain Foothold with Full RAT Capabilities](https://socprime.com/active-threats/qlnx-linux-rat-uses-rootkit-and-pam-backdoor/)

---

## Summary

In early May 2026, Trend Micro researchers disclosed a previously undocumented Linux implant dubbed **Quasar Linux (QLNX)** — a full-featured Remote Access Trojan designed specifically to compromise developer workstations and harvest the credentials needed to launch downstream software supply chain attacks. Unlike the typical supply chain incident where a specific package or GitHub Action is poisoned, QLNX represents the upstream threat: it targets the developers who *publish* packages, stealing their npm, PyPI, and cloud credentials in order to become the attacker behind future poisoned releases.

The implant combines a 58-command RAT framework with a dual-layer rootkit (userland LD_PRELOAD + kernel-level eBPF), a PAM backdoor with a hardcoded master password, seven persistence mechanisms, and a credential harvester targeting every token and key found on a modern developer or DevOps workstation. After initial execution, QLNX copies itself into an anonymous in-memory file, re-executes from RAM, and deletes the original binary — leaving no on-disk dropper footprint. At disclosure, only four antivirus engines detected the implant, demonstrating its evasion maturity.

QLNX is supply chain infrastructure. A single compromised developer laptop can yield the npm publish token needed to trojanize a package with millions of weekly downloads, or the AWS IAM role credentials needed to backdoor a container image in ECR. The March 2026 LiteLLM compromise — where stolen credentials from one tool were used to trojanize a Python package with 3.4 million daily downloads — illustrates exactly the downstream impact QLNX is designed to enable.

---

## Compromised Artifacts

QLNX does not compromise a specific package; it compromises the developer. The implant installs itself as a persistent backdoor on the victim's Linux system and drops the following artifacts:

| Artifact | Role |
|----------|------|
| `Quaser-implant` | Main RAT binary (deleted from disk after in-memory re-exec) |
| `libsecurity_utils.so.1` | LD_PRELOAD userland rootkit shared object |
| `pam_security.so` | PAM backdoor module (loaded into dynamically linked processes) |
| `libpam_cache.so` | Secondary PAM credential harvester |
| `hide_src_39ZzHo.c` | Embedded C source for LD_PRELOAD rootkit (compiled on target) |
| `pam_src_51YyC3.c` | Embedded C source for PAM backdoor (compiled on target) |
| `pcs_a3kf9x.c` | Embedded C source for secondary rootkit component |

---

## How It Worked

### Delivery

The specific initial delivery vector was not fully disclosed in public reporting. Based on Trend Micro's description of a "development and DevOps environment" target profile and the credential targets present, likely delivery vectors include trojanized open-source packages (install-time dropper), malicious VS Code or IDE extensions, spear-phishing against developer accounts, or being dropped as a second stage by another supply chain compromise. Trend Micro's internal platform flagged "an unusual Linux implant with low detection" before a full investigation was conducted, suggesting opportunistic or targeted delivery rather than mass distribution.

### In-Memory Execution and Disk Wipe

Upon execution, QLNX immediately:
1. Copies itself into a memfd (anonymous in-memory file) using `memfd_create()`
2. Re-executes from the in-memory copy
3. Deletes the original binary from disk
4. Wipes forensic environment variables and modifies `/proc/self/exe` to reflect a spoofed process name mimicking a kernel thread (e.g., `[kworker/2:1]`)

This leaves zero on-disk dropper trace. Forensic tools using `ls`, `find`, or `stat` see nothing, because the rootkit is already active before those tools run.

### Rootkit Architecture (Dual-Layer)

QLNX dynamically compiles both rootkit components using `gcc` on the target system (the embedded C source is carried as string literals inside the binary):

**Layer 1 — Userland LD_PRELOAD rootkit (`libsecurity_utils.so.1`):**
- Deployed by writing the path to `/etc/ld.so.preload` — this causes it to be injected into *every* dynamically linked process on the system, including `ls`, `find`, `ps`, `stat`, and forensic tools
- Hooks `libc` functions to hide: the implant's files and directories, its PIDs from `ps` output, its network connections from `ss`/`netstat`
- Since the rootkit is loaded before any user-space tool runs, administrators using standard Linux utilities see a clean system

**Layer 2 — Kernel eBPF rootkit controller:**
- Manages BPF maps to conceal PIDs, file paths, and network ports at the kernel level
- Provides deeper stealth that persists even against tools compiled statically (which would bypass LD_PRELOAD)

### PAM Backdoor (Two Variants)

QLNX compiles and installs two PAM backdoor modules:

- **`pam_security.so` (primary):** Hooks PAM authentication events system-wide. Harvests plaintext passwords from every local login, `sudo`, and SSH authentication. Contains a hardcoded master password that grants authentication to any account regardless of the actual password on file. Logs outbound SSH session data to capture lateral movement credentials.
- **`libpam_cache.so` (secondary):** Loads into dynamically linked processes to extract service name, username, and authentication token at the moment of authentication — capturing credentials even from password managers and SSH agents.

### Credential Harvesting Module

On command from the C2, QLNX runs a comprehensive credential sweep across the victim's filesystem:

| Category | Targets |
|----------|---------|
| Package registry tokens | `~/.npmrc` (npm tokens), `~/.pypirc` (PyPI API keys), `~/.cargo/credentials.toml` |
| Cloud credentials | `~/.aws/credentials`, `~/.aws/config`, all `AWS_*` env vars; `~/.config/gcloud/`; `~/.azure/`; `~/.vault-token` |
| Kubernetes | `~/.kube/config`, `/var/run/secrets/kubernetes.io/serviceaccount/token` |
| Container / CI | `~/.docker/config.json`, `.gitlab-ci.yml`, `Jenkinsfile`, `terraform.tfvars`, `terraform.tfstate` |
| Source control | `~/.git-credentials`, GitHub CLI token (`~/.config/gh/hosts.yml`), SSH private keys (`~/.ssh/id_*`) |
| Secrets files | Recursive sweep of `.env`, `.env.local`, `.env.production`, `.env.staging` from home dirs, `/opt`, `/app`, `/srv` |
| PAM live capture | Plaintext passwords captured at every authentication event after PAM backdoor is installed |
| Terraform | Cloud provider credentials embedded in `.tfvars` and `.tfstate` files |

Machine fingerprinting is computed as a SHA hash of `/etc/machine-id`, MAC addresses, CPU information, and hostname — used as a unique victim identifier in the C2 registration packet.

### C2 Communication

QLNX maintains persistent contact with its C2 over either custom TCP/TLS or HTTPS channels. The RAT supports 58 registered commands covering interactive shell access, file and process management, rootkit management (deploy/unload layers), credential harvest triggers, network tunneling, and keylogger start/stop. Specific C2 domains and IP addresses were not released in public reporting; consult the Trend Micro full disclosure for the IoC appendix.

### Persistence (Seven Mechanisms)

The implant ensures it survives reboots and process kills through seven independent persistence channels, any one of which is sufficient to reload the full implant:

1. **`/etc/ld.so.preload`** — rootkit loads into every new process
2. **systemd service unit** — named to mimic a legitimate system service
3. **crontab entry** — per-user and/or system crontab
4. **init.d script** — traditional SysV init for older distributions
5. **XDG autostart** — `.desktop` entry in `~/.config/autostart/`
6. **`.bashrc` / `.profile` injection** — re-executes implant on every interactive shell
7. **In-memory re-exec** — running implant respawns itself if the memfd process is killed

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Unknown (pre-May 2026) | QLNX developed and deployed; exact first-seen date not disclosed |
| ~May 5, 2026 | Trend Micro internal platform flags unusual Linux implant with low AV detection; investigation begins |
| May 5, 2026 | Trend Micro publishes full technical disclosure of QLNX; BleepingComputer, SecurityWeek follow |
| May 5, 2026 | At disclosure, only 4/70+ AV engines detect the implant on VirusTotal |

---

## Detection

```bash
# ── ROOTKIT INDICATORS ──────────────────────────────────────────────────────

# Check /etc/ld.so.preload for unexpected shared library entries
# Legitimate systems should have no entries, or only known intentional ones
cat /etc/ld.so.preload

# Look for QLNX's known malicious shared library filenames
find / -name "libsecurity_utils.so*" -o -name "pam_security.so" \
        -o -name "libpam_cache.so" 2>/dev/null

# Look for embedded C source files QLNX compiles on the target
find / -name "hide_src_*.c" -o -name "pam_src_*.c" -o -name "pcs_*.c" 2>/dev/null

# Check for recently modified PAM configuration (rootkit modifies /etc/pam.d/)
find /etc/pam.d -newer /etc/passwd -ls 2>/dev/null

# Check PAM modules directory for unknown shared objects
find /lib/security /lib64/security /usr/lib/security -name "*.so" \
  ! -name "pam_unix.so" ! -name "pam_env.so" ! -name "pam_permit.so" \
  ! -name "pam_deny.so" ! -name "pam_warn.so" ! -name "pam_systemd.so" \
  2>/dev/null | sort

# ── PERSISTENCE INDICATORS ──────────────────────────────────────────────────

# Check crontab for unexpected entries (both user and system)
crontab -l 2>/dev/null
cat /etc/cron.d/* /etc/crontab 2>/dev/null

# Check systemd services for recently-added suspicious units
find /etc/systemd/system /usr/lib/systemd/system -name "*.service" \
  -newer /etc/passwd -ls 2>/dev/null

# Check XDG autostart entries
find ~/.config/autostart /etc/xdg/autostart -name "*.desktop" 2>/dev/null | xargs grep -l "Exec" 2>/dev/null

# Check for .bashrc / .profile injection (look for unusual Exec-like lines)
grep -E "(curl|wget|bash|python|perl|ruby|/tmp|/dev/shm)" \
  ~/.bashrc ~/.profile ~/.bash_profile 2>/dev/null

# ── PROCESS INDICATORS ──────────────────────────────────────────────────────

# Look for processes with names mimicking kernel threads but owned by non-root
# Legitimate kworker processes are owned by root
ps aux | grep -E "\[kworker|\[kswap|\[ksoftirq" | grep -v "^root"

# Check for processes running from memfd or deleted binaries
ls -la /proc/*/exe 2>/dev/null | grep "(deleted)"

# ── NETWORK INDICATORS ───────────────────────────────────────────────────────

# List all established TCP connections (note: rootkit hides its own, so prefer
# running from a live boot or external network capture)
ss -tnp | grep ESTABLISHED

# ── CREDENTIAL EXPOSURE CHECK ────────────────────────────────────────────────

# Check npm token exposure
grep -i "authToken\|password\|_auth" ~/.npmrc 2>/dev/null

# Check PyPI API key exposure
cat ~/.pypirc 2>/dev/null

# Verify AWS credential files haven't been accessed recently
stat ~/.aws/credentials 2>/dev/null

# ── STATIC ANALYSIS (if a copy of the binary is available) ───────────────────

# Look for QLNX's embedded C source strings in suspicious binaries
strings /path/to/suspicious_binary | grep -E "ld.so.preload|pam_src|hide_src|memfd_create|machine-id"
```

> **Important caveat:** Because QLNX's LD_PRELOAD rootkit injects into every dynamically linked process — including `ls`, `find`, `ps`, and `ss` — the detection commands above may return clean results on a live infected system. For reliable detection, boot from external media (live USB), mount the infected disk, and run checks from the clean environment, or use statically compiled forensic tools (e.g., `busybox` static build) that are not affected by LD_PRELOAD.

---

## Remediation

1. **Do not trust standard Linux tools on the live system.** The LD_PRELOAD rootkit and eBPF component actively hide files, processes, and connections. Use a live-boot environment to investigate.

2. **Mount the suspicious disk from a clean host** and manually inspect `/etc/ld.so.preload`, `/etc/pam.d/`, `/lib/security/`, crontabs, and systemd unit files.

3. **Remove all persistence mechanisms:**
   ```bash
   # (run from live boot, with disk mounted at /mnt)
   > /mnt/etc/ld.so.preload
   find /mnt/lib/security /mnt/lib64/security -name "pam_security.so" \
     -o -name "libpam_cache.so" -delete
   # Remove crontab entries, systemd units, XDG autostart, .bashrc injections manually
   ```

4. **Rotate all credentials that may have been exposed:**
   - npm publish tokens (via `npm token revoke` and re-generate)
   - PyPI API keys (revoke in PyPI account settings)
   - AWS access keys and any EC2 instance roles associated with the machine
   - Kubernetes service account tokens and kubeconfig certificates
   - GitHub personal access tokens and SSH keys
   - GCP service account keys
   - Docker Hub and container registry credentials
   - Any `.env` file values on the compromised system

5. **Audit npm and PyPI publish logs** for any unauthorized package releases in the window between estimated compromise date and detection. Check `npm access ls-packages` and PyPI release history for unexpected versions.

6. **Audit cloud provider logs** (CloudTrail, GCP Audit Logs) for credential use from unexpected IP addresses or regions during the suspected compromise window.

7. **Reimage the affected system** — given QLNX's seven persistence mechanisms and kernel-level eBPF rootkit, a clean reinstall is the only reliable remediation. Do not attempt to patch a live infected system.

---

## Lessons Learned

- Supply chain attackers are increasingly targeting the *publisher* rather than the *repository* — compromising developer workstations to steal publish credentials is more scalable than exploiting package registries directly
- A dual LD_PRELOAD + eBPF rootkit renders standard Linux forensics unreliable on a live system; organizations should establish live-boot forensic procedures for developer endpoint incidents
- Seven independent persistence mechanisms mean that partial remediation (removing the cron entry but missing the `.bashrc` injection) leaves the system re-infected within seconds
- PAM backdoors with master passwords create a permanent authentication bypass that survives password rotation — any system where QLNX installed its PAM module must be assumed to have a persistent backdoor regardless of whether the RAT process is running
- The credential harvester explicitly targets npm tokens and PyPI API keys — the direct keys to the package publishing kingdom — confirming that supply chain compromise is QLNX's primary downstream objective, not just incidental to credential theft
- With only 4/70+ AV detections at disclosure, signature-based endpoint protection provides almost no coverage; behavioral detection (unusual gcc compilation, unexpected LD_PRELOAD modifications, memfd process re-exec) is the only reliable signal

---

## Related Incidents

- [litellm PyPI — Credential Stealer Hidden in Wheel](./2026-03-litellm-pypi-stealer.md) — The downstream impact QLNX is designed to enable: stolen credentials used to trojanize a 3.4M daily-download Python package
- [IoliteLabs Solidity VSCode Extensions — Dormant Publisher Backdoor](./2026-03-iolitelabs-vscode-backdoor.md) — Related vector: IDE extension as developer workstation implant
- [GlassWorm Native — Zig Dropper IDE Mass-Infection](./2026-04-glassworm-zig-dropper.md) — Cross-IDE native dropper targeting developer machines at scale
- [Checkmarx KICS GitHub Action Compromised](./2026-03-checkmarx-kics-action.md) — CI/CD credential theft, the pipeline equivalent of QLNX's workstation targeting
