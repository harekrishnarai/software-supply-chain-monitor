# SleeperGem — Coordinated RubyGems Compromise via Dormant Account Takeover, Forgejo C2, Persistent Daemon Backdoor

**Date:** July 2026
**Ecosystem:** RubyGems
**Severity:** High
**Type:** Dormant Account Takeover / Loader / Persistent Daemon / Privilege Escalation
**Sources:**
- [StepSecurity — SleeperGem: Compromised git_credential_manager, Dendreo, and fastlane RubyGems Drop a Persistent Backdoor](https://www.stepsecurity.io/blog/sleepergem-compromised-rubygems-drop-persistent-backdoor)
- [Aikido — SleeperGem: RubyGems supply chain attack targets dormant maintainer accounts](https://www.aikido.dev/blog/sleepergem-rubygems-supply-chain-attack)

---

## Summary

Between July 18 and July 19, 2026, an attacker published malicious versions of three RubyGems to RubyGems.org in a coordinated campaign researchers named **SleeperGem**. Two of the targeted gems had been dormant for years — `Dendreo` had not shipped a release since 2020, and `fastlane-plugin-run_tests_firebase_testlab` had no activity since before 2026. This pattern — taking over long-dormant publisher accounts and suddenly shipping new versions — is the defining characteristic of the SleeperGem campaign.

None of the malicious releases have a matching git commit or release tag in their source repositories; they were published directly to the registry. Each gem is a loader: rather than carrying a complete payload, it downloads a second stage (`deploy.sh` + a native binary) from an attacker-controlled **Forgejo** instance (`git.disroot.org`). The loader skips execution on CI systems — checking approximately 30 CI environment variables — deliberately targeting **developer laptops** where SSH keys, cloud credentials, and browser secrets live.

On developer machines, the downloaded script drops a native daemon into `~/.local/share/gcm/`, installs dual persistence (systemd user service + cron) under the innocuous name `git-credential-manager`, and probes for passwordless sudo access. If successful, it plants a **setuid root shell at `/usr/local/sbin/ping6`** — disguised as a network utility — providing a permanent privileged backdoor. StepSecurity confirmed all three packages share the same C2 infrastructure, establishing this as a single coordinated campaign.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Notes |
|---------|---------------------|-------|
| `git_credential_manager` | 2.8.0, 2.8.1, 2.8.2, 2.8.3 | Impersonates official Microsoft Git Credential Manager; 2.8.2 stages only (trigger commented out), 2.8.3 fully detonates |
| `Dendreo` | 1.1.3, 1.1.4 | Published July 18 after gem was dormant since 2020 |
| `fastlane-plugin-run_tests_firebase_testlab` | 0.3.2 | Published July 19; no matching GitHub tag; upstream stops at v0.3.1 |

---

## How It Worked

### Entry Point: Dormant Account Takeover

The attacker took over publisher accounts associated with gems that had not shipped new releases in years. `Dendreo` published two new versions on the same day after nearly six years of dormancy; `fastlane-plugin-run_tests_firebase_testlab` received a release years after its previous one. Both patterns are immediately suspicious when compared against RubyGems publish timestamps vs. source repository release tags.

### The Loader Pattern

Each malicious gem contains minimal logic — it is a loader, not a full payload. For `git_credential_manager` 2.8.2 and later, the malicious code runs on `require` (not just at install time), making `--no-post-install-message` flags ineffective. On require, the gem spawns a child Ruby process that:

1. Downloads `deploy.sh` from `git.disroot.org` (under a path impersonating an official Git ecosystem account)
2. Downloads a native binary (named identically to the impersonated tool: `git-credential-manager`)
3. Downloads use Ruby's built-in HTTP client with **certificate verification disabled** and a `User-Agent: Git` header to blend with normal Git traffic

Version 2.8.2 stages the attack (downloads payload, does not execute it). Version 2.8.3 removes the safety line and runs the full chain.

### CI Evasion

Before any malicious action, the loader checks approximately 30 CI environment variables:

```ruby
# (abbreviated) — exits immediately if any are set
ENV.key?('GITHUB_ACTIONS') || ENV.key?('GITLAB_CI') || ENV.key?('CIRCLECI') ||
ENV.key?('TRAVIS') || ENV.key?('JENKINS_URL') || ENV.key?('BUILDKITE') # etc.
```

This means a standard CI pipeline install shows no malicious behavior. StepSecurity had to strip CI environment variables to observe the full kill chain.

### Payload Execution (deploy.sh)

The downloaded `deploy.sh` script performs the following steps in order (captured verbatim from Harden-Runner process tree):

```bash
ruby <gem_path>/git_credential_manager-2.8.3/bin/install
sh -c /bin/sh '<gem_path>/git_credential_manager-2.8.3/package/deploy.sh' 2>&1 >/dev/null
cp <gem_path>/.../package/git-credential-manager $HOME/.local/share/gcm/git-credential-manager
chmod +x $HOME/.local/share/gcm/git-credential-manager
$HOME/.local/share/gcm/git-credential-manager --daemon
$HOME/.local/share/gcm/git-credential-manager install --method=systemd --service-name=git-credential-manager
systemctl --user daemon-reload
$HOME/.local/share/gcm/git-credential-manager install --method=cron --service-name=git-credential-manager
getent group sudo
```

### Persistence and Privilege Escalation

The daemon is installed under the service name `git-credential-manager` to blend with legitimate Git tooling:

- **systemd:** A user unit `git-credential-manager.service` with `ExecStart` pointing at the dropped daemon at `~/.local/share/gcm/git-credential-manager`
- **Cron:** A matching cron entry running the same binary

If the user can run `sudo` without a password (common on developer laptops configured for convenience), `deploy.sh` re-runs as root and plants a **setuid root shell** at `/usr/local/sbin/ping6` with permissions `6777` — disguised as a standard networking utility. This provides a permanent root escalation path independent of the daemon.

The credential harvesting and exfiltration logic lives inside the native binary itself — StepSecurity's external footprint analysis only confirms the binary is downloaded and launched; what it does on the machine is a separate binary analysis of the dropped artifact.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| 2026-07-18 | `Dendreo@1.1.3` and `Dendreo@1.1.4` published to RubyGems (first releases since 2020) |
| 2026-07-18 | `git_credential_manager@2.8.0`, `2.8.1`, `2.8.2` published (staging versions) |
| 2026-07-19 | `git_credential_manager@2.8.3` published (full detonation); `fastlane-plugin-run_tests_firebase_testlab@0.3.2` published |
| 2026-07-19 | Aikido Security research team identifies and names the SleeperGem campaign |
| 2026-07-19 | StepSecurity runs all compromised gem versions under Harden-Runner; confirms C2 contact to `git.disroot.org`, full kill chain captured in process tree |
| 2026-07-19–20 | StepSecurity publishes full analysis |

---

## Detection

```bash
# Check Gemfile.lock for compromised versions
grep -RniE 'git_credential_manager \((2\.8\.[0-3])\)|Dendreo \(1\.1\.[34]\)|run_tests_firebase_testlab \(0\.3\.2\)' \
  --include=Gemfile.lock .

# Check for installed malicious gems
gem list | grep -E "git_credential_manager|dendreo|run_tests_firebase_testlab"

# Check for the dropped daemon
ls -la ~/.local/share/gcm/git-credential-manager 2>/dev/null
ls -la ~/.local/share/gcm/.env 2>/dev/null

# Check for systemd persistence
systemctl --user status git-credential-manager 2>/dev/null
systemctl --user list-unit-files | grep git-credential

# Check for cron persistence
crontab -l | grep -E "gcm|git-credential-manager"

# Check for the privilege escalation backdoor (setuid shell at ping6 path)
ls -la /usr/local/sbin/ping6 2>/dev/null
# Suspicious if permissions are 6777 or file is unexpectedly large
stat /usr/local/sbin/ping6 2>/dev/null | grep -E "Access|Size"

# Check for C2 connections in network logs
grep -Ei "git\.disroot\.org" /var/log/proxy.log /var/log/nginx/access.log 2>/dev/null

# Check for the loader in gem source (C2 hardcoded in gem files)
find $(gem environment gemdir)/gems -path "*/git_credential_manager*" \
  -name "*.rb" -exec grep -l "disroot\|deploy\.sh" {} \; 2>/dev/null

# Verify suspicious Ruby process spawning a child on require
strace -p $(pgrep ruby) -e trace=execve 2>/dev/null | grep -E "deploy|git-credential"
```

---

## Remediation

1. Uninstall the malicious gems: `gem uninstall git_credential_manager -v 2.8.3` (repeat for all affected versions), `gem uninstall dendreo`, `gem uninstall fastlane-plugin-run_tests_firebase_testlab -v 0.3.2`
2. Stop and remove the systemd persistence: `systemctl --user stop git-credential-manager; systemctl --user disable git-credential-manager; systemctl --user daemon-reload`
3. Remove the cron entry: `crontab -l | grep -v "gcm\|git-credential-manager" | crontab -`
4. Delete the daemon and its files: `rm -rf ~/.local/share/gcm/`
5. Inspect and remove the privilege escalation backdoor: `ls -la /usr/local/sbin/ping6` — if permissions are `6777` or the file is unexpectedly large, remove it: `sudo rm /usr/local/sbin/ping6`
6. **Rotate all credentials** on any machine where these gems were required, including SSH keys, cloud provider credentials (`~/.aws`, `~/.config/gcloud`, Azure), GitHub tokens, browser-stored passwords, and wallet keys
7. Review outbound network logs for connections to `git.disroot.org` to determine scope

---

## Lessons Learned

- **Dormant gem accounts are soft targets:** A maintainer account that hasn't logged in for years is likely to have weak credentials, no MFA, or a stale email address the attacker can take over via account recovery. RubyGems.org and other registries should enforce MFA on all accounts or flag unusual activity on dormant accounts.
- **Registry-only releases without source tags are an instant red flag:** All three malicious gems were published to RubyGems with no corresponding git tag in the source repository. Automated monitoring of registry publish timestamps vs. source release tags would have flagged all three immediately.
- **CI evasion shifts risk entirely to developer machines:** When malware specifically targets developer laptops (by exiting on 30+ CI env vars), CI-based controls provide no protection. Fleet-wide file-integrity monitoring and endpoint detection are required to catch this class of attack.
- **Two-stage loading with remote code execution makes static analysis moot:** The gem itself contains only a downloader; the real payload is fetched at runtime from an attacker-controlled server. Registry scanning of the gem content finds nothing malicious. Behavioral detection (egress monitoring, process tree analysis) is the only effective control.
- **`git.disroot.org` is a legitimate service:** The C2 uses Disroot, a legitimate privacy-focused hosting service, making domain reputation-based blocking ineffective. Egress allowlisting based on the workflow's normal network baseline is required.

---

## Related Incidents

- [./2026-03-iolitelabs-vscode-backdoor.md](./2026-03-iolitelabs-vscode-backdoor.md) — Similar dormant publisher account takeover pattern (8-year dormant account)
- [./2026-04-pgserve-npm-canistersprawl.md](./2026-04-pgserve-npm-canistersprawl.md) — Similar loader pattern with remote second-stage download
- [./2026-06-aur-atomic-arch.md](./2026-06-aur-atomic-arch.md) — Orphaned package adoption pattern exploiting dormant/abandoned packages
