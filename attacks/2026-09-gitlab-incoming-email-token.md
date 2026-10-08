# GitLab Incoming Email Token — Account-Wide Credential Hidden in Issue-Creation Address Enables Code Push, CI/CD Execution, and IP Restriction Bypass

**Date:** September 2026
**Ecosystem:** GitLab / CI/CD
**Severity:** High
**Type:** Insecure Default / Token Misuse / Credential Exposure
**Sources:**
- [Aikido — Send GitLab an email, push to main](https://www.aikido.dev/blog/gitlab-email-push-to-main)

---

## Summary

On September 23, 2026, Aikido researcher Joe Leon published research revealing that GitLab's incoming email feature for issue creation embeds a long-lived, account-wide credential inside what the GitLab UI presents as a simple, project-scoped email address. Every GitLab account automatically receives an incoming email address of the form `incoming+<project-id>-glimt-<TOKEN>-issue@incoming.gitlab.com` when the "Email work item to this project" feature is enabled. GitLab's own UI described these addresses as issue-only and said they "cannot be used to access any other data" — both statements were incorrect.

The `glimt-*` token embedded in the address is effectively a fine-grained personal access token with significant permissions across every project the account can reach. An attacker who obtains a single leaked address (e.g., from a README, contributing guide, or support page) can: push code commits to any branch the victim can push to (including `main` and protected branches), create merge requests carrying arbitrary `.gitlab-ci.yml` changes that GitLab will execute as CI/CD jobs, read confidential issues and exfiltrate CI/CD secrets, and do all of the above while bypassing IP allowlist restrictions that block HTTP and SSH access entirely.

Aikido disclosed this to GitLab via HackerOne in May 2026 (closed as intended behavior) and via a confidential GitLab issue in June 2026. GitLab shipped minor UI copy updates acknowledging merge request creation but left the underlying mechanism intact: the token is still account-wide, still never expires, and still bypasses IP restrictions. During research, Aikido found over a dozen live incoming email addresses published deliberately in public READMEs and contributing guides of popular open source projects.

---

## Compromised Artifacts

| Artifact | Scope | Notes |
|----------|-------|-------|
| `glimt-*` token in incoming email address | Account-wide | Reaches all projects the account can access; never expires; account can only rotate by resetting all incoming email addresses simultaneously |
| GitLab.com | All accounts | All users with incoming email enabled (on by default) |
| GitLab Self-Managed | All instances with incoming email enabled | GitLab Dedicated not affected |

---

## How It Worked

### Entry Point: Leaked Incoming Email Address

Every GitLab project presents users with a "Email work item to this project" button that reveals an address of the form:

```
incoming+<project-path-slug>-<project-id>-glimt-XXXXXXXXXXXXXX-issue@incoming.gitlab.com
```

The address looks like a standard email address, and GitLab's UI called it an email address that creates one item in one project. In reality, the `glimt-*` string is a credential that authenticates the holder as the account owner across every project on the account. Because GitLab does not verify the email sender address, anyone holding the address has both authentication and authorization — not just the account owner.

### From Issues to Code Pushes

Changing the `-issue@` suffix to `-merge-request@` converts the request type. Email-submitted merge requests accept git `.patch` file attachments. GitLab applies the patch to the named source branch (creating the branch if it does not exist) and, if the patch modifies `.gitlab-ci.yml` and the victim's role allows it, GitLab executes the attacker's CI/CD job.

Complete attack path:
1. Attacker obtains a leaked `-issue@` address from a public README, support page, or search.
2. Attacker changes suffix to `-merge-request@`.
3. Attacker crafts a `.patch` file adding a malicious job to `.gitlab-ci.yml` (no prior repository knowledge required).
4. Attacker emails the patch, specifying a source branch in the subject line.
5. GitLab pushes the commit to that branch and runs the CI/CD job as the account owner.
6. Any developer who pulls from the repository downloads the attacker's code.

### IP Restriction Bypass

GitLab's IP allowlist feature is designed to block all access to a project from non-approved IP ranges. Aikido confirmed that HTTP requests and `git clone` commands were blocked from an unapproved IP, while an incoming merge-request email was accepted and applied — GitLab's inbound mail path is not subject to IP restriction controls.

### Attacker-Confirmed Capabilities

Aikido confirmed the following attack paths against their own projects:
- Pushing commits to protected branches in private repositories behind an IP allowlist
- Exfiltrating source code from private projects behind an IP allowlist
- Reading CI/CD variables and secrets from private projects
- Accessing confidential issues via `/move` quick action
- Using the `CI_JOB_TOKEN` available in pipeline jobs for lateral movement within the account

### Realistic Exposure Vector

The token is indistinguishable from an email address in both form and function. GitLab's UI actively encouraged maintainers to publish these addresses so users could report bugs; many did. Aikido surfaced over a dozen live, unexpired tokens in a single afternoon of searching public documentation.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Unknown (pre-May 2026) | Joe Leon (Aikido) discovers the issue |
| May 2026 | Aikido reports via GitLab's HackerOne program |
| May 2026 | HackerOne closes report as intended behavior |
| Jun 2026 | Aikido files confidential issue directly on GitLab repository; receives more detailed response |
| Jun–Sep 2026 | GitLab ships UI copy updates: removes "cannot be used to access any other data", adds "and merge requests" to surface descriptions, documents that incoming email is not subject to IP restrictions |
| Sep 23, 2026 | Aikido publishes full research; over a dozen live tokens publicly exposed in READMEs at time of disclosure |

---

## Detection

```bash
# Scan all git repos and documentation for leaked glimt- tokens
grep -r 'glimt-' . 2>/dev/null
git log --all --oneline -p | grep 'glimt-' 2>/dev/null

# Scan for full incoming email address patterns
grep -rE 'incoming\+[a-z0-9_-]+-glimt-[A-Za-z0-9_-]+-[a-z]+-[a-z]+@incoming\.gitlab\.com' . 2>/dev/null

# Search for older token formats (pre-glimt- prefix)
grep -rE 'incoming\+[a-z0-9_-]+-[A-Za-z0-9_]{20,}-issue@incoming\.gitlab\.com' . 2>/dev/null

# Check for commits that originated via email (look for GitLab incoming email commit signature)
git log --format="%H %ae %s" | grep '@incoming.gitlab.com' 2>/dev/null

# Audit git history for unexpected .gitlab-ci.yml changes
git log --all --diff-filter=M -- .gitlab-ci.yml 2>/dev/null

# Check for unexpected branches that may have been created via email patch
git branch -a --sort=-creatordate | head -20
```

---

## Remediation

1. **Rotate your incoming email token immediately** — go to GitLab → User Settings → Access Tokens → Incoming email token → Reset. Note: resetting invalidates ALL project-specific incoming email addresses on your account simultaneously.
2. **Scan your repositories and public documentation** for published `glimt-` tokens using the detection commands above. Rotate after any confirmed exposure.
3. **Do not rely on IP allowlists as an absolute security boundary in GitLab** — they do not apply to the inbound email path.
4. **Audit recent CI/CD job history** for jobs triggered via email-submitted merge requests or unexpected commits on unusual branches.
5. **Enable branch protection rules** to require approval for merge requests even from Maintainer-level tokens, reducing the impact of a compromise.
6. **Deploy secret scanning** (e.g., Aikido Betterleaks, GitLab Secret Detection) to detect `glimt-` tokens before they reach public repositories.

---

## Lessons Learned

- **Credentials disguised as email addresses are uniquely dangerous** — users have a trained expectation that email addresses are safe to share; embedding long-lived credentials inside an address format breaks this mental model.
- **UI trust anchors shape security posture** — GitLab's explicit statement that the address "cannot be used to access any other data" gave users false confidence; a misleading security guarantee is often worse than no guarantee at all.
- **IP allowlists are not perimeter security for services with multiple access protocols** — any network-accessible service that has both an HTTP/SSH path and an alternative ingress (email, API, webhook) must apply access controls to all paths, not just the primary ones.
- **Account-wide tokens with project-scoped UI presentation create blast radius surprises** — users expect the scope of a feature to match the scope of its credential; violating this mapping silently multiplies a token's effective risk.
- **Disclosure closed as "intended behavior" does not mean safe behavior** — GitLab's eventual partial acknowledgment came after public pressure; organizations should treat "works as designed" responses to security reports as an opportunity to reassess default configurations.

---

## Related Incidents

- [./2026-06-amazon-q-mcp-auto-execution.md](./2026-06-amazon-q-mcp-auto-execution.md) — Another example of a developer tool auto-executing workspace content (MCP configs) without user consent
- [./2026-04-prt-scan-github-actions.md](./2026-04-prt-scan-github-actions.md) — CI/CD credential theft via GitHub Actions PR exploitation
- [./2026-07-ghostapproval-ai-coding-assistants.md](./2026-07-ghostapproval-ai-coding-assistants.md) — Trust boundary flaws in developer tooling that enable code execution without user awareness
