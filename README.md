# Supply Chain Attacks — Learning Repository

A living reference for tracking, understanding, and defending against real-world software supply chain attacks. Updated as new incidents are disclosed.

---

## What Is a Supply Chain Attack?

A supply chain attack targets the tools, packages, or infrastructure that developers trust — rather than attacking an application directly. By compromising a widely-used dependency, attackers can reach thousands of downstream users in a single move.

Common vectors include:
- **Malicious package versions** published to npm, PyPI, or similar registries
- **Compromised CI/CD credentials** used to poison build pipelines
- **Tag hijacking** in GitHub Actions or other automation workflows
- **Self-replicating worms** that use stolen tokens to auto-publish new infected versions

---

## 📅 Attack Timeline

| Date | Name | Ecosystem | Impact |
|------|------|-----------|--------|
| Apr 2026 | [GlassWorm Native — Zig Dropper IDE Mass-Infection](./attacks/2026-04-glassworm-zig-dropper.md) | OpenVSX / VS Code / Cursor / Windsurf / VSCodium / Positron | Trojanized WakaTime extension; Zig-compiled native .node dropper silently infects all compatible IDEs; second-stage GlassWorm RAT with Solana C2 |
| Apr 2026 | [@velora-dex/sdk — Registry-Only npm Backdoor, macOS launchctl Persistence](./attacks/2026-04-velora-dex-sdk-backdoor.md) | npm | Import-time execution (no postinstall hook); macOS-only binary via launchctl zsh.profiler; arch-aware ARM64/Intel delivery; C2 89.36.224.5; DeFi toolkit targeting crypto keys |
| Mar 2026 | [axios npm Compromise — Maintainer Account Hijacked, Cross-Platform RAT Deployed](./attacks/2026-03-axios-npm-rat.md) | npm | ~100M weekly downloads; jasonsaayman account hijacked; axios@1.14.1 & 0.30.4 inject plain-crypto-js dropper; cross-platform RAT (macOS/Windows/Linux); 3h exposure |
| Mar 2026 | [IoliteLabs Solidity VSCode Extensions — Dormant Publisher Backdoor](./attacks/2026-03-iolitelabs-vscode-backdoor.md) | VS Code Marketplace | 27,500 installs; dormant 8-year publisher account hijacked; multi-stage backdoor in pako dependency; Chrome-impersonation DLL (Windows) + LaunchAgent (macOS) |
| Mar 2026 | [telnyx PyPI — WAV Steganography Credential Stealer (TeamPCP)](./attacks/2026-03-telnyx-pypi-wav.md) | PyPI | 742K monthly DLs; WAV steganography payload delivery; Linux/macOS infostealer + Windows Startup persistence; same TeamPCP RSA key as litellm |
| Mar 2026 | [litellm PyPI — Credential Stealer Hidden in Wheel](./attacks/2026-03-litellm-pypi-stealer.md) | PyPI | 3-stage attack: mass credential harvester + AES/RSA exfil + persistent C2 + K8s lateral movement |
| Mar 2026 | [Checkmarx KICS GitHub Action Compromised](./attacks/2026-03-checkmarx-kics-action.md) | GitHub Actions | All release tags poisoned with infostealer; credential theft + runner memory dumps + persistence |
| Mar 2026 | [Trivy Second Compromise — Malicious v0.69.4 & Actions Re-Poisoning](./attacks/2026-03-trivy-second-compromise.md) | GitHub Actions / Homebrew | Incomplete first-incident containment; malicious binary + action re-compromise; ~12h exposure |
| Mar 2026 | [CanisterWorm — TeamPCP npm Worm & Kubernetes Wiper](./attacks/2026-03-canisterworm-npm.md) | npm / Kubernetes | 47+ packages; self-propagating ICP C2 worm; destructive K8s wiper targeting Iran |
| Mar 2026 | [GlassWorm Chrome Extension RAT](./attacks/2026-03-glassworm-chrome-rat.md) | npm, PyPI, Chrome | Multi-stage RAT; force-installed Chrome extension; keylogger + hardware wallet phishing |
| Mar 2026 | [React Native Phone Number Packages Compromise](./attacks/2026-03-react-native-compromise.md) | npm | 130K+ monthly DLs; 3-wave account takeover; Solana C2; dependency-chain evasion |
| Mar 2026 | [Polymarket Bot — dev-protocol Org Hijack](./attacks/2026-03-polymarket-devprotocol.md) | npm / GitHub | Hijacked verified org (568 followers); functional trading bot hides wallet stealer + SSH backdoor |
| Mar 2026 | [GlassWorm Returns — Unicode Mass Campaign](./attacks/2026-03-glassworm-returns.md) | GitHub, npm, VS Code | 150+ repos; invisible Unicode injection; shared Solana C2 with ForceMemo |
| Mar 2026 | [ForceMemo](./attacks/2026-03-forcememo.md) | GitHub / Python | 240+ Python repos force-pushed with Solana blockchain C2 infostealer |
| Mar 2026 | [fast-draft OpenVSX — BlokTrooper RAT](./attacks/2026-03-fast-draft-bloktrooper.md) | OpenVSX / VS Code | 26K downloads; full RAT + wallet stealer + file exfil + clipboard monitor |
| Mar 2026 | [bittensor-wallet PyPI Backdoor](./attacks/2026-03-bittensor-pypi.md) | PyPI | 48-hour exposure; Rust-compiled backdoor; triple-redundant C2 + DNS tunneling |
| Mar 2026 | [xygeni-action C2 Backdoor](./attacks/2026-03-xygeni-action.md) | GitHub Actions | 7-day interactive C2 shell via tag poisoning; stolen maintainer credentials |
| Mar 2026 | [kubernetes-el Pwn Request](./attacks/2026-03-kubernetes-el.md) | GitHub Actions / Emacs | GITHUB_TOKEN stolen; rm -rf / destructive payload; removed from MELPA |
| Mar 2026 | [Trivy GitHub Actions Tag Compromise](./attacks/2026-03-trivy-github-actions.md) | GitHub Actions | CI/CD secret theft via 75 poisoned version tags; 45 repos confirmed impacted |
| Mar 2026 | [GlassWorm / CanisterWorm](./attacks/2026-03-glassworm-canisterworm.md) | npm, GitHub, OpenVSX | Browser RAT, crypto theft, Solana C2; 45 packages in <60 sec |
| Feb 2026 | [npm Gambling Backdoor — json-bigint Typosquats with Payment & Outcome Manipulation](./attacks/2026-02-npm-gambling-backdoor.md) | npm | json-bigint-extend/jsonfx/jsonfb typosquats; env-gated activation; Express payment route injection; fixflow gambling outcome manipulator; 30s remote riskCode refresh; Chinese-language RAT panel |
| Feb 2026 | [Cline Supply Chain Attack — cline@2.3.0 Installs OpenClaw](./attacks/2026-02-cline-openclaw.md) | npm | Unauthorized publish; malicious postinstall silently deploys persistent AI agent daemon; ~4K downloads in ~8h |
| Feb 2026 | [hackerbot-claw AI Attack Campaign](./attacks/2026-02-hackerbot-claw.md) | GitHub Actions | AI-powered bot; RCE in 6/7 targets; full Trivy repo takeover + releases deleted |
| Feb 2026 | [Fake ClawdBot Agent VS Code Extension — ScreenConnect RAT](./attacks/2026-02-clawdbot-vscode-screenconnect.md) | VS Code Marketplace | Brand impersonation of Clawdbot AI assistant; functional trojan deploys ScreenConnect RAT on VS Code startup; Windows-only; quick Microsoft takedown |
| Jan 2026 | [dYdX npm & PyPI Compromise — Wallet Stealer + RAT](./attacks/2026-01-dydx-npm-pypi.md) | npm / PyPI | Compromised publishing credentials; wallet seed phrase theft + RAT; 121K downloads affected; C2 pre-staged 18 days before attack |
| Jan 2026 | [G_Wagon — npm Infostealer Targeting 100+ Crypto Wallets](./attacks/2026-01-gwagon-npm-infostealer.md) | npm | ansi-universal-ui fake UI library; self-dependency double-execution; memory-only Python payload; browser creds + 100+ crypto wallets + cloud keys → Appwrite C2 |
| Jan 2026 | [spellcheckpy / spellcheckerpy — PyPI RAT via Hidden Dictionary Payload](./attacks/2026-01-spellcheckpy-pypi-rat.md) | PyPI | Typosquats of pyspellchecker; payload hidden in Basque dictionary .gz file; C2 on nation-state-linked Cloudzy infrastructure; ~1K downloads |
| Jan 2026 | [Gone Phishin' — npm Packages as CDN for Industrial Spear-Phishing](./attacks/2026-01-gone-phishin-npm-phishing.md) | npm / jsDelivr | Novel CDN-abuse vector; jsDelivr mirrors used to serve per-victim phishing kits targeting aerospace/energy firms; no developer machine compromise required |
| Dec 2025 | [Maven Central Jackson Typosquatting — First Sophisticated Malware on Maven Central](./attacks/2025-12-maven-central-jackson-typosquat.md) | Maven Central | org. vs com. namespace typosquat of jackson-databind; Spring Boot autoconfiguration auto-exec; AES-encrypted C2; Windows svchosts.exe + macOS RAT; LLM prompt injection evasion; 1.5h exposure |
| Dec 2025 | [NeoShadow — MSBuild & Blockchain npm Backdoor Campaign](./attacks/2025-12-neoshadow-npm.md) | npm | Typosquatting; MSBuild LOLbin + ChaCha20/Curve25519 C2; 0 VirusTotal detections on Wave 2 binary; blockchain-linked C2 resilience |
| Nov 2025 | [Shai-Hulud Worm Wave 2](./attacks/2025-late-shai-hulud-worm.md) | npm, GitHub, OpenVSX | 621 packages, 25K+ repos, 14K secrets; Pwn Request via asyncapi |
| Sep 2025 | [The Great npm Heist](./attacks/2025-09-great-npm-heist.md) | npm | 18 foundational packages (2B+ weekly DLs); browser crypto hijacking |
| Sep 2025 | [Shai-Hulud Worm Wave 1](./attacks/2025-late-shai-hulud-worm.md) | npm | 187+ packages; TruffleHog-validated credential theft; $50M crypto stolen |
| Aug 2025 | [NX Build System Compromise](./attacks/2025-08-nx-build-system.md) | npm, VS Code | 1.4K+ devs; wallets + tokens stolen via postinstall + IDE auto-update |
| Aug 2025 | [Info-Stealer Campaign](./attacks/2025-08-infostealer-campaign.md) | npm | Chrome credential theft via 8 fake React/Solana packages |
| Jul 2025 | [npm 'is' Package Compromise — Phishing-Driven Account Takeover](./attacks/2025-07-is-package-compromise.md) | npm | Phishing of former maintainer account; social engineering re-adds hijacked account; millions of weekly DLs at risk; ~6h exposure |
| Jul 2025 | [eslint-config-prettier / JounQin Phishing Campaign](./attacks/2025-07-eslint-config-prettier-phishing.md) | npm | Phishing via npnjs[.]com lookalike domain; 7 packages compromised (eslint-config-prettier, eslint-plugin-prettier, got-fetch + others); Windows DLL dropper + Pycoon infostealer; CVE-2025-54313 |
| Apr 2025 | [XRP Ledger Supply Chain Attack](./attacks/2025-04-xrp-supply-chain.md) | npm | Official xrpl SDK backdoored; private keys exfiltrated for 16 hours |
| Mar 2025 | [tj-actions/changed-files Compromise](./attacks/2025-03-tj-actions.md) | GitHub Actions | All version tags poisoned; CI secrets leaked to logs in 23K+ repos |

---

## 📁 Repository Structure

```
/
├── README.md                        ← You are here
├── CONTRIBUTING.md                  ← How to add a new attack entry
├── resources.md                     ← Detection tools, references, further reading
└── attacks/
    ├── 2026-04-glassworm-zig-dropper.md
    ├── 2026-04-velora-dex-sdk-backdoor.md
    ├── 2026-03-axios-npm-rat.md
    ├── 2026-03-iolitelabs-vscode-backdoor.md
    ├── 2026-03-telnyx-pypi-wav.md
    ├── 2026-03-litellm-pypi-stealer.md
    ├── 2026-03-checkmarx-kics-action.md
    ├── 2026-03-trivy-second-compromise.md
    ├── 2026-03-canisterworm-npm.md
    ├── 2026-03-glassworm-chrome-rat.md
    ├── 2026-03-react-native-compromise.md
    ├── 2026-03-polymarket-devprotocol.md
    ├── 2026-03-glassworm-returns.md
    ├── 2026-03-forcememo.md
    ├── 2026-03-fast-draft-bloktrooper.md
    ├── 2026-03-bittensor-pypi.md
    ├── 2026-03-xygeni-action.md
    ├── 2026-03-kubernetes-el.md
    ├── 2026-03-trivy-github-actions.md
    ├── 2026-03-glassworm-canisterworm.md
    ├── 2026-02-npm-gambling-backdoor.md
    ├── 2026-02-cline-openclaw.md
    ├── 2026-02-hackerbot-claw.md
    ├── 2026-02-clawdbot-vscode-screenconnect.md
    ├── 2026-01-dydx-npm-pypi.md
    ├── 2026-01-gwagon-npm-infostealer.md
    ├── 2026-01-spellcheckpy-pypi-rat.md
    ├── 2026-01-gone-phishin-npm-phishing.md
    ├── 2025-12-maven-central-jackson-typosquat.md
    ├── 2025-12-neoshadow-npm.md
    ├── 2025-late-shai-hulud-worm.md
    ├── 2025-09-great-npm-heist.md
    ├── 2025-08-nx-build-system.md
    ├── 2025-08-infostealer-campaign.md
    ├── 2025-07-is-package-compromise.md
    ├── 2025-07-eslint-config-prettier-phishing.md
    ├── 2025-04-xrp-supply-chain.md
    └── 2025-03-tj-actions.md
```

---

## 🛡️ Quick Incident Response Checklist

If you suspect a compromised package in your environment:

1. Run `npm ls <package-name>` to confirm the installed version
2. Cross-reference against the attack entries in this repo
3. If compromised: remove the package immediately
4. Audit your environment for any exfiltrated secrets (env vars, SSH keys, cloud credentials)
5. **Rotate all active credentials** — assume anything in your CI/CD environment is compromised
6. Check your CI/CD logs for unexpected network calls or process spawning

---

## 🤝 Contributing

Found a new supply chain incident? See [CONTRIBUTING.md](./CONTRIBUTING.md) for the standard template and submission guidelines. This repo is only as current as its contributors make it.

---

## 📚 Further Reading

See [resources.md](./resources.md) for detection tools, monitoring services, and curated reading on supply chain security.
