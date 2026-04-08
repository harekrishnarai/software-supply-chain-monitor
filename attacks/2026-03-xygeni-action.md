# xygeni-action C2 Reverse Shell via Tag Poisoning

**Date:** March 2026
**Ecosystem:** GitHub Actions
**Severity:** Critical
**Type:** Tag poisoning / C2 reverse shell / Credential theft
**Sources:**
- [StepSecurity — xygeni-action Compromised: C2 Reverse Shell Backdoor Injected via Tag Poisoning](https://www.stepsecurity.io/blog/xygeni-action-compromised-c2-reverse-shell-backdoor-injected-via-tag-poisoning)

---

## Summary

On March 3, 2026, the official Xygeni GitHub Action (`xygeni/xygeni-action`) was compromised when an attacker using stolen maintainer credentials injected a full C2 reverse shell backdoor. Three pull requests carrying the malicious payload were opened from compromised accounts (two human maintainer accounts and one GitHub App) within a 23-minute window. Though all three PRs were closed without merging, the attacker simultaneously moved the mutable `v5` tag to point at the backdoored commit.

Any CI pipeline referencing `xygeni/xygeni-action@v5` was silently executing a full interactive C2 implant for 7 days (March 3–10) without any change to their workflow YAML files. The backdoor registered with a C2 server, polled for arbitrary commands every 2–7 seconds for 3 minutes, and exfiltrated compressed results — giving the attacker interactive access to every affected CI runner.

---

## Compromised Artifacts

| Action | Affected Versions | Safe Alternative |
|--------|------------------|-----------------|
| `xygeni/xygeni-action` | `@v5` (tag moved to commit `4bf1d4e`) | `@v6.4.0` or SHA `13c6ed2797df7d85749864e2cbcf09c893f43b23` |

Exposure window: 2026-03-03 ~10:49 UTC to 2026-03-10 (tag removed)

---

## How It Worked

### Entry Point: Compromised Maintainer Credentials

Three different identities pushed the same payload within 23 minutes, strongly indicating credential theft rather than insider action:
- PR #46 (10:22 UTC) — opened by `nico-car` (maintainer since 2023), signed with `felix.carnicero@xygeni.io`
- PR #47 (10:41 UTC) — opened again by `nico-car`
- PR #48 (10:45 UTC) — opened by `xygeni-onboarding-app-dev[bot]` (a Xygeni-owned GitHub App); `nico-car` approved with "Looks good, telemetry step verified"

All three PRs were closed without merging. The real attack was moving the `v5` tag to commit `4bf1d4e` from PR #48's branch.

### Payload Mechanics: Interactive C2 Implant

The backdoor was inserted as a step called "Report Scanner Telemetry" in `action.yml`, positioned between scanner installation and the actual scan, and run in the background (`&`) so the legitimate scan proceeded normally.

The implant:
1. **Registered** with C2 at `security-verify.91.214.78.178.nip.io` — sending hostname, username, and OS version
2. **Polled** for arbitrary commands (`eval "$_d"`) every 2–7 seconds for 180 seconds
3. **Exfiltrated** compressed, base64-encoded command output back to the C2
4. **Skipped TLS verification** (`curl -k`) and used an authentication header (`X-B: sL5x#9kR!vQ2$mN7`) to prevent unauthorized C2 access
5. Logged an innocent-looking debug line: `"::debug::Telemetry reported: $_xv"`

During the 3-minute polling window, the attacker could execute any command on the CI runner — stealing `GITHUB_TOKEN`, `XYGENI_TOKEN`, source code, build artifacts, or pivoting to other systems.

### Tag Poisoning Mechanics

The `v5` tag is a mutable lightweight tag. Moving it to a different commit requires only write access — no PR, no review, no visible change in any downstream user's workflow YAML. GitHub Actions users referencing `@v5` silently started running the backdoor. Git history on `main` showed nothing because the malicious commit lived only as a tag reference.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-03-03 10:21 | Malicious commit `ceead6d` created |
| 2026-03-03 10:22 | PR #46 opened by `nico-car` |
| 2026-03-03 10:29 | PR #46 closed without merging |
| 2026-03-03 10:41 | PR #47 opened by `nico-car` |
| 2026-03-03 10:44 | PR #47 closed without merging |
| 2026-03-03 10:45 | PR #48 opened by `xygeni-onboarding-app-dev[bot]` |
| 2026-03-03 10:49 | PR #48 closed; `v5` tag silently moved to backdoored commit `4bf1d4e` |
| 2026-03-03 12:23–12:25 | Xygeni team deletes all workflow files from repository |
| 2026-03-09 08:49 | Xygeni releases v6.4.0 with SHA pinning guidance; `v5` tag **still compromised** |
| 2026-03-09 19:57 | Issue #54 opened publicly calling out the malicious code |
| 2026-03-10 10:20 | Xygeni publishes incident report, removes `v5` tag, rotates all tokens, enables release immutability |

---

## Detection

```bash
# Check if your workflow references the compromised v5 tag
grep -r "xygeni/xygeni-action" .github/workflows/ | grep "@v5"

# Check for the known compromised commit SHA
grep -r "4bf1d4e" .github/workflows/

# If using Harden-Runner, check for outbound calls to the C2 IP
grep "91.214.78.178" /path/to/harden-runner-logs

# Audit recent workflow run logs for the telemetry step
# Look for "Report Scanner Telemetry" or the nip.io domain pattern
grep -r "security-verify\|nip\.io\|91\.214\.78\.178" ~/.github/workflow-logs/

# Check for the C2 auth header pattern in captured network traffic
grep "X-B:" /var/log/network.log 2>/dev/null
```

---

## Remediation

1. **Immediately update** to `xygeni/xygeni-action@v6.4.0` or pin to SHA `13c6ed2797df7d85749864e2cbcf09c893f43b23`.
2. **Rotate all secrets** accessible to any workflow run that executed `xygeni/xygeni-action@v5` between March 3–10, 2026 — including `GITHUB_TOKEN`, `XYGENI_TOKEN`, and any cloud credentials.
3. **Block C2 IP** `91.214.78.178` at the firewall level.
4. **Audit all GitHub Actions** for mutable tag references (`@v1`, `@v5`, etc.) and migrate to full commit SHA pinning.
5. **Enable tag protection rules** to prevent unauthorized tag creation or modification on repositories containing published actions.

---

## Lessons Learned

- Mutable version shortcut tags (`@v5`) are the single most dangerous design pattern in GitHub Actions — they allow silent code swaps with no visible change to any consumer.
- Compromised credentials (not insider threats) can simultaneously abuse multiple identities — detecting the pattern requires monitoring for multiple accounts pushing the same payload in rapid succession.
- Three failed PRs closing without merging is not equivalent to "attack failed" — the tag move is the real attack vector and requires zero PR merges.
- Security vendors' own actions carry the same supply chain risk as any community action; "trusting the vendor" is not sufficient.
- Mutable tag attacks were previously documented in tj-actions (March 2025) and reviewdog — yet the pattern continues to be widely exploited because most teams still reference actions by mutable version tags.

---

## Related Incidents

- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — Same tag poisoning technique used to compromise tj-actions/changed-files
- [./2026-03-trivy-github-actions.md](./2026-03-trivy-github-actions.md) — Trivy GitHub Actions tag compromise (same month, different attacker)
