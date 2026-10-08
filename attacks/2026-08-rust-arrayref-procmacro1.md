# Rust Supply Chain Attack — arrayref, internment, and append-only-vec via proc-macro1 Build-Time Dropper

**Date:** August 2026
**Ecosystem:** Rust / crates.io
**Severity:** High
**Type:** Account Compromise / Build-Time Dropper / Infostealer / Credential Theft
**Sources:**
- [Rust Blog (official) — Supply chain attack on arrayref](https://blog.rust-lang.org/2026/08/20/supply-chain-attack-on-arrayref/)
- [SafeDep — Malicious Rust Crate arrayref Runs a Build-Time Payload](https://safedep.io/arrayref-proc-macro1-rust-build-time-malware/)
- [StepSecurity — arrayref, internment, and append-only-vec Poisoned by the proc-macro1 Build-Time Dropper](https://www.stepsecurity.io/blog/arrayref-rust-crate-supply-chain-attack)
- [Aikido — Popular Rust crates arrayref, append-only-vec, and internment compromised](https://www.aikido.dev/blog/two-popular-rust-crates-arrayref-and-append-only-vec-compromised-in-supply-chain-attack)
- [Semgrep — Rust crates arrayref & append-only-vec compromised via malicious proc-macro1 dependency](https://semgrep.dev/blog/2026/rust-crates-arrayref-append-only-vec-compromised-proc-macro1/)
- [Wiz — Rust Supply Chain Attack on arrayref: Significant Overlap with DPRK Campaigns](https://www.wiz.io/blog/rust-supply-chain-attack-on-arrayref-significant-overlap-with-dprk-campaigns)
- [BleepingComputer — Hackers poison arrayref Rust crate to push infostealer malware](https://www.bleepingcomputer.com/news/security/hackers-poison-arrayref-rust-crate-to-push-infostealer-malware/)

---

## Summary

On August 20, 2026, a threat actor compromised the crates.io account of `droundy`, the maintainer of three widely-used Rust crates: `arrayref` (245 million all-time downloads), `internment`, and `append-only-vec`. The attacker published malicious new versions of all three crates within a 23-minute window, adding a dependency on `proc-macro1` — a typosquatted lookalike of David Tolnay's legitimate `proc-macro2` crate. The typosquat impersonated Tolnay's identity in its Cargo metadata and forged his email address, making it convincingly deceptive.

The attack delivered its payload at **build time via Cargo's build script mechanism**. The malicious `build.rs` inside `proc-macro1` decoded a hidden server address from base64 fragments, downloaded an architecture-specific binary over TLS (with certificate validation deliberately disabled), and ran it silently in the background during compilation. The payload stole browser-saved credentials from Chrome, Brave, and Edge; installed persistent access via systemd (Linux), LaunchAgent (macOS), or registry Run key (Windows); and accepted remote commands including the ability to fetch and execute arbitrary shell or PowerShell scripts.

To maximise exposure, the attacker yanked the previous clean versions of `arrayref` (0.3.5 through 0.3.9), causing Cargo to display a "consider updating" warning that nudged developers toward the only non-yanked release — the malicious 0.3.10. Wiz subsequently linked the infrastructure to **UNC1069**, a DPRK (North Korea) state-sponsored threat actor, noting significant overlap with the Mastra npm worm (June 2026) and the axios npm RAT campaign (March 2026), suggesting this is part of a multi-ecosystem developer-targeting campaign.

---

## Compromised Artifacts

| Crate | Malicious Version | Publisher | Exposure Window | Downloads |
|-------|------------------|-----------|-----------------|-----------|
| `arrayref` | 0.3.10 | `droundy` (compromised) | 86 min (07:15–08:41 UTC) | ~245M all-time |
| `internment` | 0.8.7 | `droundy` (compromised) | 90 min (07:34–09:04 UTC) | — |
| `append-only-vec` | 0.1.9 | `droundy` (compromised) | 107 min (07:37–09:25 UTC) | — |
| `proc-macro1` | 1.0.107 (dropper) | `dtolney` (impersonation) | Removed entirely | — |
| `proc-macro-en`, `aovine`, `arone`, `aronenao`, `tinymember` | all versions | attacker | Removed entirely | — |

`proc-macro1` was published by a fake account `dtolney` that impersonates `dtolnay` (David Tolnay), complete with forged `authors = ["David Tolnay <dtolnay@gmail.com>"]` metadata.

---

## How It Worked

### Entry Point: Maintainer Account Compromise

The attacker gained access to `droundy`'s crates.io account. The `droundy` GitHub account and all associated repositories (`github.com/droundy/arrayref`, `github.com/droundy/append-only-vec`) returned 404 following the incident — access was likely achieved via credential theft, phishing, or reuse of previously stolen credentials consistent with UNC1069's known tooling.

### Stage 1: Yank-and-Nudge Delivery Mechanism

The attacker simultaneously yanked arrayref versions 0.3.5 through 0.3.9 while publishing 0.3.10. Yanking produces a Cargo warning: *"consider updating to a version that is not yanked."* With only 0.3.10 available as non-yanked, Cargo naturally resolves to the malicious version on any fresh install, `cargo update`, or CI cache miss. This converts Cargo's own safety guidance into the delivery channel.

### Stage 2: The proc-macro1 Typosquat Dependency

The malicious `arrayref@0.3.10` adds a single new line to its `Cargo.toml`:

```toml
[dependencies.proc-macro1]
version = "1.0.107"
```

`arrayref`'s `src/lib.rs` is unchanged — the legitimate macro code is untouched. Cargo builds every declared non-optional dependency whether or not the code imports it, so this manifest entry alone causes `proc-macro1` to be fetched and built. The `src/` of `proc-macro1` is a verbatim copy of `proc-macro2` with mechanical name substitution, so builds continue to succeed and the crate functions correctly.

`proc-macro1 1.0.107` adds three `[build-dependencies]` not present in real `proc-macro2`: `base64 ^0.22`, `rustls ^0.23`, and `ureq ^2` — forming a complete TLS-capable HTTP downloader toolkit.

### Stage 3: Build-Time Payload Download and Execution

The `proc-macro1/build.rs` splits the server address into base64 fragments to avoid static string detection:

```rust
const SRC_URL_PARTS: &[&str] =
    &["aHR0cHM6Ly8=", "MjMuMjU0Lg==", "MTY1Lg==", "MTEyOg==", "OTA4OS8="];
const END_URL_PARTS: &[&str] =
    &["MjMuMjU0Lg==", "MTY1Lg==", "MTEyOg==", "NDQz"];
```

Decoded: `SRC_URL_PARTS` = `https://23.254.165.112:9089/` (payload host); `END_URL_PARTS` = `23.254.165.112:443` (C2 address passed as argv[1] to the payload).

Certificate validation is disabled via a custom `AcceptAll` verifier that unconditionally returns success from every rustls `ServerCertVerifier` trait method, allowing a self-signed certificate on the raw IP to pass.

The script selects an architecture-specific binary:

| Platform | Payload name |
|----------|-------------|
| Linux / x86_64 | `rust-crate_0.1.0` |
| Windows / x86_64 | `rust-crate_0.2.0` |
| macOS / x86_64 | `rust-crate_0.3.0` |
| macOS / aarch64 | `rust-crate_0.4.0` |

On Unix, the binary is written to `/tmp/rust-setup`, chmod'd +x, and spawned detached (stdin/stdout/stderr all null). On Windows, a PowerShell script is written to `%TEMP%\rust-setup.ps1` and launched via a VBScript launcher `%TEMP%\rust-setup-launch.vbs` through `wscript.exe` to escape Cargo's job object so the payload continues running after the build finishes. The child handle is `std::mem::forget`-ted to prevent cleanup.

### Stage 4: Payload Capabilities

The second-stage binary:
- Beacons to `23.254.165.112:443` with host details and stolen credentials encoded as base64 JSON
- Reads browser-saved credentials from Chrome, Brave, and Edge SQLite login databases
- Installs persistence: systemd user service (Linux), LaunchAgent (macOS), registry `Run` key (Windows)
- Accepts remote commands, including fetching and executing arbitrary shell/PowerShell
- Falls back to a Domain Generation Algorithm (DGA) if the primary C2 goes silent

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| 2026-08-20T07:15:00Z | `arrayref@0.3.10` published; simultaneously, `arrayref@0.3.5`–`0.3.9` yanked |
| 2026-08-20T07:15:00Z | Rust Security Response Team receives report that `proc-macro1` is malicious (from Nextron Systems GmbH) |
| 2026-08-20T07:29:50Z | Socket AI Scanner independently detects `proc-macro1` |
| 2026-08-20T07:34:07Z | `internment@0.8.7` published |
| 2026-08-20T07:37:49Z | `append-only-vec@0.1.9` published |
| 2026-08-20T08:41:40Z | `arrayref@0.3.10` deleted from crates.io (86 min exposure) |
| 2026-08-20T09:04:11Z | `internment@0.8.7` deleted (90 min exposure) |
| 2026-08-20T09:25:24Z | `append-only-vec@0.1.9` deleted (107 min exposure) |
| 2026-08-20T~09:30Z | `proc-macro1` and all associated crates removed; yanked versions of arrayref unyanked; `droundy` account locked |
| 2026-08-20 | Rust Blog official advisory published (Manish Goregaokar on behalf of security-response team) |
| 2026-08-21 | Aikido, Semgrep, SafeDep technical analyses published |
| 2026-08-23 | StepSecurity analysis published, notes `internment` as third affected crate |
| 2026-08-~20 | Wiz publishes analysis linking attack to UNC1069 / DPRK infrastructure overlapping with Mastra and axios campaigns |

---

## IOCs

| Type | Indicator | Detail |
|------|-----------|--------|
| IP | `23.254.165.112` | Payload host (port 9089/HTTPS) and C2 (port 443) |
| URL | `https://23.254.165.112:9089/rust-crate_0.1.0` | Linux x86_64 payload |
| URL | `https://23.254.165.112:9089/rust-crate_0.2.0` | Windows x86_64 payload |
| URL | `https://23.254.165.112:9089/rust-crate_0.3.0` | macOS x86_64 payload |
| URL | `https://23.254.165.112:9089/rust-crate_0.4.0` | macOS aarch64 payload |
| File (Unix) | `/tmp/rust-setup` | Downloaded executable |
| File (Windows) | `%TEMP%\rust-setup.ps1` | Downloaded PowerShell script |
| File (Windows) | `%TEMP%\rust-setup-launch.vbs` | VBScript launcher |
| Crate | `proc-macro1` (any version) | Malicious typosquat of proc-macro2 |
| Crate | `proc-macro-en`, `aovine`, `arone`, `aronenao`, `tinymember` | Auxiliary attacker crates |
| SHA256 | `25ad700976873c76af785cb99b33c48db7df8b81f21d1e9e06b3676b9a9373ae` | arrayref 0.3.10 |
| SHA256 | `61198155da51b838772eecf5bfaac6cbc4dcc388dccc56658fc28a8e831b34d4` | proc-macro1 1.0.107 |
| SHA256 | `b5c1b5b0763a8809a644a8f92224653f0aca623a98eecc714d27f74b80fbe436` | proc-macro1 1.0.106 |
| Attribution | UNC1069 | DPRK-linked threat actor; shared infra with Mastra (Jun 2026) and axios (Mar 2026) campaigns |

---

## Detection

```bash
# Check local Cargo cache for malicious crate artifacts
find ~/.cargo/registry/cache -type f \( \
  -name 'append-only-vec-0.1.9.crate' -o \
  -name 'arrayref-0.3.10.crate' -o \
  -name 'internment-0.8.7.crate' -o \
  -name 'proc-macro1-*.crate' -o \
  -name 'proc-macro-en-*.crate' -o \
  -name 'aovine-*.crate' -o \
  -name 'arone-*.crate' -o \
  -name 'aronenao-*.crate' -o \
  -name 'tinymember-*.crate' \
\) -print

# Check for dropped payload binary (Unix)
ls -la /tmp/rust-setup 2>/dev/null

# Check for payload files (Windows PowerShell)
# Get-Item "$env:TEMP\rust-setup.ps1", "$env:TEMP\rust-setup-launch.vbs" 2>$null

# Check for C2 IP in network logs
grep -r "23\.254\.165\.112" ~/.cargo/registry/ 2>/dev/null

# Check for proc-macro1 in any project's Cargo.lock
find . -name 'Cargo.lock' -exec grep -l 'proc-macro1' {} \;

# Verify arrayref is on a safe version (should be 0.3.9 or earlier)
cargo tree | grep 'arrayref'

# Check for malicious systemd user service (Linux)
systemctl --user list-units | grep -v expected_services

# Check for malicious LaunchAgent (macOS)
ls ~/Library/LaunchAgents/ 2>/dev/null

# Check Cargo.lock for any of the three malicious versions
grep -E 'name = "(arrayref|internment|append-only-vec)"' Cargo.lock
grep -A1 'name = "arrayref"' Cargo.lock | grep 'version = "0.3.10"'
grep -A1 'name = "internment"' Cargo.lock | grep 'version = "0.8.7"'
grep -A1 'name = "append-only-vec"' Cargo.lock | grep 'version = "0.1.9"'
```

---

## Remediation

1. **Run the detection commands above** to check if any malicious crates are in your local registry cache or Cargo.lock files.
2. **Update arrayref to 0.3.9** (unyanked by the Rust Security Response Team): `cargo update arrayref --precise 0.3.9`
3. **Remove malicious crate files** from `~/.cargo/registry/cache/` if found.
4. **Delete dropped payload files** if present: `/tmp/rust-setup` (Unix), `%TEMP%\rust-setup.ps1` / `%TEMP%\rust-setup-launch.vbs` (Windows).
5. **Block / monitor C2 IP** `23.254.165.112` at the firewall level immediately.
6. **If any of the three malicious versions were compiled between 07:15–09:25 UTC on Aug 20, 2026**, assume credentials are compromised:
   - Rotate all browser-saved passwords (Chrome, Brave, Edge)
   - Rotate SSH keys, cloud IAM credentials, npm/PyPI tokens, API keys
   - Check for unexpected systemd user services, LaunchAgents, or registry Run entries
   - Review login history for unexpected access
7. **Run `cargo audit`** to confirm no other advisories affect your dependency graph.
8. **Pin crate versions** in `Cargo.toml` using `=x.y.z` exact versions rather than caret ranges for security-sensitive projects.

---

## Lessons Learned

- **Build scripts are a critical attack surface in Rust.** `build.rs` runs automatically for every declared dependency, even if the consuming crate's code never imports the dependency. A single added line in `Cargo.toml` is sufficient to trigger arbitrary code execution on every build.
- **Yank-and-nudge is a weaponizable delivery mechanism.** Yanking previous safe versions turns Cargo's own UX ("consider updating") into a distribution channel for the malicious release. Registries should rate-limit yanking and alert maintainers on abnormal yank patterns.
- **Typosquatting by impersonating a well-known maintainer's identity** (forged email, forged GitHub username `dtolney` vs `dtolnay`) is highly effective and difficult to detect via visual inspection of metadata alone.
- **Base64-fragmented C2 addresses evade static analysis.** The server address was never stored as a plaintext string — it was assembled at runtime from fragments. This technique is increasingly common in supply chain malware.
- **DPRK is now actively targeting crates.io**, not just npm and PyPI. The multi-ecosystem approach (Rust + npm + PyPI) by UNC1069 signals maturation in state-sponsored developer targeting.
- **The payload targets developers specifically**: browser credentials are secondary; the primary value is CI/CD tokens, cloud IAM credentials, and private source code. Build-time execution gives attackers access before any runtime security controls.

---

## Related Incidents

- [./2026-06-mastra-npm-easy-day-js.md](./2026-06-mastra-npm-easy-day-js.md) — UNC1069 infrastructure overlap; Mastra npm campaign (Jun 2026)
- [./2026-03-axios-npm-rat.md](./2026-03-axios-npm-rat.md) — UNC1069 infrastructure overlap; axios npm RAT campaign (Mar 2026)
- [./2026-06-onering-rust-code-exfil.md](./2026-06-onering-rust-code-exfil.md) — Earlier Rust/crates.io build.rs attack (Jun 2026)
- [./2026-05-trapdoor-npm-pypi-crates.md](./2026-05-trapdoor-npm-pypi-crates.md) — Earlier crates.io supply chain campaign (May 2026)
