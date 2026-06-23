# Atomic Arch — 400+ AUR Packages Hijacked via Orphan Adoption, Rust Credential Stealer with eBPF Persistence

**Date:** June 2026
**Ecosystem:** AUR (Arch User Repository) / Linux
**Severity:** Critical
**Type:** Package Ownership Hijacking / Malicious Build Script / Credential Stealer / eBPF Rootkit
**Sources:**
- [StepSecurity — 400+ AUR Packages Hijacked: What the "Atomic Arch" Campaign Means for Supply-Chain Security](https://www.stepsecurity.io/blog/400-aur-packages-hijacked-atomic-arch-campaign)

---

## Summary

On June 11, 2026, security researchers (Sonatype) and the Arch Linux community disclosed a large-scale supply chain attack against the Arch User Repository (AUR) dubbed "Atomic Arch." Attackers systematically adopted orphaned AUR packages — packages abandoned by their original maintainers — and injected malicious build logic into their PKGBUILDs. Follow-up community analysis revealed more than 400 packages had been modified. The attack is notable for exploiting AUR's legitimate ownership-transfer mechanism rather than typosquatting or account compromise.

When users installed or updated any of the hijacked AUR packages, the PKGBUILD silently executed `npm install atomic-lockfile` (and variants such as `js-digest`), pulling in a malicious Node.js package that delivered a Rust-compiled credential stealer targeting developer and CI infrastructure. On privileged systems, the malware also deployed eBPF-based persistence to hide its processes and file activity, making detection and cleanup significantly harder.

The Atomic Arch playbook is a direct application of a technique increasingly seen in npm, PyPI, and GitHub Actions: rather than creating lookalike packages (typosquatting), attackers inherit the user base and reputation of trusted but abandoned packages in a single move. This ownership-hijacking approach bypasses most package-name-based detection.

---

## Compromised Artifacts

| Artifact | Type | Notes |
|----------|------|-------|
| 400+ orphaned AUR packages | PKGBUILD scripts | Systematically adopted and modified |
| `atomic-lockfile` (npm) | Malicious Node package | Primary dropper, disguised as a lockfile utility |
| `js-digest` (npm) | Malicious Node package | Variant dropper used in some packages |

The exact list of compromised AUR package names was not published due to the scale; affected packages span common developer tooling categories. All shared the pattern of injecting `npm install atomic-lockfile` (or variant) into the PKGBUILD's `prepare()` or `build()` function.

---

## How It Worked

### Entry Point: AUR Orphan Adoption

The AUR allows any user to request adoption of a package whose maintainer has become inactive. This is a legitimate maintenance mechanism. Attackers exploited it systematically:

1. **Identified orphaned packages** with an existing user base and installation history
2. **Submitted adoption requests** through normal AUR channels, inheriting the package name, version history, and reputation
3. **Modified PKGBUILDs** to inject a single additional install step: `npm install atomic-lockfile` (or `js-digest`)

### Stage 2: Malicious npm Package Execution

When a user ran `yay -Syu` or manually installed any compromised package, the PKGBUILD downloaded and executed `atomic-lockfile` from npm. This package contained a Rust-compiled binary credential stealer that ran with whatever privileges the build process had (often the user's own, sometimes root via `makepkg --asroot`).

### Stage 3: Credential Stealer Payload

The Rust-compiled payload targeted high-value assets on developer and CI hosts:
- Browser cookies and session tokens (SSO, cloud consoles, SaaS applications)
- SSH private keys and `known_hosts`
- GitHub tokens, npm tokens, and other developer tokens
- Slack, Discord, and Teams session tokens
- Docker/Podman credentials and cloud access keys (AWS, GCP, Azure)

### Stage 4: eBPF Persistence (Elevated Privilege)

On systems where the malware executed with root or elevated privileges, it deployed eBPF-based kernel persistence to:
- Hide its own processes and files from standard userspace tools (`ps`, `ls`, `netstat`)
- Intercept credentials captured at the kernel level
- Survive standard malware scanning and manual inspection

Systems where this stage succeeded must be treated as fully compromised — a malware scan is insufficient. Full rebuild from trusted media is required.

---

## Timeline

| Date (UTC) | Event |
|-----------|-------|
| Before Jun 11 | Attackers systematically adopt orphaned AUR packages over an unknown period |
| Jun 11, 2026 | Sonatype researchers and Arch Linux community disclose the "Atomic Arch" campaign |
| Jun 11–12 | Community analysis confirms 400+ packages affected |
| Jun 12, 2026 | StepSecurity publishes analysis; Arch Linux Trusted User team begins mass-reverting hijacked packages |

---

## Detection

```bash
# Check AUR helper install logs for atomic-lockfile or js-digest
grep -r "atomic-lockfile\|js-digest" ~/.cache/yay/ /var/log/pacman.log 2>/dev/null

# Check if atomic-lockfile is installed globally
npm ls -g atomic-lockfile 2>/dev/null
npm ls -g js-digest 2>/dev/null

# Check for unexpected node_modules in system paths
find /usr /opt /home -name "atomic-lockfile" -type d 2>/dev/null

# Check for suspicious eBPF programs loaded (requires root)
bpftool prog list 2>/dev/null | grep -v "kprobe\|tracepoint\|xdp\|cgroup\|perf"

# Look for hidden processes (compare procfs vs ps output)
ls /proc | grep -E '^[0-9]+$' | sort -n > /tmp/proc_pids.txt
ps aux | awk 'NR>1{print $2}' | sort -n > /tmp/ps_pids.txt
diff /tmp/proc_pids.txt /tmp/ps_pids.txt

# Check for outbound connections from npm/node during recent installs
# Review systemd journal for unexpected network activity during package builds
journalctl --since="2026-06-11" | grep -E "node|npm|atomic"

# Scan for the Rust credential stealer binary (check temp/home dirs)
find /tmp /home -name "*.so" -newer /etc/hostname 2>/dev/null
find /tmp /home -executable -type f -newer /etc/hostname 2>/dev/null | xargs file 2>/dev/null | grep "ELF.*Rust"
```

---

## Remediation

1. **Identify potentially affected hosts**: Enumerate all Arch Linux and Arch-derivative systems (EndeavourOS, Manjaro with AUR enabled, WSL2 Arch) in your environment
2. **Review AUR install history** since early June 2026: `grep "installed\|upgraded" /var/log/pacman.log | grep -v "core\|extra\|multilib" | tail -200`
3. **Assume full credential compromise** on any host where `atomic-lockfile` or `js-digest` may have executed — rotate immediately: SSH keys, GitHub tokens, npm tokens, cloud API keys, Vault tokens, browser sessions, SSO sessions, Docker registry credentials
4. **Rebuild compromised systems from scratch**: Do not attempt to clean eBPF-persisted malware; rebuild from trusted installation media and re-provision with fresh credentials
5. **Add AUR adoption monitoring**: Subscribe to AUR package alerts for packages you depend on, and pin specific PKGBUILD checksums in CI
6. **Restrict AUR usage in CI/CD**: If self-hosted runners use Arch, prohibit AUR package installs in CI pipelines or replace with official repository packages only

---

## Lessons Learned

- **Ownership hijacking beats typosquatting**: Inheriting an abandoned trusted package is more effective than creating a lookalike — it comes with an existing user base and no suspicious new name.
- **Build scripts are an execution environment**: PKGBUILDs, postinstall hooks, GitHub Actions workflows, and Dockerfiles all execute with powerful privileges during installation. They deserve the same security scrutiny as application code.
- **The AUR's community trust model is a structural risk**: Any model where package ownership can transfer without cryptographic verification of publisher identity creates an attack surface. The same risk exists in Homebrew (via formula and tap adoption), npm (via maintainer additions), and GitHub Actions (via org membership changes).
- **eBPF rootkits erase the value of scanning**: When the malware runs as root, standard forensic tools can no longer be trusted. The only safe recovery path is full system rebuild.
- **Developer credential theft has organizational blast radius**: An attacker who compromises a single developer's Arch laptop can pivot into GitHub, npm, cloud accounts, and CI/CD pipelines that power non-Arch production systems.

---

## Related Incidents

- [2026-05-megalodon-github-actions.md](./2026-05-megalodon-github-actions.md) — GitHub Actions version tag poisoning to achieve similar repo-wide credential theft
- [2026-04-prt-scan-github-actions.md](./2026-04-prt-scan-github-actions.md) — Mass PR campaign exploiting package manager hooks for CI secret theft
- [2025-09-ghostaction-campaign.md](./2025-09-ghostaction-campaign.md) — GhostAction: 327 CI/CD accounts compromised, 3,325 secrets exfiltrated
