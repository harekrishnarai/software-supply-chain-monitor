# Trivy Second Compromise — Malicious v0.69.4 Release & GitHub Actions Re-Poisoning

**Date:** March 19, 2026
**Ecosystem:** GitHub Actions / Homebrew / npm
**Severity:** Critical
**Type:** Tag poisoning / Malicious binary release / CI/CD secret theft
**Sources:**
- [StepSecurity — Trivy Compromised a Second Time](https://www.stepsecurity.io/blog/trivy-compromised-a-second-time---malicious-v0-69-4-release)

---

## Summary

Three weeks after the initial hackerbot-claw incident that resulted in a full repository takeover of Trivy on February 28, 2026, attackers re-compromised the Trivy ecosystem in a broader and more coordinated assault. Aqua Security confirmed that containment of the first incident was incomplete, and the attackers exploited residual access to publish a malicious Trivy binary (v0.69.4), re-poison all version tags in `aquasecurity/trivy-action`, compromise `aquasecurity/setup-trivy`, delete the original incident discussion to suppress warnings, and deploy spam bots to bury legitimate security disclosures.

The `aquasecurity/trivy-action` was compromised for approximately 12 hours, `aquasecurity/setup-trivy` for approximately 4 hours, and the malicious v0.69.4 binary was live for approximately 3 hours before Homebrew maintainers issued an emergency downgrade.

---

## Compromised Artifacts

| Artifact | Malicious Version(s) / Tags | Duration |
|----------|------------------------------|----------|
| `aquasecurity/trivy-action` | All tags except one rewritten to malicious commits | ~12 hours |
| `aquasecurity/setup-trivy` | All tags except v0.2.6 deleted/rewritten | ~4 hours |
| Trivy CLI binary | v0.69.4 (published to GitHub Releases) | ~3 hours |
| Homebrew `trivy` formula | v0.69.4 pulled via automation | Until emergency PR #273304 |

---

## How It Worked

### 1. Malicious Trivy v0.69.4 Binary Published

The attackers published a malicious `trivy` v0.69.4 release to the official GitHub repository. Homebrew's automated CI pipeline (`BrewTestBot`) picked up the new version and merged an update PR within approximately 42 minutes of publication. The malicious binary contained the same TeamPCP credential stealer payload seen in the first compromise.

### 2. Incident Discussion Deleted and Spam Bot Flood

The original incident discussion thread where community members reported the compromise was deleted by the attackers. They then deployed approximately 70 spam bot accounts that posted generic praise comments within a single second at 00:08 UTC — a coordinated bot attack to bury real security discussion.

### 3. trivy-action Re-Compromised

The same credential stealer was injected into `aquasecurity/trivy-action` via imposter commits. All tags (except one) were modified to point to malicious commits containing a modified `entrypoint.sh` with +105/-2 lines of injected payload. The payload included Runner process environment harvesting, a base64-encoded Python credential stealer (labelled `TeamPCP Cloud stealer`), RSA encryption, and exfiltration to `scan[.]aquasecurtiy[.]org`.

### 4. setup-trivy Compromised

`aquasecurity/setup-trivy` was similarly compromised. The compromised version installed the malicious v0.69.4 binary. All version tags except v0.2.6 were deleted during incident response, breaking CI pipelines pinned to deleted tags.

### 5. Homebrew Emergency Downgrade

Homebrew maintainer woodruffw filed PR #273304 to emergency downgrade trivy back to v0.69.3, using special labels to bypass bot-driven CI delays.

---

## The Payload

The payload is functionally identical to the first Trivy compromise — a multi-stage credential stealer targeting:

| Category | Targets |
|----------|---------|
| CI/CD Secrets | Runner.Worker process memory dump; `isSecret:true` JSON extraction |
| SSH | `~/.ssh/id_rsa`, `id_ed25519`, `id_ecdsa`, `authorized_keys`, host keys |
| Cloud (AWS) | `~/.aws/credentials`, EC2 IMDS, ECS credentials |
| Cloud (GCP) | `~/.config/gcloud/*`, `application_default_credentials.json` |
| Cloud (Azure) | `~/.azure/*`, `AZURE_*` env vars |
| Kubernetes | `~/.kube/config`, `kubectl get secrets --all-namespaces -o json` |
| Docker | `~/.docker/config.json`, `/kaniko/.docker/config.json` |
| Env Files | `.env`, `.env.local`, `.env.production`, `.env.staging` across multiple paths |

Collected data is encrypted with AES-256-CBC (AES key wrapped with RSA-4096) and exfiltrated via a GitHub-based relay.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Feb 28, 2026 | First Trivy compromise (hackerbot-claw full repo takeover) |
| Mar 19, 2026 ~17:51 | `aqua-bot` deletes v0.70.0 tag |
| Mar 19, 2026 ~18:30 | `github-actions[bot]` creates `ci/helm-chart/bump-trivy-to-0.69.4` branch |
| Mar 19, 2026 ~18:30 | `aqua-bot` opens helm chart bump PR for v0.69.4 |
| Mar 19, 2026 ~22:13 | Community member opens discussion #10420 reporting compromise |
| Mar 19, 2026 | Malicious v0.69.4 binary published to GitHub Releases |
| Mar 19, 2026 | Homebrew BrewTestBot auto-merges v0.69.4 update (~42 min after publication) |
| Mar 19, 2026 00:08 | ~70 spam bot accounts flood discussion with generic praise comments |
| Mar 19, 2026 | Original incident discussion deleted by attackers |
| Mar 20, 2026 | trivy-action tags rewritten with credential stealer payload |
| Mar 20, 2026 | setup-trivy tags rewritten/deleted |
| Mar 20, 2026 | Homebrew emergency downgrade PR #273304 filed |
| Mar 20, 2026 | Aqua Security confirms incomplete containment of first incident; begins remediation |

---

## Detection

```bash
# Check if you used the malicious trivy version
trivy --version  # Should NOT be v0.69.4

# Check for the malicious exfiltration domain in CI logs
grep -r "aquasecurtiy\.org" /var/log/  # Note: typo in domain is intentional by attacker

# Check trivy-action commit SHAs against known-malicious commits
# See: https://github.com/step-security/trivy-compromise-scanner

# Check for the malicious setup-trivy commit
git -C /path/to/setup-trivy log --oneline | grep "8afa9b9"

# Verify trivy-action is pinned to a safe SHA
grep -r "aquasecurity/trivy-action@" .github/workflows/

# Check Homebrew trivy version
brew info trivy | head -1
```

---

## Remediation

1. **Pin GitHub Actions to full commit SHAs**, not version tags:
   ```yaml
   # Vulnerable
   - uses: aquasecurity/trivy-action@0.33.0
   # Safe — use SHA from official disclosure
   - uses: aquasecurity/trivy-action@<verified-safe-sha>
   ```
2. **Rotate all secrets** present in any CI/CD pipeline that used compromised Trivy actions or binaries between March 19–20
3. **Downgrade Trivy CLI** to v0.69.3 if you installed v0.69.4 via any channel
4. **Audit CI/CD logs** for outbound connections to `scan[.]aquasecurtiy[.]org` or IP `45.148.10.212`
5. **Review Homebrew installations** — ensure `brew upgrade trivy` did not install v0.69.4
6. Do not rely on GitHub's "Immutable" release badge as a security guarantee

---

## Lessons Learned

- Incomplete incident response creates opportunities for re-compromise; credential rotation must be exhaustive
- Attackers actively suppress disclosure by deleting issues and deploying spam bot floods
- Homebrew's automated update pipeline can propagate malicious releases within minutes
- Tag-based pinning for GitHub Actions remains fundamentally insecure; only full SHA pinning is safe
- The same TeamPCP threat actor demonstrated persistent access and willingness to re-attack the same target

---

## Related Incidents

- [Trivy GitHub Actions Tag Compromise (First Compromise)](./2026-03-trivy-github-actions.md)
- [hackerbot-claw AI Attack Campaign](./2026-02-hackerbot-claw.md)
- [CanisterWorm NPM Worm (post-Trivy)](./2026-03-canisterworm-npm.md)
- [tj-actions/changed-files Compromise](./2025-03-tj-actions.md)
