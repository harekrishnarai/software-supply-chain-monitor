# Checkmarx KICS GitHub Action Compromised — Malware Injected in All Git Tags

**Date:** March 23, 2026
**Ecosystem:** GitHub Actions
**Severity:** Critical
**Type:** Tag poisoning / Infostealer / CI/CD secret theft
**Sources:**
- [StepSecurity — Checkmarx KICS GitHub Action Compromised: Malware Injected in All Git Tags](https://www.stepsecurity.io/blog/checkmarx-kics-github-action-compromised-malware-injected-in-all-git-tags)

---

## Summary

On March 23, 2026, a critical security advisory revealed that all Git tags in the `Checkmarx/kics-github-action` repository had been compromised with an infostealer payload injected into `setup.sh`. While the master branch appeared clean, every release tag was rewritten to point to malicious commits — meaning any workflow referencing the KICS Action by version tag (e.g., `@v2.1.7`, `@v1.7.0`, or `@latest`) executed attacker-controlled code.

KICS (Keeping Infrastructure as Code Secure) is Checkmarx's open-source infrastructure-as-code scanner, widely used in enterprise CI/CD pipelines for scanning Terraform, Kubernetes, Docker, CloudFormation, and other IaC files. The attack follows the same tag-poisoning pattern seen in the Trivy GitHub Actions compromise the previous week, and in the tj-actions/changed-files incident of March 2025.

Following the disclosure, the repository and GitHub issue were temporarily taken offline before being restored. Credit for discovery goes to GitHub user cyril-flieller.

---

## Compromised Artifacts

| Artifact | Malicious Version(s) / Tags | Notes |
|----------|------------------------------|-------|
| `Checkmarx/kics-github-action` | All release tags (e.g., `@v2.1.7`, `@v1.7.0`, `@latest`) | Master branch appears clean |
| `setup.sh` (within action) | Payload injected across all tagged versions | Infostealer + persistence |

---

## How It Worked

### 1. Tag Poisoning

The attacker rewrote all release tags in the `Checkmarx/kics-github-action` repository to point to malicious commits. The master branch was left clean, making the compromise harder to detect via casual repository inspection — only CI/CD workflows referencing version tags were affected.

### 2. Malicious Payload in setup.sh

The injected payload in `setup.sh` performed four distinct operations:

- **Credential theft:** Targeted cloud provider credentials across AWS, Azure, and GCP, along with SSH keys and Kubernetes service account tokens.
- **CI/CD runner memory dumps:** Performed memory dumps of CI/CD runner processes by reading `/proc/<pid>/mem` to extract secrets stored in the GitHub Actions `Runner.Worker` process memory — a technique consistent with recent supply chain attacks (Trivy, tj-actions).
- **Encrypted exfiltration:** Stolen data was encrypted and exfiltrated to `checkmarx[.]zone`, an attacker-controlled domain designed to impersonate the legitimate Checkmarx brand.
- **Persistence mechanisms:** The malware attempted to maintain access through a systemd backdoor and by deploying privileged Kubernetes pods in environments where cluster credentials were available.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 23, 2026 | All release tags in `Checkmarx/kics-github-action` rewritten to malicious commits |
| Mar 23, 2026 | GitHub user cyril-flieller discovers and reports the compromise via GitHub issue |
| Mar 23, 2026 | GitHub issue and repository taken offline |
| Mar 23, 2026 | StepSecurity publishes advisory |
| Mar 23, 2026 | Repository restored online |

---

## Detection

```bash
# Check if your workflows reference kics-github-action by version tag
grep -r "checkmarx/kics-github-action@" .github/workflows/

# Check CI/CD logs for the attacker-controlled exfiltration domain
grep -ri "checkmarx\.zone" /var/log/

# Check for unexpected systemd services (persistence mechanism)
systemctl list-units --type=service --all | grep -i kics

# Check for memory dump artifacts in runner temp directories
find /tmp -name "*.mem" -o -name "*.dump" 2>/dev/null

# Search for privileged Kubernetes pods that may have been deployed
kubectl get pods --all-namespaces -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged==true) | .metadata.name'

# Audit GitHub Actions workflow runs for the compromised action
gh run list --workflow=<workflow-file> --limit 20
```

---

## Remediation

1. **Immediately disable** any workflow referencing `checkmarx/kics-github-action` by version tag
2. **Rotate all secrets** accessible to affected CI/CD workflows: cloud provider credentials (AWS, Azure, GCP), SSH keys, Kubernetes tokens, database credentials, API keys, Docker registry credentials, and GitHub PATs
3. **Audit CI/CD logs** for outbound connections to `checkmarx[.]zone`
4. **Check for persistence:** Look for unauthorized systemd services and privileged Kubernetes pods deployed during the exposure window
5. **Pin to commit SHAs** when the action is restored to a verified safe state:
   ```yaml
   # SAFE - immutable reference
   uses: checkmarx/kics-github-action@<verified-safe-commit-sha>
   ```
6. **Review Runner.Worker process memory** exposure — if memory dumps were successful, any secret ever loaded into the runner process should be considered compromised

---

## Lessons Learned

- Tag poisoning in GitHub Actions continues to be a highly effective attack vector targeting enterprise CI/CD pipelines
- Attackers are systematically targeting security tooling (Trivy, KICS) — the very tools organizations depend on to detect vulnerabilities
- The use of brand-impersonating domains (`checkmarx[.]zone`) adds social engineering to the technical attack, making network-level detection harder
- Runner process memory dumping is becoming a standard technique in GitHub Actions compromises, extracting secrets that may not appear in environment variables
- Persistence via systemd backdoors and privileged K8s pods shows attackers are not just stealing secrets — they are establishing long-term access
- Organizations should implement network egress controls in CI/CD runners to block connections to unauthorized domains

---

## Related Incidents

- [Trivy Second Compromise — Malicious v0.69.4 & Actions Re-Poisoning](./2026-03-trivy-second-compromise.md)
- [Trivy GitHub Actions Tag Compromise](./2026-03-trivy-github-actions.md)
- [xygeni-action C2 Backdoor](./2026-03-xygeni-action.md)
- [kubernetes-el Pwn Request](./2026-03-kubernetes-el.md)
- [tj-actions/changed-files Compromise](./2025-03-tj-actions.md)
