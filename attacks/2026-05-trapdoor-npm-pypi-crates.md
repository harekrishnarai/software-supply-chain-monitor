# TrapDoor — Cross-Ecosystem Credential Stealer Targets npm, PyPI, and Crates.io with AI Assistant Poisoning

**Date:** May 2026
**Ecosystem:** npm / PyPI / Crates.io
**Severity:** High
**Type:** Credential Stealer / AI Assistant Poisoning / Cross-Ecosystem Campaign
**Sources:**
- [Socket Research Team — TrapDoor Crypto Stealer Supply Chain Attack Hits 34 Packages and Hundreds of Versions Across npm, PyPI, and Crates.io](https://socket.dev/blog/trapdoor-crypto-stealer-npm-pypi-crates)
- [The Hacker News — TrapDoor Supply Chain Attack Spreads Credential-Stealing Malware via npm, PyPI, and CratesIO](https://thehackernews.com/2026/05/trapdoor-supply-chain-attack-spreads.html)

---

## Summary

TrapDoor is a coordinated cross-ecosystem supply chain campaign discovered by Socket on May 22, 2026, spanning npm, PyPI, and Crates.io. It is notably the first significant supply chain attack campaign to include Crates.io (the Rust package registry) alongside the more commonly targeted npm and PyPI registries. The campaign deployed 34+ malicious packages across 384+ versions, disguised as developer utilities, security scanners, and crypto tooling for DeFi, Solana, AI, and Web3 developers.

The attacker, operating under the GitHub account `ddjidd564`, used ecosystem-specific execution paths — postinstall hooks in npm, import-time execution in PyPI, and `build.rs` build scripts in Crates.io — to reach developers during normal workflows. The shared npm payload, `trap-core.js`, is a 1,149-line credential harvester that also validates stolen credentials via live AWS and GitHub API calls to separate useful secrets from expired ones.

What distinguishes TrapDoor from comparable campaigns is its novel AI assistant poisoning technique. The malware plants hidden instructions into `.cursorrules` and `CLAUDE.md` files using zero-width Unicode characters, designed to trick AI coding assistants (Cursor, Claude Code) into running a "security scan" that results in credential discovery and exfiltration. The attacker also opened pull requests against major AI and developer projects — including LangChain, LlamaIndex, OpenHands, and browser-use — attempting to embed these poisoned configuration files into mainstream repositories.

---

## Compromised Artifacts

### npm Packages (published by account `asdxzxc`)

| Package | Notes |
|---------|-------|
| `async-pipeline-builder` | Postinstall + trap-core.js |
| `build-scripts-utils` | Postinstall + trap-core.js |
| `chain-key-validator` | Crypto key targeting |
| `crypto-credential-scanner` | Credential harvester |
| `defi-env-auditor` | DeFi developer targeting |
| `defi-threat-scanner` | Security tool disguise |
| `deployment-key-auditor` | SSH/cloud key targeting |
| `dev-env-bootstrapper` | Dual-purpose: malware + delivery vector |
| `eth-wallet-sentinel` | Ethereum wallet targeting |
| `llm-context-compressor` | AI developer targeting |
| `mnemonic-safety-check` | Crypto wallet mnemonic theft |
| `model-switch-router` | AI developer lure |
| `node-setup-helpers` | Generic developer lure |
| `project-init-tools` | Generic developer lure |
| `prompt-engineering-toolkit` | AI developer targeting |
| `solidity-deploy-guard` | Solidity developer targeting |
| `token-usage-tracker` | AI/cloud token theft |
| `wallet-backup-verifier` | Crypto wallet targeting |
| `wallet-security-checker` | Crypto wallet targeting |
| `web3-secrets-detector` | Web3 developer targeting |
| `workspace-config-loader` | General developer lure |

### PyPI Packages (accounts `asdmini67`, `dae5411`)

| Package | Notes |
|---------|-------|
| `cryptowallet-safety` | Wallet targeting |
| `data-pipeline-check` | Developer tool lure |
| `defi-risk-scanner` | DeFi developer lure |
| `env-loader-cli` | Env var harvester |
| `eth-security-auditor` | Earliest observed (May 22, 2026 20:20 UTC) |
| `git-config-sync` | Git credential theft |
| `solidity-build-guard` | Solidity developer targeting |

### Crates.io Packages (targeting Sui/Move developers)

| Package | Notes |
|---------|-------|
| `move-analyzer-build` | Move language tooling lure |
| `move-compiler-tools` | Move language tooling lure |
| `move-project-builder` | Move language tooling lure |
| `sui-framework-helpers` | Sui blockchain targeting |
| `sui-move-build-helper` | Sui blockchain targeting |
| `sui-sdk-build-utils` | Sui blockchain targeting |

---

## How It Worked

### Entry Points (Ecosystem-Specific)

The campaign used three different execution paths to match each registry's norms:

- **npm**: `postinstall` hook in `package.json` runs `trap-core.js` immediately after installation
- **PyPI**: Import-time execution — malicious code runs when the package is first imported. Packages download and execute remote JavaScript using `node -e` from `ddjidd564[.]github[.]io`
- **Crates.io**: `build.rs` build script runs automatically during `cargo build`, before the developer executes any package functionality — enabling keystore theft before the developer ever calls the library

### npm Payload: trap-core.js

The shared npm payload (`trap-core.js`, 1,149 lines) is a comprehensive credential harvester with active validation and lateral movement capabilities:

1. **Credential scanning**: Sweeps SSH keys, AWS credentials, GitHub tokens, browser profile data, crypto wallet extension data, environment variables, API keys, and developer configuration files
2. **Active validation**: Makes live API calls to AWS and GitHub to verify whether stolen credentials are active and useful — effectively filtering low-value data
3. **Encryption**: Uses Fernet symmetric encryption and ECDH key exchange to protect exfiltrated data
4. **Lateral movement**: Reuses stolen SSH keys to attempt access to additional systems
5. **Persistence and propagation**: Plants persistence through `.cursorrules`, `CLAUDE.md`, Git hooks, shell hooks, systemd services, cron jobs, and SSH

### Crates.io Payload

The `build.rs` scripts in the Rust packages search for local Sui and Move wallet keystores, encrypt discovered data using XOR with the hardcoded key `cargo-build-helper-2026`, and exfiltrate to GitHub Gists.

### AI Assistant Poisoning

A novel capability in TrapDoor: the npm payload writes `.cursorrules` and `CLAUDE.md` files containing hidden instructions embedded with zero-width Unicode characters. These files are designed to deceive AI coding assistants (Cursor IDE, Claude Code) into executing a "security audit" workflow that is actually credential discovery and exfiltration. The technique may not work uniformly across all models, but its presence signals active attacker experimentation with AI development environments as a malware vector.

The attacker also opened pull requests to major AI open source projects — `browser-use/browser-use`, `langchain-ai/langchain`, `langflow-ai/langflow`, `run-llama/llama_index`, `FoundationAgents/MetaGPT`, and `OpenHands/OpenHands` — attempting to introduce campaign-linked `.cursorrules` and `CLAUDE.md` files under benign titles like "docs: add .cursorrules with dev standards and build verification". These PRs referenced the campaign marker `P-2024-001` and pointed to attacker-controlled configuration at `ddjidd564[.]github[.]io/defi-security-best-practices/config.json`.

### Attacker Playbook

The attacker's GitHub Pages repository contained `AUDIT-MATRIX.md`, a design document describing a "Universal AI Agent Extraction Framework" with staged workflows for capability detection, data extraction, self-replication fallback, and telemetry. The document explicitly describes disguising malicious extraction behaviors as benign developer tasks — security audits, wallet safety checks, cloud configuration validation — mirroring the campaign's package naming strategy.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| May 19, 2026 | Earliest confirmed campaign activity (backdated analysis) |
| May 22, 2026 20:20:18 | First widely observed package: `eth-security-auditor@0.1.0` on PyPI |
| May 22–24, 2026 | Packages published in waves across npm, PyPI, and Crates.io |
| May 24, 2026 | Socket publishes TrapDoor disclosure |
| May 24, 2026 | Socket reports malicious packages to all three affected registries |
| Post-disclosure | Packages removed; monitoring continues for new campaign infrastructure |

---

## Detection

```bash
# Check for TrapDoor npm packages
for pkg in async-pipeline-builder build-scripts-utils chain-key-validator \
  crypto-credential-scanner defi-env-auditor defi-threat-scanner \
  deployment-key-auditor dev-env-bootstrapper eth-wallet-sentinel \
  llm-context-compressor mnemonic-safety-check model-switch-router \
  node-setup-helpers project-init-tools prompt-engineering-toolkit \
  solidity-deploy-guard token-usage-tracker wallet-backup-verifier \
  wallet-security-checker web3-secrets-detector workspace-config-loader; do
  if npm ls "$pkg" 2>/dev/null | grep -q "$pkg"; then
    echo "FOUND: $pkg"
  fi
done

# Check for TrapDoor PyPI packages
for pkg in cryptowallet-safety data-pipeline-check defi-risk-scanner \
  env-loader-cli eth-security-auditor git-config-sync solidity-build-guard; do
  pip show "$pkg" 2>/dev/null && echo "FOUND PyPI: $pkg"
done

# Check for TrapDoor Crates.io packages
for pkg in move-analyzer-build move-compiler-tools move-project-builder \
  sui-framework-helpers sui-move-build-helper sui-sdk-build-utils; do
  cargo search "$pkg" 2>/dev/null | grep -q "^$pkg " && echo "CHECK Crates: $pkg"
done

# Check for TrapDoor campaign marker in project files
grep -r "P-2024-001" . 2>/dev/null
grep -r "ddjidd564" . 2>/dev/null
grep -r "defi-security-best-practices" . 2>/dev/null
grep -r "cargo-build-helper-2026" . 2>/dev/null

# Check for malicious AI assistant config files (zero-width Unicode)
grep -rP "[\x{200B}-\x{200D}\x{FEFF}\x{2060}]" .cursorrules CLAUDE.md 2>/dev/null

# Check for trap-core.js payload
find / -name "trap-core.js" -size +40k 2>/dev/null

# Check node_modules for payload
find node_modules -name "trap-core.js" 2>/dev/null
```

---

## Remediation

1. Remove all identified TrapDoor packages from npm, PyPI, and Cargo dependency trees immediately
2. Inspect `.cursorrules` and `CLAUDE.md` files for zero-width Unicode characters or references to `ddjidd564`, `P-2024-001`, or `defi-security-best-practices`
3. Inspect Git hooks (`.git/hooks/`), shell profiles (`~/.bashrc`, `~/.zshrc`, `~/.profile`), systemd user units (`~/.config/systemd/user/`), and crontab (`crontab -l`) for unauthorized entries
4. Rotate all potentially exposed credentials: AWS, GitHub, SSH keys, npm/PyPI tokens, crypto wallet seeds
5. Audit GitHub and AWS API access logs for calls from unexpected sources around May 19–24, 2026
6. Review any open pull requests introducing `.cursorrules` or `CLAUDE.md` files; check for zero-width Unicode characters in proposed changes
7. Lock your Crates.io packages to specific versions; audit `build.rs` scripts in new Rust dependencies
8. Check for outbound connections to `ddjidd564.github.io` or GitHub Gist exfiltration in network logs

---

## Lessons Learned

- **Crates.io is now in scope**: TrapDoor marks the first significant coordinated supply chain attack campaign targeting the Rust package registry. Security tooling must now cover `build.rs` scripts as a malware execution vector.
- **AI assistants are becoming attack surfaces**: Planting malicious `.cursorrules` and `CLAUDE.md` files — in both installed packages and open source PRs — demonstrates attackers are actively experimenting with AI coding tools as a new persistence and propagation vector.
- **Active credential validation raises the stakes**: Validating stolen secrets via live API calls shows campaign maturity — the attacker filters low-value data, improving the efficiency of downstream exploitation.
- **Cross-registry fingerprinting is critical**: Individual malicious packages on Crates.io appeared isolated; only behavioral and infrastructure overlap with npm/PyPI revealed the coordinated campaign.
- **Package names signal intent**: Names like `crypto-credential-scanner`, `defi-threat-scanner`, and `wallet-security-checker` masquerade as defensive tooling — exactly what their target demographic (security-conscious crypto developers) would install.

---

## Related Incidents

- [./2026-04-velora-dex-sdk-backdoor.md](./2026-04-velora-dex-sdk-backdoor.md) — Registry-only npm backdoor also targeting crypto/DeFi developers
- [./2026-04-pgserve-npm-canistersprawl.md](./2026-04-pgserve-npm-canistersprawl.md) — CanisterSprawl worm also used ICP blockchain for exfiltration
- [./2026-01-gwagon-npm-infostealer.md](./2026-01-gwagon-npm-infostealer.md) — npm infostealer targeting 100+ crypto wallets
