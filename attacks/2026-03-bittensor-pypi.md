# bittensor-wallet 4.0.2 PyPI Backdoor

**Date:** March 2026
**Ecosystem:** PyPI
**Severity:** Critical
**Type:** Backdoor / Infostealer / Crypto wallet key theft
**Sources:**
- [StepSecurity — bittensor-wallet 4.0.2 Compromised on PyPI - Backdoor Exfiltrates Private Keys](https://www.stepsecurity.io/blog/bittensor-wallet-4-0-2-compromised-on-pypi---backdoor-exfiltrates-private-keys)

---

## Summary

On March 15, 2026, a backdoored release of `bittensor-wallet` (version 4.0.2) was uploaded to PyPI. The package remained live for approximately 48 hours before being yanked on March 17. `bittensor-wallet` is the official Rust-backed Python library for managing Bittensor cryptographic keys — coldkeys, hotkeys, signing, and staking — making it a direct path to private key material on the Bittensor (TAO) decentralized AI marketplace.

The backdoor was compiled into the native Rust extension rather than appended as Python code, making it significantly harder to detect than typical supply chain injections. It is hooked into every wallet decryption operation and uses three independent exfiltration channels (HTTPS, DGA-generated domains, and DNS tunneling) with anti-analysis countermeasures including uptime checks, debugger detection, and XOR-obfuscated strings. The attacker also stripped artifact attestation from the release to prevent cryptographic verification of the wheel provenance.

---

## Compromised Artifacts

| Package | Malicious Version | SHA256 (source tarball) |
|---------|------------------|------------------------|
| `bittensor-wallet` (PyPI) | 4.0.2 | `6a416b72ff24804abc12484a3b41413a8580acedd8a5f8c84224fcf0732c2f8e` |

Safe version: `4.0.1`
Exposure window: ~48 hours (2026-03-15 ~05:06 UTC to 2026-03-17 ~12:06 UTC)

---

## How It Worked

### Entry Point

The backdoor is compiled into the native Rust extension (`src/keyfile.rs`) and hooked at three call sites via `collect_format_metrics(&decrypted_data)` — triggering on every wallet decryption operation. The attacker also stripped build provenance attestation from `release.yml`, removing the sigstore signature and making it impossible to verify wheel provenance.

### Anti-Analysis Checks

Before exfiltrating, the malware checks whether it is being analyzed:
- **Uptime check**: aborts if system uptime < 20 minutes (sandbox detection)
- **Debugger check**: reads `TracerPid` from `/proc/self/status`; aborts if non-zero
- **Tool detection**: scans running processes for `strace`, `gdb`, `wireshark`, `frida`, `bpftrace`; aborts if found

All detection strings are XOR-obfuscated (key `0x42`) in the binary.

### Wallet Type Detection

The malware inspects the Python call stack via `pyo3` to determine whether the decrypted key is a coldkey (main vault, controls staked funds) or hotkey (day-to-day key), tagging the payload as `"c"` or `"h"`.

### Payload Encryption

The stolen private key material is encrypted with a hardcoded attacker NaCl RSA-4096 public key using hybrid encryption (AES-256-CBC + RSA-OAEP), ensuring only the attacker can decrypt intercepted traffic.

Attacker NaCl pubkey (hex): `daeb c8f3 3fd7 9a8e d65b d438 3280 cab1 3f00 f2a0 3ff5 13ca 7c50 aa85 7ecd d46f`

### Three-Layer Exfiltration

**Method A — HTTPS via Python urllib (pyo3)**
Posts encrypted payload to hardcoded C2 domains. Uses `daemon=False` threading to ensure delivery even from short-lived CLI commands.

**Method B — Raw TLS via dlopen/dlsym**
If Method A fails, opens `libssl.so` directly using unsafe FFI — bypassing Python-layer network monitoring and `urllib` monkey-patching.

**Method C — DNS Tunneling**
If both HTTPS methods fail, the payload is hex-encoded, split into 60-character chunks, and sent as DNS A queries:
```
<hex_chunk>.<index>.<total>.<session_id>.t.opentensor-cdn.com
```
No successful DNS response is needed — the attacker's nameserver reconstructs the key from the query log.

### Three-Layer C2 Resolution

1. **Hardcoded domains** (XOR-encoded, key `0x5A`):
   - `finney.opentensor-metrics.com`
   - `finney.metagraph-stats.com`
   - `finney.subtensor-telemetry.com`

2. **Daily rotating DGA domains** — generates 3 new hostnames under `*.opentensor-cdn.com` by hashing the current day number. Domains rotate daily, evading static blocklists.

3. **DNS TXT C2** — queries `_dmarc.opentensor-cdn.com` (disguised as a DMARC lookup) to fetch dynamically updated backup server lists.

### Deduplication and Retry

A SHA256 hash of each stolen payload is kept in a `HashSet` to avoid exfiltrating the same key twice. Failed sends queue up to 64 entries and are retried every 2–10 minutes (randomized jitter) by a background thread named `cache-gc` to blend in with GC thread names. After exfiltration, key buffers are zeroed using volatile writes to erase forensic traces.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| 2026-03-15 ~05:06 | Malicious `bittensor-wallet==4.0.2` uploaded to PyPI |
| 2026-03-15 – 2026-03-17 | ~48-hour exposure window; any coldkey/hotkey decrypted during this period is compromised |
| 2026-03-17 ~12:06 | Package yanked from PyPI after Socket automated scanning flagged it |
| 2026-03-17 | StepSecurity publishes full technical analysis including Harden-Runner live capture |

---

## Detection

```bash
# Check if you have the malicious version installed
pip show bittensor-wallet | grep -E "^Version:"
# Malicious: Version: 4.0.2

# Verify SHA256 of installed source tarball
pip download bittensor-wallet==4.0.2 --no-deps -d /tmp/btcheck
sha256sum /tmp/btcheck/bittensor_wallet-4.0.2.tar.gz
# Malicious hash: 6a416b72ff24804abc12484a3b41413a8580acedd8a5f8c84224fcf0732c2f8e

# Check for the background cache-gc thread in running wallet processes
ps aux | grep python | xargs -I{} sh -c 'cat /proc/$(echo {} | awk "{print \$2}")/task/*/comm 2>/dev/null | grep cache-gc'

# Check network logs for C2 domains
grep -E "opentensor-metrics\.com|metagraph-stats\.com|subtensor-telemetry\.com|opentensor-cdn\.com" \
  /var/log/syslog /var/log/auth.log 2>/dev/null

# Check for DNS tunneling queries (28 chunks per session)
# Look for sequences of subdomains matching: <hex>.<index>.<total>.<session>.t.opentensor-cdn.com
dig _dmarc.opentensor-cdn.com TXT  # Should return NXDOMAIN or error if C2 is down

# Verify wheel provenance (would have caught this — attestation was stripped)
gh attestation verify bittensor_wallet-4.0.2-*.whl --repo opentensor/btwallet
```

---

## Remediation

1. **Immediate** — if `bittensor-wallet==4.0.2` was installed between March 15–17, 2026, assume all coldkeys and hotkeys whose keyfiles were decrypted during that window are compromised. **Generate new keys and move funds immediately.**
2. **Downgrade**: `pip install bittensor-wallet==4.0.1`
3. **Block C2 domains at DNS/firewall**:
   - `*.opentensor-metrics.com`
   - `*.metagraph-stats.com`
   - `*.subtensor-telemetry.com`
   - `*.opentensor-cdn.com` (includes DGA subdomains and DNS tunnel target)
4. **Audit network logs** for DNS queries or HTTPS requests to above domains.
5. **Check for `cache-gc` thread** in any running Python/Rust wallet process.
6. **Pin with hash verification** going forward:
   ```
   bittensor-wallet==4.0.1 \
     --hash=sha256:edc2588d5e272835285e4171dd3daf862149f617015bf52e43d433d8e5c297c5
   ```

---

## Lessons Learned

- Compiling malware into a native Rust extension makes it invisible to Python-focused static analysis tools and `grep`-based code review.
- Stripping build attestation from the release workflow removes the only cryptographic proof linking a wheel to a legitimate CI pipeline — defenders should treat unsigned releases as untrusted.
- Triple-redundant exfiltration (HTTPS + raw TLS + DNS tunneling) means blocking HTTP alone is insufficient; DNS-layer monitoring is essential for high-value packages.
- DGA-based C2 with daily rotation defeats static domain blocklists; defenders need behavioral detection (anomalous DNS patterns) rather than just indicator matching.
- Packages managing private key material (crypto wallets, secret managers, SSH libraries) deserve heightened scrutiny on every release.

---

## Related Incidents

- [./2025-04-xrp-supply-chain.md](./2025-04-xrp-supply-chain.md) — Similar credential exfiltration via compromised official SDK
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — npm infostealer campaign targeting crypto/browser credentials
