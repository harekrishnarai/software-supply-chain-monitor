# telnyx PyPI — WAV Steganography Credential Stealer (TeamPCP)

**Date:** March 2026
**Ecosystem:** PyPI
**Severity:** Critical
**Type:** Malicious package / Infostealer / WAV steganography / Windows persistence dropper
**Sources:**
- [StepSecurity — TeamPCP Plants WAV Steganography Credential Stealer in telnyx PyPI Package](https://www.stepsecurity.io/blog/teampcp-plants-wav-steganography-credential-stealer-in-telnyx-pypi-package)
- [Aikido — Popular telnyx package compromised on PyPI by TeamPCP](https://www.aikido.dev/blog/telnyx-pypi-compromised-teampcp-canisterworm)

---

## Summary

On March 27, 2026, TeamPCP injected a WAV steganography-based credential stealer into two releases of the official Telnyx Python SDK on PyPI (`telnyx==4.87.1` and `4.87.2`). The compromise was disclosed via GitHub issue [team-telnyx/telnyx-python#235](https://github.com/team-telnyx/telnyx-python/issues/235). Telnyx is a cloud communications platform SDK used in telephony applications, call automation pipelines, and communications infrastructure — with 742,000 downloads in the 30 days preceding the attack.

The attack is the latest escalation in the weeks-long TeamPCP supply chain campaign that previously targeted Trivy (Mar 19), npm via CanisterWorm (Mar 20), Checkmarx GitHub Actions (Mar 23), and LiteLLM (Mar 24). Attribution is high-confidence: the injected payload contains a byte-for-byte identical RSA-4096 public key to the litellm compromise, uses the same `tpcp.tar.gz` exfiltration archive name with the `X-Filename: tpcp.tar.gz` HTTP header, and employs the identical AES-256-CBC + RSA-OAEP hybrid encryption scheme observed across all TeamPCP campaigns.

The attack introduces a **novel payload delivery mechanism not used in prior TeamPCP operations**: the credential-stealing script and the Windows PE binary are not embedded directly in the package but are instead hidden inside `.wav` audio files served from C2 infrastructure. The WAV files are structurally valid audio containers; the malicious content is encoded in the audio frame data using a base64 + XOR scheme. This technique evades content-based network filtering tools that would block raw binary transfers but pass audio file downloads. A version-to-version capitalization bug (`Setup()` vs `setup()`) caused the Windows attack to fail silently in `4.87.1`; it was corrected in `4.87.2`, which is fully functional on all platforms.

---

## Compromised Artifacts

| Artifact | Malicious Version(s) | Platform Impact |
|----------|----------------------|-----------------|
| `telnyx` (PyPI) | `4.87.1` | Linux/macOS attack functional; Windows broken (casing bug) |
| `telnyx` (PyPI) | `4.87.2` | Linux/macOS and Windows both fully functional |

Last known clean version: `telnyx==4.87.0` (published to GitHub March 26, 2026).

---

## How It Worked

### Entry Point: `telnyx/_client.py` Modified at Import Time

Unlike litellm (which used a `.pth` file or proxy server injection), the telnyx attack modifies only one file: `src/telnyx/_client.py` — the top-level client module imported by every application that uses the SDK. The malicious code runs the moment any application executes `import telnyx`. There is no postinstall hook to block; the vector is the import itself.

74 lines of malicious code are injected across four locations in the file:

1. **Lines 4–10 (imports):** Seven new standard-library imports added after the module docstring — `subprocess`, `tempfile`, `time`, `os`, `base64`, `sys`, `wave` — plus a duplicate `os` import as minor obfuscation. A casual reviewer scanning the import block for additions may miss them.
2. **Lines 41–42 (helper):** A `_d()` base64 decoder function used to deobfuscate Windows attack strings at runtime.
3. **Line 459 (payload variable):** A 4,436-character base64 blob `_p` inserted just before `__all__`. This decodes to the full Linux/macOS credential harvester script.
4. **Lines 7761–7825 (functions + trigger):** `setup()` (Windows dropper), `FetchAudio()` (Linux/macOS harvester), and module-level calls to both — executing immediately on import.

---

### WAV Steganography Delivery

Both attack paths (Linux/macOS and Windows) use the same steganographic technique to deliver their second-stage payload from `http://83.142.209.203:8080/`:

1. Fetch a `.wav` file (either `ringtone.wav` or `hangup.wav`) using a `Mozilla/5.0` User-Agent to blend with browser traffic.
2. Open the file as a WAV audio container using Python's `wave` module.
3. Read all audio frames and base64-decode them.
4. Treat the first 8 bytes as an XOR key; XOR-decode the remainder.
5. The result is either a Python script (Linux/macOS) or a native PE binary (Windows).

The WAV files are structurally valid audio and pass MIME-type checks. Content-based filtering that does not parse WAV audio frame data will not detect the hidden payload. TeamPCP first used this technique five days earlier in their Kubernetes wiper variant (March 22, 2026); the telnyx attack represents the first use of WAV steganography to deliver a credential stealer.

---

### Linux / macOS: Detached Credential Harvester

`FetchAudio()` runs on non-Windows systems. It immediately spawns a detached subprocess (`start_new_session=True`), decoupling the harvester from the parent application — the calling code returns immediately with no visible errors or delays.

The detached process decodes `_p` (the base64 blob at line 459) and executes it. This decoded script implements the `audioimport()` function, which:

1. Fetches `ringtone.wav` from `http://83.142.209.203:8080/`
2. Decodes the WAV frame data to obtain a third-stage Python credential harvester script
3. Pipes the script directly to a new Python interpreter via stdin — **no script file is ever written to disk**
4. Collects: SSH keys, AWS credentials + IMDS tokens, Kubernetes service account tokens, GCP/Azure credentials, `.env` files, Docker registry tokens, shell history, and all environment variables
5. Encrypts the collected data with a random 32-byte AES-256-CBC session key
6. Wraps the session key with TeamPCP's RSA-4096 public key (OAEP padding) — only the attacker can decrypt
7. Packs both into `tpcp.tar.gz` and POSTs to `http://83.142.209.203:8080/` with `X-Filename: tpcp.tar.gz`
8. The temp directory is automatically cleaned up; no artifacts remain on disk after exfiltration

---

### Windows: Persistent PE Binary via Startup Folder

`setup()` runs on Windows (functional only in `4.87.2`; broken in `4.87.1` due to `Setup()` NameError). All sensitive string literals are base64-obfuscated using the `_d()` helper:

| Obfuscated value | Decoded |
|------------------|---------|
| `QVBQREFUQQ==` | `APPDATA` |
| `TWljcm9zb2Z0XFdpbmRvd3NcU3RhcnQg...` | `Microsoft\Windows\Start Menu\Programs\Startup` |
| `bXNidWlsZC5leGU=` | `msbuild.exe` |
| `aHR0cDovLzgzLjE0Mi4yMDkuMjAzOjgwODAvaGFuZ3VwLndhdg==` | `http://83.142.209.203:8080/hangup.wav` |

The attack flow:

1. Checks for `%APPDATA%\...\Startup\msbuild.exe.lock`; if it exists and is less than 12 hours old, exits (anti-duplication throttle).
2. Fetches `hangup.wav` from `http://83.142.209.203:8080/hangup.wav` and decodes the WAV frames to a native Windows PE binary.
3. Writes the binary to `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\msbuild.exe`.
4. Executes silently with `creationflags=0x08000000` (CREATE_NO_WINDOW).
5. Creates `msbuild.exe.lock` and hides it with `attrib +h` to prevent casual discovery.

The Startup folder is a built-in Windows persistence mechanism — any executable there runs automatically on every user login. Using `msbuild.exe` as the filename impersonates the legitimate Microsoft Build Engine, a trusted system binary, reducing the chance of analyst scrutiny when the Startup folder is reviewed.

---

### 4.87.1 vs. 4.87.2: Attacker Iteration

The sole code difference between the two compromised versions is the casing of the Windows trigger:

```
4.87.1:  Setup()   ← Python NameError (name 'Setup' is not defined) — silently swallowed
4.87.2:  setup()   ← correct, Windows attack fully functional
```

Python identifiers are case-sensitive. In `4.87.1`, `setup()` is defined but `Setup()` (capitalised) is called — raising a `NameError` that is silently caught. The Windows attack fails with no visible error. `FetchAudio()` uses correct casing in both versions and was functional from the first release.

The fix published in `4.87.2` within the same day indicates the attacker tested the Windows path, identified the bug, and corrected it — consistent with the rapid operational tempo seen in earlier TeamPCP campaigns (litellm `1.82.7`/`1.82.8` also iterated within hours).

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 19, 2026 | Trivy compromised; CVE-2026-33634 (CVSS 9.4); CI/CD credential mass harvest begins |
| Mar 20, 2026 | TeamPCP deploys CanisterWorm across 46+ npm packages using credentials stolen from Trivy users |
| Mar 22, 2026 | TeamPCP first observed using WAV steganography in Kubernetes wiper payload |
| Mar 23, 2026 | Checkmarx KICS and AST GitHub Actions compromised; `checkmarx.zone` C2 domain active |
| Mar 24, 2026 | LiteLLM `1.82.7`/`1.82.8` published to PyPI with TeamPCP payload; same RSA key |
| Mar 27, 2026 ~03:51 UTC | `telnyx==4.87.1` published to PyPI with malicious `_client.py` |
| Mar 27, 2026 | `telnyx==4.87.2` published; corrects Windows casing bug; both platforms now fully exploited |
| Mar 27, 2026 | Disclosed via GitHub issue team-telnyx/telnyx-python#235 |
| Mar 27, 2026 | StepSecurity and Aikido publish independent analyses |

---

## IOCs

| Type | Indicator |
|------|-----------|
| Malicious package | `telnyx==4.87.1` |
| Malicious package | `telnyx==4.87.2` |
| C2 / payload delivery | `http://83.142.209.203:8080/` |
| WAV payload (Linux/macOS) | `http://83.142.209.203:8080/ringtone.wav` |
| WAV payload (Windows) | `http://83.142.209.203:8080/hangup.wav` |
| Exfiltration endpoint | `http://83.142.209.203:8080/` (POST) |
| Exfil archive name | `tpcp.tar.gz` |
| Exfil HTTP header | `X-Filename: tpcp.tar.gz` |
| Windows persistence | `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\msbuild.exe` |
| Windows lock file | `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\msbuild.exe.lock` |
| RSA key prefix | `MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAvahaZDo8...` (identical to litellm key) |

**SHA-256 hashes (StepSecurity — `.tar.gz` source distributions):**

| File | SHA-256 |
|------|---------|
| `telnyx-4.87.1.tar.gz` | `f66c1ea3b25ec95d0c6a07be92c761551e543a7b256f9c78a2ff781c77df7093` |
| `telnyx-4.87.2.tar.gz` | `a9235c0eb74a8e92e5a0150e055ee9dcdc6252a07785b6677a9ca831157833a5` |

**SHA-256 hashes (Aikido — wheel distributions):**

| File | SHA-256 |
|------|---------|
| `telnyx==4.87.1` (wheel) | `7321caa303fe96ded0492c747d2f353c4f7d17185656fe292ab0a59e2bd0b8d9` |
| `telnyx==4.87.2` (wheel) | `cd08115806662469bbedec4b03f8427b97c8a4b3bc1442dc18b72b4e19395fe3` |

---

## Detection

```bash
# Check installed telnyx version
pip show telnyx | grep Version

# Hash check the installed _client.py against a known-clean copy
# (look for unexpected imports: wave, subprocess near the top, and a large _p = "..." blob)
python -c "import importlib.util, sys; spec=importlib.util.find_spec('telnyx'); print(spec.submodule_search_locations if spec else 'not installed')"

# Find and inspect the installed _client.py for injection markers
find $(python -c "import site; print(site.getsitepackages()[0])" 2>/dev/null) \
  -path "*/telnyx/_client.py" -exec grep -l "FetchAudio\|ringtone\.wav\|hangup\.wav\|tpcp\.tar\.gz" {} \;

# Check for outbound connections to the C2 IP
grep -r "83\.142\.209\.203" /var/log/ 2>/dev/null

# Linux/macOS: check for temp artifacts from harvester (may already be cleaned up)
find /tmp -name "*.enc" -o -name "tpcp.tar.gz" 2>/dev/null

# Windows: check for malicious startup binary
# Run in PowerShell:
# Test-Path "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\msbuild.exe"
# Test-Path "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\msbuild.exe.lock"

# Check whether telnyx 4.87.0 (clean) is installed
pip show telnyx | grep -E "^Version: 4\.87\.0$" && echo "CLEAN" || echo "CHECK VERSION"

# Check for WAV steganography pattern in any Python scripts (XOR decode of WAV frames)
grep -r "wave\.open\|readframes.*getnframes\|xor\|ringtone\.wav\|hangup\.wav" \
  $(python -c "import site; print(site.getsitepackages()[0])" 2>/dev/null) 2>/dev/null
```

---

## Remediation

1. **Check your installed version immediately:**
   ```bash
   pip show telnyx
   ```
   If `Version` is `4.87.1` or `4.87.2`, the system is compromised.

2. **Downgrade to the last clean release:**
   ```bash
   pip install "telnyx==4.87.0"
   ```

3. **Windows only — remove the persistence binary:**
   - Check: `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\msbuild.exe`
   - If present, delete both `msbuild.exe` and `msbuild.exe.lock`
   - The binary has already executed; treat the machine as fully compromised.

4. **Rotate all credentials accessible on the affected system:**
   - SSH private keys (and remove corresponding `authorized_keys` entries on remote hosts)
   - AWS access keys, session tokens, IAM role credentials; IMDS-accessible credentials
   - Kubernetes service account tokens and kubeconfig certificates
   - GCP service account keys and `application_default_credentials.json`
   - Azure credentials and `.azure/` directory contents
   - All `.env` file values (local, staging, production)
   - Docker and container registry tokens
   - npm, PyPI, and other package registry tokens
   - Any API keys present as environment variables

5. **Audit network logs** for outbound HTTP connections to `83.142.209.203` on port 8080. Any connection indicates payload delivery or credential exfiltration occurred.

6. **If used in CI/CD:** Any pipeline that ran `pip install telnyx` without version pinning between the malicious publish time and removal is compromised. Rotate all secrets in that environment.

7. **Verify clean install** by inspecting `telnyx/_client.py` for absence of `FetchAudio`, `setup`, `ringtone.wav`, `_p =`, or the `wave` import.

---

## Lessons Learned

- WAV steganography is an effective evasion technique against network-level content inspection — defenders should enforce egress allowlisting by domain/IP in CI/CD and production, not by content type, since valid-looking audio files can carry arbitrary payloads.
- TeamPCP's rapid version iteration (4.87.1 → 4.87.2 within the same day to fix a bug) demonstrates active operational monitoring; defenders cannot rely on a single malicious release window being short.
- The attack runs at `import telnyx` time with no postinstall hook — package install hooks can be blocked by tools like `--no-scripts`, but import-time injection requires inspecting the package source before installation.
- An unbroken chain of credential theft across five compromises in eight days (Trivy → CanisterWorm → Checkmarx → LiteLLM → telnyx) demonstrates how a single CI/CD token theft can cascade: each new set of stolen credentials enables the next attack.
- The Windows LOLBin impersonation (`msbuild.exe` in the Startup folder) exploits analyst familiarity with trusted binary names; Startup folder monitoring should alert on any new binary regardless of filename.
- The `X-Filename: tpcp.tar.gz` HTTP exfiltration header is a persistent TeamPCP campaign signature that IDS/IPS rules can be written against to detect active exfiltration in real time.
- Pip's default behaviour of installing the latest version on `pip install --upgrade` means that any environment running unattended upgrades without version pinning is silently vulnerable to this class of attack.

---

## Related Incidents

- [litellm PyPI — Credential Stealer Hidden in Wheel](./2026-03-litellm-pypi-stealer.md) — same TeamPCP actor, same RSA key, same encryption scheme; three days prior
- [Checkmarx KICS GitHub Action Compromised](./2026-03-checkmarx-kics-action.md) — same campaign; `checkmarx.zone` C2 domain
- [Trivy Second Compromise](./2026-03-trivy-second-compromise.md) — upstream credential theft that enabled the LiteLLM and telnyx attacks
- [CanisterWorm — TeamPCP npm Worm & Kubernetes Wiper](./2026-03-canisterworm-npm.md) — WAV steganography first seen in this campaign's K8s wiper
- [Trivy GitHub Actions Tag Compromise](./2026-03-trivy-github-actions.md) — root of the TeamPCP credential chain
