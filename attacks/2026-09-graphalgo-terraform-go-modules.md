# Graphalgo Campaign — Malicious Terraform Providers and Go Modules Deploy Dual-Channel RAT (First Terraform Malware)

**Date:** September 2026
**Ecosystem:** Terraform Registry / Go Modules / npm (prior waves)
**Severity:** High
**Type:** Targeted RAT / Supply Chain Attack / Typosquatting
**Sources:**
- [Aikido — Graphalgo Malware Spreads to Terraform and Go](https://www.aikido.dev/blog/graphalgo-terraform-go-modules)
- [ReversingLabs — Fake Recruiter Campaign Targets Crypto Devs (Feb 2026)](https://www.reversinglabs.com/blog/fake-recruiter-campaign-crypto-devs)
- [ReversingLabs — Graphalgo Campaign Respawned (Apr 2026)](https://www.reversinglabs.com/blog/graphalgo-campaign-respawned)
- [CheckMarx — npm btree malware campaign](https://checkmarx.com/zero-post/npm-btree-malware-campaign-affects-millions-of-downloads-no-need-for-install-script/)
- [JFrog — Equation of Compromise](https://research.jfrog.com/post/equation-of-compromise/)
- [SafeDep — mathmain encrypted loader](https://safedep.io/mathmain-encrypted-loader/)

---

## Summary

On September 22, 2026, Aikido Security identified Go malware distributed through at least two Terraform providers and two Go Modules, linking the campaign to the DPRK-attributed Graphalgo operation first reported by ReversingLabs in February 2026. This is the first confirmed instance of systematic malware distribution through the Terraform Registry — a vector that targets DevOps workstations with a direct pathway to production infrastructure credentials.

The four malicious packages are: Terraform providers `gocommunity-io/dockerd` and `kreuzwenker/docker` (a typosquat of the legitimate `kreuzwerker/docker` with 56 million downloads), and Go Modules `gocommunity.io/orderedbtree` and `gogets.dev/btreex`. The malware is a Go port of the Graphalgo JavaScript payloads seen across npm since April 2026, sharing the same threat actor public key, blockchain contract address, and Slack workspace infrastructure.

Critically, both Terraform providers implement a trigger condition: the malware activates only when the SHA-256 hash of two Terraform variables (`containerName` + `networkID`) matches a specific attacker-controlled value. This means the providers are inert in most environments, activating only against pre-selected targets — a hallmark of a targeted espionage or credentials-theft operation rather than opportunistic mass compromise. The Slack C2 channel contained 18 unique victim hostnames and 1,402 blockchain contract transactions, suggesting a small but active campaign. The threat actor also constructed two fake Go package ecosystems (`gocommunity[.]io` and `gogets[.]dev`) to lend legitimacy to the malicious packages, indicating sustained investment in the Go supply chain attack surface.

---

## Compromised Artifacts

| Package / Provider | Type | Malicious Versions | Notes |
|--------------------|------|--------------------|-------|
| `gocommunity-io/dockerd` | Terraform Provider | All versions | TA-controlled |
| `kreuzwenker/docker` | Terraform Provider | All versions | Typosquat of `kreuzwerker/docker` (56M DLs) |
| `gocommunity.io/orderedbtree` | Go Module | First published Aug 11, 2026 | Plaintext payload |
| `gogets.dev/btreex` | Go Module | First published Sep 8, 2026 | Backdated commits to Nov 2025 |

---

## How It Worked

### Entry Point: Terraform Provider Trigger

Both Terraform providers (`gocommunity-io/dockerd` and `kreuzwenker/docker`) contain hidden entry points in `/internal/provider/resource_docker_container_funcs.go`. The malicious code activates only when the SHA-256 hash of the concatenated `containerName` and `networkID` Terraform input variables equals:

```
b9966e3762e9a0d5d263b8cb3cca07294f81af9714d40ddf4628cb85d74e8ad5
```

This hash is used as an AES key to decrypt an embedded file path pointing to the second-stage archive. The malware unzips the archive, AES-decrypts each file, and executes the result via a detached `go run .` command. The target archive in both providers is `examples/resources/docker_container/import-resource.sqlite3` (SHA256: `5f892a5424e88a21a3eb3d7f82ebf04d8ac31cdb19ada25153be4165df977d0f`).

### Go Module Variants

`gocommunity.io/orderedbtree` (Aug 11) embedded the malware in plaintext. `gogets.dev/btreex` (Sep 8) hid it inside a ZIP archive masquerading as a SQL file (`btreex.sql`, SHA256: `ab01686d87565250fc4989faddb877d793667b07ec217a61cbd798f5695d62f5`). The btreex variant also required specially crafted input — an object with a specific price integer value — before activating, consistent with the targeted approach of the Terraform providers. The threat actor used forged commits to backdate the btreex repository to November 2025; Go's module proxy treats commit dates as authoritative, so `pkg.go.dev` displayed the falsified date.

### Second-Stage RAT: Dual C2

The second-stage RAT is a Go agent with two simultaneous C2 channels:

**Slack C2:** On first execution, the RAT collects system information (OS, architecture, hostname, username, home directory, Node.js availability, timestamp) and posts it in plaintext to the `frontend-devs` channel of `portfolio-devs.slack[.]com`. It then generates an ephemeral key pair and derives a shared key using the threat actor's hardcoded public key. All subsequent communication is encrypted and routed to the `qa-announcements` channel of `portfolio-testers.slack[.]com`. The malware polls for Slack instructions every 10 seconds.

**Blockchain C2:** The RAT connects to an Ethereum smart contract on the Arbitrum Sepolia testnet at address `0xAD02b5cDE693529d3bdA0266299501ad0193036C`. It retrieves a private key, decrypts it, creates a shared key with the threat actor's public key, writes its own ephemeral public key to the contract via `setCPubKey`, and reads encrypted commands from `serviceData1` and `serviceData2`. The shared wallet has write-only access for `setCPubKey`, preventing rogue clients from injecting dummy commands. The RAT polls for blockchain instructions every 3 seconds.

Decrypted C2 messages contain a header indicating whether the body should be executed as Go code, JavaScript code (via `node`), or a self-deletion command.

### Scale and Targeting

The `qa-announcements` Slack channel contained 1,240 encrypted messages since July 16, 2026. The `frontend-devs` channel contained 725 plaintext check-in messages from 18 unique hostnames — 3 Windows, 5 Linux, 10 macOS. The contract logged 1,402 transactions from August 6, 2026. The threat actor's own test device (hostname: `Frank`, username: `Frank1`, Windows) appeared in the earliest messages from July 16.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Feb 2026 | ReversingLabs first documents Graphalgo campaign in npm, attributes to DPRK |
| Apr 2026 | ReversingLabs documents Graphalgo respawn; public key `302a300...` first observed in `modern-events` npm package |
| Jul 16, 2026 | Earliest Slack check-in message (threat actor testing on hostname `Frank`) |
| Aug 6, 2026 | First blockchain contract transaction recorded |
| Aug 11, 2026 | `gocommunity.io/orderedbtree` Go Module first published |
| Early Sep 2026 | `gocommunity-io/dockerd` and `kreuzwenker/docker` Terraform providers published |
| Sep 8, 2026 | `gogets.dev/btreex` Go Module published (backdated to Nov 2025) |
| Sep 22, 2026 | Aikido discloses campaign; first public documentation of Terraform provider malware vector |

---

## Detection

```bash
# Check for malicious Terraform providers in .terraform.lock.hcl
grep -E 'gocommunity-io/dockerd|kreuzwenker/docker' .terraform.lock.hcl .terraform/**/.terraform.lock.hcl 2>/dev/null

# Check Terraform provider cache for the malicious archive
find ~/.terraform.d/plugins -name 'import-resource.sqlite3' 2>/dev/null
sha256sum ~/.terraform.d/plugins/**/import-resource.sqlite3 2>/dev/null
# Malicious hash: 5f892a5424e88a21a3eb3d7f82ebf04d8ac31cdb19ada25153be4165df977d0f

# Check Go module cache for malicious packages
ls $(go env GOPATH)/pkg/mod/gocommunity.io/ 2>/dev/null
ls $(go env GOPATH)/pkg/mod/gogets.dev/ 2>/dev/null

# Check for btreex SQL archive
find $(go env GOPATH)/pkg/mod -name 'btreex.sql' 2>/dev/null
sha256sum $(go env GOPATH)/pkg/mod/**/btreex.sql 2>/dev/null
# Malicious hash: ab01686d87565250fc4989faddb877d793667b07ec217a61cbd798f5695d62f5

# Check for the RAT state directory
ls -la ~/.openclaw/ 2>/dev/null

# Check for outbound connections to Slack API (non-business use) or Arbitrum
# In network logs (macOS):
log show --predicate 'process == "go" AND eventMessage CONTAINS "slack.com"' --last 7d 2>/dev/null
# In /var/log (Linux):
grep -r 'api\.slack\.com\|arbitrum\|sepolia' /var/log/ 2>/dev/null

# Check for the threat actor's public key in any local Go binaries
grep -r '302a300506032b656e032100bad013df6eec5d686f4cc8551e0a5c87a0135164bdd1dafb1c75141d1b526702' \
  $(go env GOPATH)/pkg/mod 2>/dev/null

# Audit recent Terraform applies and Go module downloads on a machine
ls -lt ~/.terraform.d/plugins/ 2>/dev/null | head -20
cat $(go env GOPATH)/pkg/mod/cache/download/sumdb/sum.golang.org/lookup/gogets.dev/btreex* 2>/dev/null
```

---

## Remediation

1. **Remove malicious providers and modules:** Delete any Terraform configuration referencing `gocommunity-io/dockerd` or `kreuzwenker/docker`. Remove `gocommunity.io/orderedbtree` and `gogets.dev/btreex` from `go.mod` and purge from the module cache (`go clean -modcache`).
2. **Isolate the machine immediately:** The RAT uses Slack API and Ethereum RPC calls for C2. Removing the package does not terminate an active RAT session. Network-isolate the machine before rotating credentials.
3. **Rotate all credentials:** Prioritize cloud provider credentials (AWS, GCP, Azure), Terraform state backend credentials, infrastructure tokens, GitHub/GitLab tokens, npm/PyPI publish tokens, and SSH keys. Treat all credentials that touched the machine as compromised regardless of whether they appear in the RAT's explicit target list.
4. **Audit Terraform applies:** Review all `terraform apply` runs executed from the affected machine. Assume any production infrastructure changes during the exposure window may have been observed or manipulated.
5. **Reimage the machine:** The RAT executes a detached `go run .` second stage. Process termination and package deletion do not guarantee clean state. Reimage is required for confidence.
6. **Block blockchain RPC and anomalous Slack API usage:** Alert on outbound HTTP to `api.slack.com` from CI/CD runners or Terraform workstations that have no business need. Alert on connections to Arbitrum/Ethereum RPC endpoints (`arbitrum.io`, `sepolia.io`, `infura.io`, `alchemy.com`) from developer tooling.

---

## Lessons Learned

- **Terraform providers are a blind spot in supply chain scanning.** Most SCA and malware tooling covers npm, PyPI, Maven, and RubyGems — not Terraform Registry or Go Modules. This campaign exploits that gap directly.
- **Targeted activation defeats broad scanning.** Requiring a specific SHA-256 hash of runtime inputs to arm the payload means automated sandboxes will never trigger it. Behavioral detection and network-level signals (Slack API calls from Terraform, Ethereum RPC from build pipelines) are the only scalable detection controls.
- **Fake ecosystem infrastructure signals sustained investment.** Creating `gocommunity[.]io` and `gogets[.]dev` with functional-looking websites requires effort. This is not a throwaway operation — the threat actor is building durable social engineering infrastructure for Go developers.
- **DevOps workstations are higher-value targets than developer workstations.** Terraform users have direct write access to cloud infrastructure. A credential stolen from a DevOps workstation converts directly into production compromise, not just source code or package exfiltration.
- **Backdated commits in Go Modules are credible.** The Go module proxy and `pkg.go.dev` surface forged commit timestamps without any verification. There is currently no mechanism to detect this without independent timestamp sources.
- **Blockchain C2 is maturing.** The Arbitrum Sepolia implementation here features per-client key isolation, write-only shared wallet permissions, and polling across two independent channels. Blocking or alerting on blockchain RPC traffic from development tooling should be a standard hardening step.

---

## Related Incidents

- [ChainDrop npm Worm — EtherHiding C2 (Aug 2026)](./2026-08-chaindrop-npm-worm.md)
- [IronWorm — Ethereum C2 in npm (Jun 2026)](./2026-06-ironworm-npm-weavedb.md)
- [@joyfill npm Supply Chain Attack — Blockchain C2 RAT (Jul 2026)](./2026-07-joyfill-npm-rat.md)
