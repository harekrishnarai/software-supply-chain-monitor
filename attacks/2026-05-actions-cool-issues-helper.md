# actions-cool/issues-helper GitHub Action Compromised — All 53 Tags Moved to Imposter Commits

**Date:** May 2026
**Ecosystem:** GitHub Actions
**Severity:** Critical
**Type:** Tag Poisoning / Imposter Commit / CI/CD Secret Theft
**Sources:**
- [StepSecurity — actions-cool/issues-helper GitHub Action Compromised: All Tags Point to Imposter Commit That Exfiltrates CI/CD Credentials](https://www.stepsecurity.io/blog/actions-cool-issues-helper-github-action-compromised-all-tags-point-to-imposter-commit-that-exfiltrates-ci-cd-credentials)

---

## Summary

On May 18, 2026, the popular GitHub Action `actions-cool/issues-helper` was compromised. All 53 existing version tags in the repository were moved to point to a single imposter commit — a commit not reachable from the action's default branch history. The imposter commit contains malicious code that, when executed inside a GitHub Actions runner, downloads the Bun JavaScript runtime, reads Runner.Worker process memory to harvest all decrypted CI/CD secrets, and exfiltrates them to `t.m-kosche.com` — the same TeamPCP C2 infrastructure used in the concurrent Mini Shai-Hulud @antv campaign.

A second action in the same organization, `actions-cool/maintain-one-comment`, was compromised by the same actor using the identical pattern: all 15 tags moved to imposter commits in a 39-second window, same Bun + Runner.Worker memory-read payload, same exfiltration domain. Because every tag now resolves to a malicious commit, any workflow referencing the action by version tag pulls the malicious code on its next run. Only workflows pinned to a known-good full commit SHA are unaffected.

The attack is part of the broader TeamPCP/Mini Shai-Hulud campaign wave that hit on May 18–19, 2026, simultaneously targeting the @antv npm ecosystem and Microsoft's durabletask PyPI package. The `t.m-kosche.com` domain appearing across all three incidents confirms shared infrastructure.

---

## Compromised Artifacts

| Action | Tags Compromised | Compromise Window (UTC) |
|--------|-----------------|------------------------|
| `actions-cool/issues-helper` | 53 tags (v1–v3.8.0) | May 18, 19:10:24 → 19:13:40 (3 min 16 sec) |
| `actions-cool/maintain-one-comment` | 15 tags (v0.0.1-wip–v3.3.0) | May 18, 19:30:30 → 19:31:09 (39 sec) |

All 68 imposter commits are dangling — none reachable from the action's default branch.

Selected imposter commit SHAs for `issues-helper`:

| Tag | Imposter SHA |
|-----|-------------|
| v3.8.0 | `1c9e803c80cc7fed000022d4c94f4b5bc2e90062` |
| v3.7.6 | `f0448c62fc57b8a5ce23d8acd6e795cdd76a3b6c` |
| v3 | `147337a919d92f4bf42f02843682d694650f1e22` |
| v2 | `3480e51eec2be03a70d6fe17507413a96fe5b57d` |
| v1 | `5c267592a87e92c2b005b338bd0d2724c2f64acb` |

---

## How It Worked

### Entry Point: Tag Takeover

The attacker gained write access to the repository (likely via a stolen GitHub token from a prior supply chain incident) and moved all existing tags to newly generated imposter commits. The attacker generated a unique imposter commit per tag — each with a fake "Build action for vX.Y.Z" commit message styled to mimic the legitimate maintainer's pattern. The timestamps reveal the attack: all 53 `issues-helper` imposter commits were created within a 3-minute, 16-second window.

Any workflow referencing `actions-cool/issues-helper@v3` (or any other tag) resolved to the attacker's commit on its next run — no workflow file changes required.

### Payload Mechanics

The imposter commit's `index.js` is executed by the Bun JavaScript runtime (downloaded to the runner during execution):

1. **Bun download**: The payload downloads the Bun runtime to `/home/runner/.bun/bin/bun`
2. **Runner.Worker memory read**: Spawns `python3` (via `sudo`) to read `/proc/<Runner.Worker PID>/mem`, then pipes through `tr -d '\0' | grep -aoE '"[^"]+":{"value":"[^"]*","isSecret":true}'` to extract all CI/CD secrets in plaintext, bypassing log masking
3. **GitHub token theft**: Runs `gh auth token` and `gh auth status --show-token`
4. **Exfiltration**: POSTs encrypted stolen data to `t.m-kosche.com:443`

The process tree observed in a controlled Harden-Runner test run:
```
npm install / action run
└─ bun.exe (PID 2401)
   ├─ gh auth token
   ├─ sudo (PID 2428)
   │   └─ python3.12 (PID 2429) → reads /proc/2175/mem (Runner.Worker)
   ├─ tr (strips null bytes)
   ├─ grep (extracts isSecret:true JSON)
   └─ sort (deduplicates)
```

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| May 18, 19:10:24 | First imposter commit created for `issues-helper` |
| May 18, 19:13:40 | Last of 53 imposter commits created (3 min 16 sec window) |
| May 18, 19:30:30 | `maintain-one-comment` tag takeover begins |
| May 18, 19:31:09 | `maintain-one-comment` tag takeover complete (39 sec) |
| May 18–19 | Simultaneous @antv npm and durabletask PyPI attacks using same t.m-kosche.com C2 |
| May 19 | StepSecurity adds t.m-kosche.com to Harden-Runner global block list |

---

## Detection

```bash
# Check if any workflow references actions-cool/issues-helper or maintain-one-comment
grep -r "actions-cool/issues-helper\|actions-cool/maintain-one-comment" .github/workflows/

# Verify that the SHA used in any pinned reference is legitimate (reachable from main branch)
# Imposter commits are NOT reachable from the default branch:
git ls-remote https://github.com/actions-cool/issues-helper.git | grep -E "refs/tags/"
# Compare tag SHAs against the imposter list above

# Check GitHub Actions logs for:
# - bun runtime download to /home/runner/.bun/bin/bun
# - python3 reading /proc/*/mem
# - gh auth token / gh auth status --show-token calls
# - Outbound connections to t.m-kosche.com

# Search for Runner.Worker memory reads in workflow logs:
# Look for: FILE_READ /proc/<PID>/mem -> Runner.Worker

# Check if the referenced commit is a dangling commit:
# GitHub will show a warning: "This commit does not belong to any branch on this repository"

# Detect imposter commits by timestamp clustering:
# All 53 issues-helper commits were created within 196 seconds
```

---

## Remediation

1. **Pin all GitHub Action references to full commit SHAs** instead of version tags:
   ```yaml
   # UNSAFE (resolves to attacker's imposter commit)
   uses: actions-cool/issues-helper@v3
   
   # SAFE (pinned to a specific, verified commit)
   uses: actions-cool/issues-helper@<full-40-char-sha>
   ```
   Verify the SHA is reachable from the action's default branch before pinning.

2. **Remove or disable any workflow** referencing `actions-cool/issues-helper` or `actions-cool/maintain-one-comment` until the maintainers confirm remediation

3. **Rotate all CI/CD secrets** that may have been present in any workflow that ran the action during the compromise window — including `GITHUB_TOKEN`, npm tokens, AWS/GCP/Azure credentials, and all repository/environment/organization secrets

4. **Audit workflow run logs** for the compromise window (May 18, 19:10 UTC onward) for any job that used these actions

5. **Block `t.m-kosche.com`** at the network perimeter and in any egress allowlists for CI/CD runners

6. **Notify the maintainers** via GitHub issue if not already done (StepSecurity filed GitHub Issue #11 for `maintain-one-comment`)

---

## Lessons Learned

- Mutable tags in GitHub Actions are a fundamental supply chain risk — a single tag move can re-point every consumer without any pull request or review
- The 3-minute, 16-second window to compromise all 53 tags shows how quickly an attacker can execute a tag takeover once they have write access
- Runner.Worker memory scraping captures every CI/CD secret regardless of log masking — the only defense is preventing the malicious code from executing in the first place (via SHA pinning or network egress controls)
- The simultaneous multi-vector attack (GitHub Actions tags + npm packages + PyPI package) using the same C2 infrastructure demonstrates that TeamPCP operates coordinated, synchronized campaigns
- Imposter commit message styling ("Build action for vX.Y.Z") specifically mimics legitimate maintainer patterns to evade casual log inspection

---

## Related Incidents

- [./2026-05-antv-npm-shai-hulud.md](./2026-05-antv-npm-shai-hulud.md) — Simultaneous TeamPCP npm wave
- [./2026-05-durabletask-pypi.md](./2026-05-durabletask-pypi.md) — Simultaneous TeamPCP PyPI attack
- [./2026-04-checkmarx-kics-docker-vscode.md](./2026-04-checkmarx-kics-docker-vscode.md) — Prior TeamPCP imposter commit via GitHub Actions
- [./2026-03-xygeni-action.md](./2026-03-xygeni-action.md) — xygeni-action tag poisoning with 7-day C2 shell
- [./2025-03-tj-actions.md](./2025-03-tj-actions.md) — tj-actions/changed-files tag poisoning (23K+ repos affected)
