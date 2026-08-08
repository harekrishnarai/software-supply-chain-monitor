# Coordinated AsyncAPI Supply Chain Attack — Miasma v3 RAT via Compromised CI/CD Pipelines

**Date:** July 2026
**Ecosystem:** npm / GitHub Actions
**Severity:** Critical
**Type:** CI/CD Pipeline Compromise / Worm / RAT (load-time dropper, not install-time)
**Sources:**
- [StepSecurity — Coordinated AsyncAPI Supply Chain Attack: Miasma RAT Delivered via Compromised CI/CD Pipelines in Two Repositories](https://www.stepsecurity.io/blog/compromised-next-branch-pushes-malicious-asyncapi-generator-generator-helpers-and-generator-components-to-npm)

---

## Summary

On July 14, 2026, a coordinated supply chain attack hit two separate AsyncAPI GitHub repositories — `asyncapi/generator` and `asyncapi/spec-json-schemas` — producing four malicious npm packages that collectively cover the official AsyncAPI code-generation toolchain and its canonical JSON Schema definitions. The attack marks a significant escalation of the Miasma campaign: the payload is now self-identified as **Miasma v3**, features a metamorphic mutation engine, six independent C2 channels (including Nostr and an Ethereum blockchain dead-drop), and a built-in AI tool poisoning module targeting developer AI coding assistants.

The attacker's technique is notable for what was *not* stolen: no npm token. Instead, the attacker obtained push credentials to both repositories and used each project's own legitimate GitHub Actions OIDC trusted-publisher pipeline to publish the poisoned packages. The resulting tarballs carry **valid OIDC provenance attestations** — signed by Sigstore/Fulcio, recording the correct workflow file, branch, and commit SHA — because the attacker let the real CI do the work. This demonstrates that SLSA provenance proves only that an authorized workflow ran, not that the triggering commits were legitimate.

The dropper is not a postinstall hook. It fires at `require()` time — meaning **it executes the first time any downstream application actually imports the library**, not when a developer runs `npm install`. This side-steps warnings about lifecycle scripts and means the payload reaches production workloads, not just developer machines. StepSecurity independently confirmed C2 connections to `85.137.53.71` by executing the payload chain inside a monitored, isolated GitHub Actions environment.

---

## Compromised Artifacts

| Package | Malicious Version | Last Safe Version | Repository |
|---------|------------------|-------------------|------------|
| `@asyncapi/generator` | 3.3.1 | 3.3.0 | asyncapi/generator (`next` branch) |
| `@asyncapi/generator-helpers` | 1.1.1 | 1.1.0 | asyncapi/generator (`next` branch) |
| `@asyncapi/generator-components` | 0.7.1 | 0.7.0 | asyncapi/generator (`next` branch) |
| `@asyncapi/specs` | 6.11.2-alpha.1, 6.11.2 | 6.11.1 | asyncapi/spec-json-schemas (`master` branch) |

Injected files within each package:
- `apps/generator/lib/templates/config/validator.js` → ships in `@asyncapi/generator@3.3.1`
- `packages/helpers/src/utils.js` → ships in `@asyncapi/generator-helpers@1.1.1`
- `packages/components/src/utils/ErrorHandling.js` (babel-compiled to `lib/`) → ships in `@asyncapi/generator-components@0.7.1`
- `index.js` (ESM/TypeScript compiled build output) → ships in `@asyncapi/specs@6.11.2`

---

## How It Worked

### Entry Point: Compromised Push Credentials (Not Stolen npm Token)

Both attacks used the same placeholder git identity: `"Your Name" <you@example.com>`, GitHub login `invalid-email-address`. This is the unconfigured git default — consistent with a stolen automation token or compromised CI secret, not a legitimate contributor. GitHub could not map the pushing account to a real user.

**Attack 1 — asyncapi/generator (`next` branch), 06:58 UTC:**
Malicious commit `3eab3ec9304aa26081358330491d3cfeb55cc245` was pushed directly to the `next` branch at 06:58:42 UTC. Twelve seconds later, the repository's `release-with-changesets.yml` workflow triggered (Actions run `29313420558`) and published all three generator packages via npm OIDC at 07:10 UTC.

**Attack 2 — asyncapi/spec-json-schemas (`master` branch), 07:51–08:28 UTC:**
The `if-nodejs-release.yml` workflow fires on any push to `master` whose commit message starts with `fix:` or `feat:`. The attacker pushed a series of commits exploiting this trigger:

| Commit | Time (UTC) | Message | Effect |
|--------|-----------|---------|--------|
| `36269ce81837` | 07:56:11 | "fix: correct JSON schema" | Injected dropper into index.js (padded with ~1000 leading spaces to hide in diff) |
| `49cb17a9f920` | 07:56:14 | "6.13.5" | Version bump; did not trigger publish path |
| `61a930fca724` | 08:04:02 | "fix: correct JSON schema" | Published `@asyncapi/specs@6.11.2-alpha.1` at 08:06 UTC |
| `689f5b96693a` | 08:28:02 | "fix: patch parser version" | Added blank line to package.json; published `@asyncapi/specs@6.11.2` at 08:30 UTC |

### Payload Mechanics

The injected dropper is a ~7.7KB `obfuscator.io`-obfuscated block, padded with roughly a thousand leading spaces to push it off-screen in diff views. It uses a two-layer cipher decoded via a string-array-and-rotation scheme. **There is no `preinstall`/`postinstall`/`install` script** in any affected `package.json`. The dropper fires purely on `require()` during normal library use.

**Stage 1 — fires on `require()`:**
```javascript
spawn("node", ["-e", <stage2>], {
  detached: true,
  stdio: "ignore",
  windowsHide: true
}).unref()
```
The original Node process continues normally (no visible error), while the dropper spawns a hidden, detached child.

**Stage 2 — IPFS-hosted downloader:**
The detached child creates a hidden OS-specific directory disguised as a Node.js runtime folder, then fetches `sync.js` from a public IPFS gateway:

| OS | Drop Path | IPFS CID (generator attack) |
|----|----------|------------------------------|
| Linux | `~/.local/share/NodeJS/sync.js` | `QmQobZSp1wRPrpSEQ56qnyq7ecZh5Bg5k1fnjt4SUwwHb9` |
| macOS | `~/Library/Application Support/NodeJS/sync.js` | (same) |
| Windows | `%LOCALAPPDATA%\NodeJS\sync.js` | (same) |

The specs attack used a different CID: `Qmet4fhsAaWMBUxNDfREHwgiyDeSWy4YSYs9wiKUW5jGyf`. Both resolve to the same Miasma v3 payload.

**Stage 3 — Miasma v3 (`sync.js`):**
`sync.js` is decrypted via AES-256-GCM with a key derived from HKDF-SHA256 (key material `rt-file-key-material-v1`, info string `rt-file-key`), then a ROT cipher (shift=4, delta=90, printable ASCII range 33–126) decodes the intermediate form. The decrypted output is a **3.08MB bundled Node.js application**, self-identified as **Miasma v3** with a metamorphic header `// mutated v3 profile=low runtime=1`.

### C2 Infrastructure (from Baked Config)

The baked config blob is encrypted with `HKDF(sha256, rt-vault-master-key-32b-aaaaaaaa, "", rt-baked-key, 32)`:

| Field | Value | Role |
|-------|-------|------|
| `c2Server` | `http://85.137.53.71:8080` | Primary C2 (commands) |
| `uploadServer` | `http://85.137.53.71:8081` | Credential exfiltration |
| `c2ProxyMgmt` | `http://85.137.53.71:8091` | Proxy management |
| `nostrRelays` | `wss://relay.damus.io`, `wss://relay.nostr.com/` | Decentralized C2 via Nostr |
| `blockchain.contractAddress` | `0x12c37A86a0Ed0beBe5d1d6a43E42f07860eAc710` | Ethereum mainnet dead-drop |
| `dhtBootstrap` | `router.bittorrent.com:6881`, `dht.transmissionbt.com:6881` | BitTorrent DHT rendezvous |
| `target.name` | `miasma-train-p1` | Campaign identifier |
| `maxGen` | `4` | Maximum worm propagation generations |
| `batch.defaultStrategy` | `CANARY` | Deploy to 5% of targets first, then waves of 100 |

### RAT Capabilities (Miasma v3 Modules)

The decrypted 3.08MB bundle exposes the following source-path namespaces, revealing a professionally engineered botnet framework:

**Six independent C2 channels** (channel-orchestrator.js coordinates failover): HTTP REST, Nostr relay, IPFS, BitTorrent DHT, libp2p GossipSub P2P mesh, and Ethereum blockchain. Blocking any single endpoint does not sever the connection.

**Credential harvesting** (recon/credential-harvester.js, recon/token-vault.js): browser Login Data / Cookies / Local State (Chrome, Brave, Edge); SSH keys (`~/.ssh/id_rsa`, `id_ed25519`); `~/.npmrc`, `~/.gitconfig`, GitHub CLI config; AWS credentials; `kube/config`; Docker credentials; macOS Keychain; `GITHUB_TOKEN`, `NPM_TOKEN`, `PYPI_TOKEN`.

**AI tool poisoning** (recon/ai-tool-poisoner.js): capability to inject malicious context into AI coding assistant sessions (Claude Code, GitHub Copilot, Cursor, and others). Any AI-assisted code generated on a compromised machine should be treated as potentially tainted.

**LAN lateral movement** (comm/subnet-scan.js, comm/lan-discovery.js, comm/mdns-channel.js): enumerate and spread across the local network beyond the initial host.

**Metamorphic engine** (metamorphic/mutation-engine.js): re-obfuscates each propagated copy. The `// mutated v3 profile=low runtime=1` header confirms this is generation 3 of a polymorphic lineage.

**Worm propagation** (propagate/): five modules for build-toolchain injection and runtime injection. Baked config sets `propagate.npm: false` for this deployment — targeted delivery rather than mass npm worm spread.

**Persistence** (deadman/real-persistence-writer.js): systemd, crontab, macOS launchd, Windows Registry autostart.

**Deadman / self-destruct** (deadman/safe-wipe.js): securely erases evidence and terminates the implant on command or triggered condition.

**Shell execution** (comm/shell-executor.js): arbitrary command execution via C2.

**Cryptographic bot authentication**: a two-entry EC public key chain (`__RT_SPAWN_CHAIN_B64__`) rooted at attacker public key `0432fa4b...` signs individual bot instances with delegated sub-keys, allowing the C2 operator to authenticate each bot without revealing the root key.

---

## Timeline

| Time (UTC) | Event |
|-----------|-------|
| 06:58:42 | Malicious commit `3eab3ec9304aa26081358330491d3cfeb55cc245` pushed to `asyncapi/generator` `next` branch |
| 07:10 | `@asyncapi/generator@3.3.1`, `@asyncapi/generator-helpers@1.1.1`, `@asyncapi/generator-components@0.7.1` published via OIDC trusted publisher |
| 07:51 | Second attack begins on `asyncapi/spec-json-schemas` `master` branch |
| 07:56:11 | Commit `36269ce81837` injects dropper into `index.js` (padded with ~1000 leading spaces) |
| 08:04:02 | Commit `61a930fca724` triggers workflow publish path |
| 08:06 | `@asyncapi/specs@6.11.2-alpha.1` published |
| 08:28:02 | Commit `689f5b96693a` (blank line to package.json) triggers final publish |
| 08:30 | `@asyncapi/specs@6.11.2` published |
| Jul 14 | StepSecurity publishes disclosure; Harden-Runner runtime analysis confirms C2 connections to `85.137.53.71` |

---

## IOCs

| Indicator | Type | Notes |
|-----------|------|-------|
| `85.137.53.71` | C2 IP | Ports 8080 (commands), 8081 (exfil), 8091 (proxy mgmt); StepSecurity-confirmed |
| `0x12c37A86a0Ed0beBe5d1d6a43E42f07860eAc710` | Ethereum contract | Blockchain dead-drop C2 |
| `QmQobZSp1wRPrpSEQ56qnyq7ecZh5Bg5k1fnjt4SUwwHb9` | IPFS CID | sync.js for generator attack |
| `Qmet4fhsAaWMBUxNDfREHwgiyDeSWy4YSYs9wiKUW5jGyf` | IPFS CID | sync.js for specs attack |
| `3eab3ec9304aa26081358330491d3cfeb55cc245` | Git commit | Malicious commit — asyncapi/generator |
| `36269ce81837`, `61a930fca724`, `689f5b96693a` | Git commits | Malicious commits — asyncapi/spec-json-schemas |
| `router.bittorrent.com:6881` | DHT node | BitTorrent DHT bootstrap |
| `dht.transmissionbt.com:6881` | DHT node | BitTorrent DHT bootstrap |
| `wss://relay.damus.io` | Nostr relay | Decentralized C2 channel |
| `wss://relay.nostr.com/` | Nostr relay | Decentralized C2 channel |
| `miasma-train-p1` | Campaign tag | Baked campaign identifier |
| `rt-file-key-material-v1` | HKDF key material | Stage 3 decryption constant |
| `~/.local/share/NodeJS/sync.js` | File path (Linux) | Dropped payload |
| `~/Library/Application Support/NodeJS/sync.js` | File path (macOS) | Dropped payload |
| `%LOCALAPPDATA%\NodeJS\sync.js` | File path (Windows) | Dropped payload |

---

## Detection

```bash
# 1. Check for malicious package versions installed
npm ls @asyncapi/generator @asyncapi/generator-helpers @asyncapi/generator-components @asyncapi/specs 2>/dev/null | grep -E "3\.3\.1|1\.1\.1|0\.7\.1|6\.11\.2"

# 2. Check for the dropped payload on disk
ls -la ~/.local/share/NodeJS/sync.js 2>/dev/null         # Linux
ls -la ~/Library/Application\ Support/NodeJS/sync.js 2>/dev/null  # macOS

# 3. Check for outbound connections to the C2 IP
ss -tnp | grep 85.137.53.71       # Linux (live)
lsof -i | grep 85.137.53.71       # macOS (live)
# In historical network logs:
grep "85.137.53.71" /var/log/syslog 2>/dev/null
grep "85.137.53.71" ~/.bash_history 2>/dev/null

# 4. Check for connections to IPFS gateways (may indicate stage 2 download)
grep -r "ipfs.io" ~/.local/share/NodeJS/ 2>/dev/null

# 5. Check for BitTorrent DHT bootstrap connections (unexpected in developer environments)
ss -tnp | grep -E "6881"

# 6. Check for persistence entries
systemctl --user list-units | grep NodeJS 2>/dev/null   # Linux systemd
crontab -l | grep NodeJS 2>/dev/null                     # Linux/macOS crontab
launchctl list | grep NodeJS 2>/dev/null                 # macOS launchd

# 7. Verify git commit identity on AsyncAPI repos (check for placeholder identity)
git log --format="%H %ae %an" | grep "you@example.com"

# 8. Verify npm provenance attestation commit SHA against known-good commits
npm audit signatures @asyncapi/generator @asyncapi/specs

# 9. Scan process list for hidden/detached Node processes writing to NodeJS directory
ps aux | grep node | grep -v grep

# 10. Check for AI assistant config poisoning
grep -r "85.137.53.71\|miasma-train" ~/.claude/ ~/.cursor/ ~/.config/copilot/ 2>/dev/null
```

---

## Remediation

1. **Immediately downgrade** all four affected packages to their last safe versions:
   ```bash
   npm install @asyncapi/generator@3.3.0 @asyncapi/generator-helpers@1.1.0 @asyncapi/generator-components@0.7.0 @asyncapi/specs@6.11.1
   ```

2. **Remove the dropped payload** from all developer machines and CI runners:
   ```bash
   rm -f ~/.local/share/NodeJS/sync.js        # Linux
   rm -f ~/Library/Application\ Support/NodeJS/sync.js  # macOS
   # Windows: del %LOCALAPPDATA%\NodeJS\sync.js
   ```

3. **Assume full compromise** of any machine that imported an affected package version after 07:10 UTC on July 14, 2026. The dropper fires on `require()`, not `npm install`.

4. **Rotate all credentials** accessible on affected machines: SSH keys, `~/.npmrc` tokens, `~/.gitconfig`, GitHub tokens, AWS credentials, kubeconfig, Docker credentials, macOS Keychain entries, all browser-stored passwords and cookies.

5. **Audit CI/CD pipelines** for any runs that installed an affected package version. If found, treat the runner's environment as fully compromised and rotate all secrets injected into that runner.

6. **Remove persistence entries** if found (systemd units, crontab entries, launchd plists, Windows Registry Run keys referencing NodeJS).

7. **Block the C2 IP** at firewall/egress level: `85.137.53.71` on all ports. Also block `ipfs.io` at the egress layer if not required by your workflows.

8. **Investigate AI coding assistant output** from any sessions on compromised machines — the `ai-tool-poisoner.js` module may have injected malicious instructions.

9. **Audit branch-protection rules** on all repositories using OIDC trusted publishing. The push identity `"Your Name" <you@example.com>` / GitHub login `invalid-email-address` should trigger immediate investigation if found in git log.

10. **Lock IPFS gateway egress** from CI runners — legitimate builds do not fetch from `ipfs.io`.

---

## Lessons Learned

- **OIDC provenance attestations are not tamper-evidence** — they prove an authorized workflow ran, not that the triggering commit was legitimate. Provenance does not protect against a compromised push credential. Defend the branch, not just the publish token.
- **Load-time droppers bypass `--ignore-scripts`** — this dropper has no `preinstall`/`postinstall` hook. The only protection is not importing the package, or catching the anomalous behavior at runtime (network egress monitoring, process spawn alerts).
- **Six redundant C2 channels make blocking impractical** — HTTP, Nostr, IPFS, BitTorrent DHT, libp2p GossipSub, and Ethereum blockchain. Any single-vector blocking strategy fails. Focus on detecting the initial compromise and the drop path, not the C2 channel.
- **Miasma's metamorphic engine means signatures will miss future generations** — each propagated copy is re-obfuscated. IOC-based detection needs to focus on behavioral patterns (drop paths, C2 constants, HKDF key material strings) not static byte signatures.
- **The "canary" batch strategy (`5% → waves of 100`) means targeted orgs may not see mass reports** before being hit. Don't wait for widespread disclosure — monitor your own installs.
- **IPFS as a payload delivery channel is increasingly common** and is routinely allowed by firewall rules. Explicitly audit and restrict egress to `ipfs.io` from CI runners and developer machines.
- **AI tool poisoning is now a first-class capability** in the Miasma framework. A compromised developer environment should be treated as a compromised AI coding assistant environment too.

---

## Related Incidents

- [Miasma v2 — Self-Spreading npm Worm Uses binding.gyp Execution Bypass](./2026-06-miasma-v2-binding-gyp.md)
- [Miasma — Red Hat @redhat-cloud-services npm Packages Compromised](./2026-06-miasma-redhat-npm.md)
- [GitHub Actions Miasma Tag Hijacking — codfish/semantic-release-action](./2026-06-github-actions-miasma-tag-hijacking.md)
- [Miasma Azure/durabletask Repo Injection](./2026-06-miasma-azure-repo-injection.md)
- [Leo Platform npm Supply Chain Attack — Miasma Toolkit](./2026-06-leo-platform-npm-miasma.md)
- [Hades Campaign PyPI](./2026-06-hades-campaign-pypi.md)
- [Bitwarden CLI Shai-Hulud Third Coming](./2026-04-bitwarden-cli-shai-hulud.md)
