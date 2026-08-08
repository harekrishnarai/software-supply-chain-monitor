# jscrambler npm Supply Chain Attack — Malicious Preinstall Binary Drops Cross-Platform Rust Stealer with eBPF Rootkit

**Date:** July 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Maintainer Account Compromise / Preinstall Binary Dropper / Native Credential Stealer
**Sources:**
- [StepSecurity — jscrambler npm package publishes malicious preinstall binary](https://www.stepsecurity.io/blog/jscrambler-npm-package-publishes-malicious-preinstall-binary)

---

## Summary

On July 11, 2026, five versions of `jscrambler` — the official npm CLI for Jscrambler's commercial JavaScript obfuscation and Code Integrity API — were published to npm with a malicious payload. The package has around ten active maintainers and a version history dating back to 0.1.0, giving it an established presence in CI/CD build pipelines that made it a high-value target. The first malicious release, 8.14.0, appeared at 15:12 UTC; four further compromised versions followed in a two-hour window as the attacker iteratively modified the delivery mechanism. The tarball grew from 37.8 kB (8.13.0, the last clean release on June 30) to 7.9 MB.

The payload is not JavaScript. A 7.8 MB file named `dist/intro.js` — indistinguishable by name from legitimate build artifacts — is in fact a custom binary container holding three gzip-compressed native executables (Linux ELF x86-64, Windows PE32+, macOS Mach-O arm64). The loader extracts the platform-matched binary to a randomly named dotfile in the OS temp directory and spawns it fully detached, so it survives after `npm install` exits. Static analysis revealed capabilities far beyond a typical npm stealer: an embedded SQLite engine targeting Chrome/Firefox credential stores, an embedded LevelDB engine targeting Chromium extension wallet storage (MetaMask and similar), an embedded BIP39 English wordlist for seed-phrase validation, and on Linux, an in-memory eBPF loader that installs embedded BPF bytecode directly into the kernel without ever writing the program to disk.

StepSecurity's OSS AI Package Analyst flagged 8.14.0 with a suspicion score of 0 (maximum) on publish. Clean version 8.22.0 is the current latest; compromised and clean releases are interleaved in the 8.14.0–8.22.0 range, so pinning to a version number rather than `latest` or a caret range is essential.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Published (UTC) | Trigger Mechanism |
|---------|---------------------|-----------------|-------------------|
| `jscrambler` | 8.14.0 | 2026-07-11 15:12 | preinstall hook → dist/setup.js |
| `jscrambler` | 8.16.0 | 2026-07-11 17:26 | preinstall hook → dist/setup.js |
| `jscrambler` | 8.17.0 | 2026-07-11 17:41 | preinstall hook → dist/setup.js |
| `jscrambler` | 8.18.0 | 2026-07-11 17:46 | inlined in dist/index.js (fires on require()) |
| `jscrambler` | 8.20.0 | 2026-07-11 17:53 | inlined in dist/index.js (fires on require()) |

Last known clean release: `8.13.0` (2026-06-30). Confirmed clean current release: `8.22.0`.

---

## How It Worked

### Entry Point — Binary Container Disguised as JavaScript

`dist/intro.js` carries a custom 5-byte magic header (`1B 43 53 49 01`) that no legitimate JS file would produce. The format that follows is: a platform-id byte, an 8-byte size field, an 8-byte little-endian compressed size, then gzip-compressed data — repeated once per supported platform (Linux, Windows, macOS). The file is placed alongside legitimately named build artifacts such as `mutations.js` and `queries.js`, minimising the surface that an automated scanner or code reviewer has to flag as anomalous. The loader script (`dist/setup.js`) is itself short and unobfuscated, because all sensitive logic lives inside the compiled binary and never appears as text.

### Generation 1 (8.14.0–8.17.0): Preinstall Hook

`package.json` gains a new `scripts.preinstall` entry pointing to `dist/setup.js`. On `npm install`, npm runs this automatically. The loader reads `dist/intro.js`, verifies the magic header, selects the gzip block matching `process.platform`, decompresses it to a randomly named dotfile in the OS temp directory (with a `.exe` suffix on Windows), sets the executable bit, and spawns the binary fully detached and `unref()`'d — orphaning the child process from the npm install tree so it keeps running after install completes and leaves no obvious parent-process relationship in tools like `ps`.

### Generation 2 (8.18.0–8.20.0): Inline Execution on require()

The attacker removed the `scripts.preinstall` entry entirely — making these two releases appear clean to any scanner that checks only for lifecycle hooks — and instead inlined the identical loader logic as an immediately-invoked function at the very top of `dist/index.js`, the package's main entry point. This means the payload fires the moment anything calls `require("jscrambler")`: a build tool, a CI helper script, or any application using the Jscrambler API programmatically. No explicit `npm install` step is required to trigger it. The `dist/intro.js` container binary is byte-for-byte identical across all five compromised releases.

### Payload Mechanics — Cross-Platform Credential and Wallet Stealer

Static analysis of the extracted binaries surfaced a consistent capability signature across all three platforms:

- **Embedded SQLite engine** with FTS3/FTS4 tokenizer strings — the storage format Chrome and Firefox use for `Login Data`, `Cookies`, and `Web Data` credential stores.
- **Embedded LevelDB engine** — the format Chromium uses for Local Storage and IndexedDB, where browser-extension wallets like MetaMask persist their encrypted vault.
- **Embedded BIP39 English wordlist** — used to parse and validate crypto wallet seed phrases extracted from any of the above stores.

### Advanced Capabilities: Kernel Instrumentation and Anti-Analysis

**Linux binary:** dynamically links against `libbpf.so.1` and imports `bpf_object__open_mem`, `bpf_object__load`, `bpf_program__attach`, `bpf_map__fd`, and related libbpf functions. `bpf_object__open_mem` loads a compiled eBPF program from an in-memory buffer, meaning the BPF bytecode is embedded inside the binary itself and never touches the filesystem before being loaded into the kernel. This provides kernel-level instrumentation, not just userspace file access. The binary also imports the generic `syscall()` function rather than named wrappers for sensitive operations such as `ptrace` or `memfd_create`, keeping those calls out of the dynamic symbol table and defeating simple import-based detection. Full recovery of what the embedded eBPF program does requires extracting and disassembling the BPF bytecode (part of ongoing Stage 2 analysis at time of writing).

**Windows binary:** imports `IsDebuggerPresent` (standard anti-debugging check) and `GetExtendedTcpTable` from `iphlpapi.dll` (enumerates active TCP connections with owning process IDs — commonly used by stealers to detect security tooling by network footprint, or to identify and close a running browser before copying its credential store).

**macOS binary:** imports `sysctl` and `sysctlbyname`, the standard mechanism for checking the `P_TRACED` flag to detect an attached debugger.

### C2 Infrastructure

Harden-Runner network monitoring captured the dropped binary making outbound connections to `archive.torproject.org` and `check.torproject.org`, along with direct IP connections to `37.27.122.124` and `57.128.246.79`. Full C2 endpoint recovery from binary disassembly is ongoing.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| 2026-06-30 | `jscrambler@8.13.0` published — last known clean release (37.8 kB tarball) |
| 2026-07-11 15:12 | `jscrambler@8.14.0` published — tarball grows to 7.9 MB; preinstall hook added |
| 2026-07-11 15:12 | StepSecurity AI Package Analyst flags 8.14.0 with suspicion score 0 (maximum), triggering investigation |
| 2026-07-11 17:26 | `jscrambler@8.16.0` published — same preinstall mechanism, identical container binary |
| 2026-07-11 17:41 | `jscrambler@8.17.0` published — same preinstall mechanism |
| 2026-07-11 17:46 | `jscrambler@8.18.0` published — preinstall hook removed; payload inlined into dist/index.js |
| 2026-07-11 17:53 | `jscrambler@8.20.0` published — same inline mechanism |
| 2026-07-12 | StepSecurity publishes full technical disclosure; IOCs added to global block list |
| 2026-07-12 | `jscrambler@8.22.0` confirmed clean (current latest at time of disclosure) |

---

## Detection

```bash
# Check if any compromised jscrambler version is installed
npm ls jscrambler 2>/dev/null | grep -E '8\.(14|16|17|18|20)\.0'

# Check package-lock.json for any compromised version
grep -E '"jscrambler": "8\.(14|16|17|18|20)\.0"' package-lock.json

# Check for the oversized dist/intro.js container (7.8 MB in a clean CLI would be ~37 kB total)
find node_modules/jscrambler/dist -name 'intro.js' -size +1M

# Verify the magic bytes of dist/intro.js (first 5 bytes: 1B 43 53 49 01 indicates malicious container)
xxd node_modules/jscrambler/dist/intro.js 2>/dev/null | head -1

# Hunt for the dropped binary in temp directories (randomly named dotfile)
find /tmp $TMPDIR ${TEMP} -maxdepth 2 -name '.[a-z0-9]*' -perm /111 2>/dev/null

# Windows: check %TEMP% for randomly named dotfile executables
# dir /a %TEMP%\.[a-z0-9]*.exe

# Check for orphaned processes spawned around install/require time
# Linux: look for processes with no terminal and unusual parent
ps aux | grep -v grep | awk '$7 == "?" && $8 !~ /^(Ss|S|R)/' 

# SHA256 verification of extracted payload blobs
# Linux ELF: fbbcf4d8f98168f78f5c0c47a9ae56d59ec8ac84a7c9ca6b797fedfb8d62d2bd
# Windows PE32+: b7ca95d1b23c8e67416a25cedf741de0917c2096bbc9d24649eea7853d054903
# macOS Mach-O arm64: c8fd47d36bdf7c825378593ab82ed8c24d1dc52e26b507812393e24e1d5201fd

# Check for network connections to known C2 IPs
ss -tnp | grep -E '37\.27\.122\.124|57\.128\.246\.79'
netstat -tnp 2>/dev/null | grep -E '37\.27\.122\.124|57\.128\.246\.79'

# Check for DNS lookups to Tor C2 domains in system resolver cache
# Linux (systemd-resolved)
resolvectl query check.torproject.org 2>/dev/null
resolvectl query archive.torproject.org 2>/dev/null
```

---

## Remediation

1. **Pin to a clean version immediately:** `npm install jscrambler@8.22.0`. Do NOT use a caret/tilde range — compromised and clean releases are interleaved between 8.14.0 and 8.22.0, and a range spec could resolve to a malicious version.
2. **Treat any host where a compromised version was installed or required as fully compromised.** The dropped binary had write access to browser credential stores (Chrome/Firefox `Login Data`, `Cookies`, `Web Data`) and Chromium Local Storage/IndexedDB used by extension wallets (MetaMask, etc.).
3. **Rotate all credentials accessible from the affected machine:** cloud provider consoles, npm publish tokens, GitHub tokens, SSO sessions, API keys, and any credentials stored in the browser.
4. **Move crypto assets** from any wallet whose extension storage was accessible to a new wallet with a freshly generated seed phrase, generated on a clean device.
5. **Hunt for the dropped binary** in OS temp directories (`/tmp`, `%TEMP%`, `$TMPDIR`) matching the pattern `.[a-z0-9]{6,}` (`.exe` on Windows), and kill any still-running orphaned processes.
6. **CI/CD pipelines:** if jscrambler was installed in any build or release workflow, assume every secret injected into that job (npm tokens, cloud credentials, signing keys) was reachable by the dropped process. Rotate all injected secrets for affected pipelines.
7. **Check package-lock.json and `node_modules/`** across all repos and developer machines before the next build.

---

## Lessons Learned

- **Compromised and clean versions interleaved in the same major.minor range** defeats naïve "stay on latest" strategies; organizations must pin exact versions and use a registry proxy with a cooldown policy.
- **Hiding compiled binaries with a `.js` extension** bypasses scanners that rely on file extension to decide whether to inspect content — magic-byte checks are necessary.
- **Removing the preinstall hook in later versions** (Generation 2) would have defeated any scanner checking only `package.json` scripts; payload analysis must cover the full file tree and module entry points.
- **In-memory eBPF loading** (`bpf_object__open_mem`) produces no on-disk artifact for the BPF bytecode, complicating forensic recovery and defeating file-based detection of rootkit components.
- **Using `syscall()` directly** rather than named libc wrappers for sensitive operations removes those calls from the dynamic symbol table, defeating import-based behavioral heuristics.
- **Established, commercial packages with large maintainer teams are high-value targets** precisely because they appear in CI/CD pipelines at build time, giving the payload access to all secrets available to the build runner.
- **AI-assisted publish analysis** (StepSecurity's OSS AI Package Analyst) flagged this on publish — demonstrating that automated scoring of tarball diffs can close the detection gap between compromise and human review.

---

## Related Incidents

- [./2026-07-injective-npm-supply-chain.md](./2026-07-injective-npm-supply-chain.md) — Same week, npm maintainer compromise targeting crypto wallet keys
- [./2026-06-ironworm-npm-weavedb.md](./2026-06-ironworm-npm-weavedb.md) — Rust-compiled ELF dropper with eBPF rootkit, similar kernel instrumentation technique
- [./2026-06-miasma-v2-binding-gyp.md](./2026-06-miasma-v2-binding-gyp.md) — Native binary delivery via npm lifecycle hooks
- [./2026-03-axios-npm-rat.md](./2026-03-axios-npm-rat.md) — High-DL npm package maintainer account hijacked, cross-platform RAT deployed
