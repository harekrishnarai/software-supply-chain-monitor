# tj-actions/changed-files Supply Chain Attack

**Date:** March 2025
**Ecosystem:** GitHub Actions
**Severity:** Critical
**Type:** GitHub Action compromise / CI/CD secret leakage
**CVE:** CVE-2025-30066
**Sources:**
- [aikido.dev — Get the TL;DR: tj-actions/changed-files Supply Chain Attack](https://www.aikido.dev/blog/get-the-tl-dr-tj-actions-changed-files-supply-chain-attack)
- [stepsecurity.io — Harden-Runner detection: tj-actions/changed-files action is compromised](https://www.stepsecurity.io/blog)

---

## Summary

The `tj-actions/changed-files` GitHub Action — used in over **23,000 repositories** — was compromised in mid-March 2025. Attackers injected malicious code that caused CI/CD secrets to be printed into workflow run logs. All tagged versions were simultaneously modified, making version-tag pinning completely ineffective. Public repositories were most at risk because their workflow logs are publicly readable, but private repositories also faced exposure of secrets written to logs.

The attack was first reported by Step Security, assigned CVE-2025-30066, and resolved within roughly 24 hours of public disclosure on March 14–15, 2025.

---

## Compromised Artifacts

| Action | Affected Versions |
|--------|------------------|
| `tj-actions/changed-files` | All tagged versions (all tags force-modified) |

---

## How It Worked

### Initial Compromise

The attacker compromised a **GitHub Personal Access Token (PAT)** belonging to the `tj-actions-bot` account — a service account used for automated operations on the repository. With this token, the attacker was able to:

1. Push malicious code directly to the repository
2. **Force-push all existing version tags** to point to the malicious commit

Tag force-pushing is the key technique here: it meant that any workflow pinned to *any* version tag — including pinning to specific versions like `v44`, `v45`, etc. — was silently redirected to execute the attacker's code. Only SHA-pinned workflows were immune.

### Malicious Payload

The injected script dumped secrets from the runner's environment and **printed them directly into the GitHub Actions workflow log output**. This included:

- `GITHUB_TOKEN` and any other tokens available in the runner environment
- Repository secrets passed to the workflow
- Any credentials set as environment variables

For public repositories, these logs are readable by anyone, meaning any secret that appeared in a log was immediately exposed to the public.

### Cached Action Risk

Workflows that had previously cached the action continued running the malicious version even after the tags were reverted, until their caches were manually purged.

---

## Timeline

| Date | Event |
|------|-------|
| Before Mar 14, 2025 | Malicious code begins impacting repositories; secrets leak into logs |
| Mar 14, 2025 | Step Security identifies the compromise and raises public awareness |
| Mar 15, 2025 | Malicious script hosted on GitHub Gist is removed |
| Mar 15, 2025 | Repository briefly taken offline, reverted, restored without malicious commits |
| Mar 15, 2025 | Maintainer publishes statement; CVE-2025-30066 assigned |

---

## Detection

```bash
# Search for usage of the compromised action in your codebase
grep -r "tj-actions/changed-files" .github/

# Check GitHub Actions workflow logs for secrets printed in plaintext
# Look at any runs that used tj-actions/changed-files before Mar 15, 2025

# If using GitHub CLI, search your org's repos:
# (Replace [your-org] with your organization name)
gh search code "tj-actions/changed-files" --owner [your-org]
```

---

## Remediation

1. **Stop using `tj-actions/changed-files` immediately** — remove all references from workflows
2. **Rotate all secrets** used in pipelines that included this action, especially for public repositories where logs may have been read
3. **Check workflow logs** from runs before March 15 for any secrets printed in plaintext — also check your third-party services for suspicious use of those tokens
4. **Purge GitHub Actions caches** to eliminate cached versions of the compromised action: Settings → Actions → Caches
5. For public repositories: **audit all exposed tokens** and treat them as fully compromised if they appeared in any log

---

## Preventive Measures

- **Pin GitHub Actions to full commit SHAs**, never to mutable tags. Use a tool like [pin-github-action](https://github.com/mheap/pin-github-action) to automate this:

```yaml
# Unsafe — tag can be force-pushed to malicious code
- uses: tj-actions/changed-files@v45

# Safe — SHA is immutable
- uses: tj-actions/changed-files@d6babd6a12e1e4b46e5d5b85b2b2b2b2b2b2b2b2  # v45
```

- **Least-privilege tokens**: Limit the secrets available in workflows; use scoped tokens
- Use **StepSecurity Harden Runner** to detect unexpected outbound network calls from runners
- **Consider alternatives** to community actions for critical CI/CD steps

---

## Lessons Learned

- **Mutable tags are a systemic risk.** Force-pushing a tag is trivially easy for anyone with repo write access. When a maintainer account is compromised, every consumer of that action who trusts a tag is immediately affected across all versions simultaneously.
- **PAT compromise = supply chain compromise.** A single compromised bot account token can grant push access to repositories used by tens of thousands of downstream consumers.
- **Caching amplifies persistence.** Actions caches extend the attack window beyond when tags are reverted; cleanup requires explicit action from every consumer.
- **Log hygiene matters.** Secrets should never appear in workflow logs. Use `::add-mask::` for sensitive values, and audit log output for accidental secret exposure.

---

## Related Incidents

- [Trivy GitHub Actions Tag Compromise (Mar 2026)](./2026-03-trivy-github-actions.md) — same technique: force-pushing tags to compromise GitHub Actions consumers (75 of 76 tags force-pushed)
- [GlassWorm / CanisterWorm (Mar 2026)](./2026-03-glassworm-canisterworm.md) — CanisterWorm linked to same-day Trivy attack; same threat actor (TeamPCP)
