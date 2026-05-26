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
| May 2026 | [Laravel-Lang Supply Chain Attack — 4 Composer Packages, Every Tag Rewritten, PHP 2-Stage Credential Stealer](./attacks/2026-05-laravel-lang-packagist.md) | Packagist / Composer (PHP) | All 502+ tags across laravel-lang/lang, http-statuses, actions, attributes force-pushed to malicious commits; autoload.files hijack fires on boot; 5,900-line PHP stealer targets cloud creds, browser passwords, crypto wallets, VPNs; AES-256 exfil to flipboxstudio.info |
| May 2026 | [Megalodon — Mass GitHub Actions d-PPE Campaign Backdoors 5,561 Repositories](./attacks/2026-05-megalodon-github-actions.md) | GitHub Actions / npm | 5,561 repos backdoored in 6 hours via direct push to unprotected default branches; SysDiag/Optimize-Build workflow names; OIDC + cloud + SSH + npm token theft; @tiledesk/tiledesk-server v2.18.6–2.18.12 poisoned on npm; C2 216.126.225.129:8443 |
| May 2026 | [Nx Console VS Code Extension Compromised — Orphan Commit Dropper, Sigstore Forgery & GitHub Breach](./attacks/2026-05-nx-console-vscode.md) | VS Code Marketplace / npm / GitHub | nrwl.angular-console v18.95.0 (2.2M installs); 11-min exposure; stolen contributor token; orphan commit payload; 8-stage attack incl. Sigstore forgery + Python C2 backdoor; linked to exfiltration of 3,800 GitHub internal repos |
| May 2026 | [Microsoft durabletask PyPI Compromise — TeamPCP rope.pyz Modular Cloud Intrusion Framework](./attacks/2026-05-durabletask-pypi.md) | PyPI | 3 malicious versions (1.4.1–1.4.3) of Microsoft's Azure Durable Functions SDK (400K monthly DLs); 28KB rope.pyz with 7 parallel cloud collectors; AWS SSM + K8s lateral movement; geotargeted rm -rf wiper; TeamPCP t.m-kosche.com C2 |
| May 2026 | [Mini Shai-Hulud Wave 4 — AntV/atool npm Account Compromise, 323 Packages, Runner Memory Scraping](./attacks/2026-05-antv-npm-shai-hulud.md) | npm | 323 packages across @antv, timeago.js (1.5M DLs), echarts-for-react, jest-canvas-mock; optionalDependencies git ref bypasses static analysis; Runner.Worker memory scraping; 2,500+ exfil repos created; Claude Code + VS Code persistence backdoors; TeamPCP t.m-kosche.com C2 |
| May 2026 | [actions-cool/issues-helper GitHub Action Compromised — All 53 Tags Moved to Imposter Commits](./attacks/2026-05-actions-cool-issues-helper.md) | GitHub Actions | 53 tags + 15 tags (maintain-one-comment) moved to imposter commits in <4 min; Bun + Runner.Worker memory scraping; t.m-kosche.com exfil; TeamPCP attribution; simultaneous with @antv + durabletask attacks |
| May 2026 | [node-ipc npm Infostealer — DNS-Exfiltration Credential Stealer](./attacks/2026-05-node-ipc-npm-infostealer.md) | npm | 3 malicious versions (9.1.6, 9.2.3, 12.0.1); ~2h exposure; DNS-only exfiltration to bt.node.js; targets cloud creds, SSH, AI tools (Claude, Kiro), Salesforce, M365; no disk persistence post-exfil |
| May 2026 | [Mini Shai-Hulud Wave 3 — TanStack + Multi-Namespace npm Worm (TeamPCP)](./attacks/2026-05-mini-shai-hulud-tanstack-npm.md) | npm / PyPI | 373 malicious package-versions across 169 npm names + 2 PyPI packages; @tanstack/react-router (~12M weekly DLs), @uipath, @mistralai, @squawk compromised; Pwn Request + cache poisoning + OIDC extraction; Session network exfil (takedown-resistant); gh-token-monitor persistence daemon; cross-ecosystem npm→PyPI worm propagation |
| May 2026 | [QLNX — Quasar Linux RAT Targets Developer Workstations to Enable Supply Chain Compromise](./attacks/2026-05-qlnx-quasar-linux-rat.md) | Linux Developer Environments | Full-featured Linux RAT (58-command C2, dual LD_PRELOAD+eBPF rootkit, PAM backdoor with master password, 7 persistence mechanisms); harvests npm/PyPI publish tokens + cloud creds to enable downstream package poisoning; only 4/70+ AV detections at disclosure |
| May 2026 | [CVE-2026-31431 "Copy Fail" — Linux Kernel LPE Breaks CI/CD Pipelines and Kubernetes Containers](./attacks/2026-05-cve-2026-31431-copy-fail.md) | Linux / Kubernetes / CI/CD | Deterministic LPE in Linux kernels 4.14–6.19.12; 732-byte Python PoC; container escape on all major distros since 2017; shared CI/CD runners fully compromised; preliminary wild exploitation observed |
| Apr 2026 | [GenAI Chrome Extension RAT Campaign — 18 Malicious AI Extensions Exfiltrate Developer Secrets](./attacks/2026-04-genai-chrome-extension-campaign.md) | Chrome Web Store | 18 malicious extensions (95K+ users): MCP-themed RAT with remote JS exec, Gmail AitB exfiltration, Claude/OpenAI/Gemini API key theft, persistent cross-device search hijacker, PAC-script proxy spyware; AI-generated malware code detected |
| Apr 2026 | [Mini Shai-Hulud Expands to PHP — intercom/intercom-php@5.0.2 Packagist Compromise](./attacks/2026-04-intercom-php-packagist.md) | Packagist / Composer (PHP) | Same-day cross-ecosystem blitz with lightning PyPI + intercom-client npm; Composer plugin injection executes Bun + router_runtime.js at install time; GitHub tokens, SSH keys, cloud creds exfil to zero.masscan.cloud; first Mini Shai-Hulud expansion into PHP ecosystem |
| Apr 2026 | [Mini Shai-Hulud Wave — SAP npm Packages + intercom-client Multi-Cloud Worm](./attacks/2026-04-mini-shai-hulud-sap-npm.md) | npm | mbt@1.2.48 + 3 @cap-js packages; CircleCI PR steals CLOUD_MTA_BOT_NPM_TOKEN; OIDC publishing abuse; 29h later intercom-client@7.0.4 (361K DLs) compromised; Bun v1.3.13 loader + 11.7MB execution.js/router_runtime.js; AWS/GCP/Azure/K8s credential sweep; AI agent persistence via .claude/settings.json + .vscode/tasks.json; propagation via OhNoWhatsGoingOnWithGitHub dead-drop |
| Apr 2026 | [lightning (PyTorch Lightning) PyPI Compromise — Mini Shai-Hulud in AI/ML Ecosystem](./attacks/2026-04-lightning-pypi-shai-hulud.md) | PyPI | lightning 2.6.2 & 2.6.3; import-time daemon thread (bypasses --ignore-scripts); hidden _runtime/router_runtime.js; steals GPU cluster creds, HuggingFace/W&B tokens, cloud IAM; npm tarball poisoning for cross-ecosystem spread; GitHub account compromise used to suppress disclosure |
| Apr 2026 | [Fake tanstack npm — .env File Exfiltration via Svix Webhook Relay](./attacks/2026-04-fake-tanstack-npm.md) | npm | 4 versions in 27 minutes (2.0.4–2.0.7); postinstall sweeps all .env* files; exfil via legitimate Svix webhook relay; live attacker iteration; ~19K monthly downloads; name-squatting on @tanstack org |
| Apr 2026 | [elementary-data PyPI & GHCR Compromise — GitHub Actions Script Injection Forges Signed Release](./attacks/2026-04-elementary-data-pypi-ghcr.md) | PyPI / GHCR / GitHub Actions | `elementary-data==0.23.3` + `:latest` GHCR image; PR comment script injection → GITHUB_TOKEN abuse → forged signed release commit; `.pth` fires on every Python invocation; 3-stage XOR credential harvester; AWS Secrets Manager live API exfil; C2 `skyhanni.cloud` |
| Apr 2026 | [Bitwarden CLI Shai-Hulud Third Coming — MCP-Aware Credential Worm via Compromised CI/CD](./attacks/2026-04-bitwarden-cli-shai-hulud.md) | npm | `@bitwarden/cli@2026.4.0`; CI/CD pipeline compromise bypasses trusted publishing; steals Claude Code auth tokens + MCP server configs; cloud secret manager harvest; Shai-Hulud worm propagation; same TeamPCP C2 as Checkmarx KICS Docker attack |
| Apr 2026 | [Checkmarx KICS Second Compromise — Docker Hub Images, VS Code Extensions, GitHub Actions Worm](./attacks/2026-04-checkmarx-kics-docker-vscode.md) | Docker Hub / VS Code / GitHub Actions / npm | TeamPCP's second Checkmarx attack; malicious KICS images exfiltrate IaC scan secrets; mcpAddon.js via backdated orphaned GitHub commit; GitHub Actions `toJSON(secrets)` heist; npm worm propagation; ~84-min Docker exposure window |
| Apr 2026 | [GPT-Proxy Backdoor — kube-health-tools & kube-node-health](./attacks/2026-04-gpt-proxy-kube-health-backdoor.md) | npm / PyPI | Native binary droppers; Go RAT with Chisel reverse tunnels; OpenAI-compatible LLM proxy routes AI traffic through compromised servers to Chinese resellers; self-deleting |
| Apr 2026 | [TeamPCP xinference PyPI Compromise — Two-Stage Credential Stealer](./attacks/2026-04-xinference-pypi-teampcp.md) | PyPI | xinference 2.6.0–2.6.2; `__init__.py` injection; exfil to `whereisitat.lucyatemysuperbox.space`; SSH keys, cloud creds, Kubernetes tokens, crypto wallets; TeamPCP attribution |
| Apr 2026 | [CanisterSprawl — pgserve npm Compromise](./attacks/2026-04-pgserve-npm-canistersprawl.md) | npm | pgserve 1.1.11–1.1.13; 1,143-line postinstall worm; RSA-4096+AES-256 encryption; ICP blockchain canister exfil (untakedownable); self-propagates to all publishable npm & PyPI packages |
| Apr 2026 | [prt-scan — AI-Augmented GitHub Actions Credential Theft Campaign](./attacks/2026-04-prt-scan-github-actions.md) | GitHub Actions / npm | 6 attacker accounts; 500+ malicious PRs; pull_request_target exploitation; 5-phase CI secret theft; AI-generated language-aware payloads; npm packages backdoored |
| Apr 2026 | [Strapi CMS npm Typosquats — Redis/PostgreSQL RCE, Docker Escape, Persistent Implant](./attacks/2026-04-strapi-npm-typosquats.md) | npm | 36 fake Strapi plugins; 4 sock-puppet accounts; 8-stage payload evolution: Redis RCE → Docker escape → DB exploitation → persistent implant targeting crypto exchange |
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
| Sep 2025 | [chalk / debug / color npm Cryptostealer — Phished Contributor, 2.6B Weekly Downloads](./attacks/2025-09-chalk-debug-npm-cryptostealer.md) | npm | 18 packages + duckdb follow-up; single contributor phished; browser crypto hijacking across 5 blockchains; <1h exposure for most |
| Oct 2025 | [postmark-mcp — First Confirmed Malicious MCP Server on npm](./attacks/2025-10-postmark-mcp-bcc-injection.md) | npm / MCP | Typosquatted Postmark MCP server; 15 clean versions before silent BCC injection; intercepts AI-agent-sent security emails |
| Sep 2025 | [The Great npm Heist](./attacks/2025-09-great-npm-heist.md) | npm | 18 foundational packages (2B+ weekly DLs); browser crypto hijacking |
| Sep 2025 | [GhostAction Campaign — CI/CD Secret Theft via Malicious Workflows](./attacks/2025-09-ghostaction-campaign.md) | GitHub Actions | 327 accounts compromised; 817 repos; 3,325 secrets (npm/PyPI/DockerHub/AWS) exfiltrated via fake "security" workflow |
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
    ├── 2026-05-laravel-lang-packagist.md
    ├── 2026-05-megalodon-github-actions.md
    ├── 2026-05-nx-console-vscode.md
    ├── 2026-05-durabletask-pypi.md
    ├── 2026-05-antv-npm-shai-hulud.md
    ├── 2026-05-actions-cool-issues-helper.md
    ├── 2026-05-node-ipc-npm-infostealer.md
    ├── 2026-05-mini-shai-hulud-tanstack-npm.md
    ├── 2026-05-qlnx-quasar-linux-rat.md
    ├── 2026-05-cve-2026-31431-copy-fail.md
    ├── 2026-04-genai-chrome-extension-campaign.md
    ├── 2026-04-intercom-php-packagist.md
    ├── 2026-04-mini-shai-hulud-sap-npm.md
    ├── 2026-04-lightning-pypi-shai-hulud.md
    ├── 2026-04-fake-tanstack-npm.md
    ├── 2026-04-elementary-data-pypi-ghcr.md
    ├── 2026-04-bitwarden-cli-shai-hulud.md
    ├── 2026-04-checkmarx-kics-docker-vscode.md
    ├── 2026-04-gpt-proxy-kube-health-backdoor.md
    ├── 2026-04-xinference-pypi-teampcp.md
    ├── 2026-04-pgserve-npm-canistersprawl.md
    ├── 2026-04-prt-scan-github-actions.md
    ├── 2026-04-strapi-npm-typosquats.md
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
    ├── 2025-10-postmark-mcp-bcc-injection.md
    ├── 2025-09-chalk-debug-npm-cryptostealer.md
    ├── 2025-late-shai-hulud-worm.md
    ├── 2025-09-great-npm-heist.md
    ├── 2025-09-ghostaction-campaign.md
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
