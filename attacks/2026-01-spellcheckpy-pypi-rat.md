# spellcheckpy / spellcheckerpy — PyPI Typosquats Deliver Python RAT via Hidden Dictionary Payload

**Date:** January 2026
**Ecosystem:** PyPI
**Severity:** High
**Type:** Typosquatting / Infostealer / RAT
**Sources:**
- [Aikido Security — Malicious PyPI Packages spellcheckpy and spellcheckerpy Deliver Python RAT](https://www.aikido.dev/blog/malicious-pypi-packages-spellcheckpy-and-spellcheckerpy-deliver-python-rat)
- [The Hacker News — Fake Python Spellchecker Packages on PyPI Delivered Hidden Remote Access Trojan](https://thehackernews.com/2026/01/fake-python-spellchecker-packages-on.html)

---

## Summary

On January 20–21, 2026, Aikido's malware detection pipeline flagged two newly published PyPI packages: `spellcheckerpy` and `spellcheckpy`. Both claimed to be authored by the maintainer of the legitimate `pyspellchecker` library — a false claim designed to lend credibility. The packages collectively received approximately 1,000 downloads before being removed from PyPI.

The attack employed a novel payload concealment technique: the malicious code was hidden inside a **Basque language dictionary file** (`resources/eu.json.gz`), which exists in the legitimate `pyspellchecker` package as a word-frequency corpus. A base64-encoded payload was embedded within the gzipped JSON, making it invisible to simple string-based scanners that don't inspect compressed resource files. The payload downloaded a full-featured Python RAT from an external domain — `updatenet[.]work` — hosted by RouterHosting LLC (also known as Cloudzy), a provider with a documented history of hosting nation-state threat actors.

The attack showed a deliberate staged rollout: early versions (1.0.x–1.1.x) fetched and decoded the payload without executing it, likely for testing infrastructure reachability. Only `spellcheckpy@1.2.0`, published January 21, 2026, gained full execution capability.

---

## Compromised Artifacts

| Package | Ecosystem | Malicious Versions | Notes |
|---|---|---|---|
| `spellcheckerpy` | PyPI | All published versions | Typosquat of `pyspellchecker` |
| `spellcheckpy` | PyPI | All published versions including 1.2.0 | Typosquat; v1.2.0 first to execute RAT |

Both packages were removed from PyPI following disclosure.

---

## How It Worked

### Entry Point — Typosquatting with Stolen Maintainer Identity

Both packages impersonated the author of the widely-used `pyspellchecker` library. The package descriptions, metadata, and resource file structure mirrored the legitimate package closely, including the presence of dictionary files for multiple languages.

### Payload Concealment — Hidden in a Dictionary File

The malicious payload was embedded inside `resources/eu.json.gz` — the Basque language word-frequency dictionary. In the legitimate `pyspellchecker`, this file contains genuine Basque word data. In the malicious packages, the gzip file contained a base64-encoded Python infostealer payload hidden alongside plausible-looking dictionary content.

This concealment technique was chosen specifically to evade:
- Simple `grep`-based scanners (the payload is inside a compressed binary file)
- Source-code-only audits (the dictionary appears to be a normal data file)
- Hash-based detection of the main Python files (the malicious logic was in resources, not `.py` files)

### Staged Execution

- **Versions 1.0.x–1.1.x**: Fetched and decoded the payload from the dictionary file but never executed it — infrastructure testing phase.
- **Version 1.2.0** (January 21, 2026): Added execution capability. Calling `SpellChecker()` (the standard class instantiation) triggered the malicious routine. The RAT executed upon first import + class instantiation, with no additional function calls required.

### Payload Mechanics

The payload functioned as a two-stage loader:

**Stage 1 — Downloader:**
Extracted and decoded from `eu.json.gz`. Connected to `updatenet[.]work` to retrieve the full Python RAT binary.

**Stage 2 — Python RAT:**
A full-featured remote access trojan. Capabilities observed included:
- Remote command execution
- File system access and exfiltration
- Credential harvesting (browser data, saved tokens)
- Persistence mechanisms

### C2 Infrastructure

| Domain | Purpose | Hosting |
|---|---|---|
| `updatenet[.]work` | Stage 2 RAT download and C2 | RouterHosting LLC (Cloudzy) |

RouterHosting LLC / Cloudzy has a documented history of providing hosting services to nation-state-affiliated threat groups, though no definitive attribution to a specific actor was made for this campaign.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2026-01-20 | `spellcheckerpy` published to PyPI; Aikido detection fires |
| 2026-01-20 | `spellcheckpy` initial versions published — fetch-only, no execution |
| 2026-01-21 | `spellcheckpy@1.2.0` published — first version with RAT execution capability |
| 2026-01-23 | Aikido publishes full disclosure; packages removed from PyPI |
| 2026-01-23 | The Hacker News and SC Media cover the incident |

---

## Detection

```bash
# Check for installed malicious packages
pip show spellcheckpy 2>/dev/null
pip show spellcheckerpy 2>/dev/null
pip list | grep -E "spellcheck(py|erpy)"

# Check requirements files and lockfiles
grep -r "spellcheckpy\|spellcheckerpy" requirements*.txt pyproject.toml Pipfile* 2>/dev/null

# Inspect the eu.json.gz file if a version was installed
# (Locate the installed package directory first)
python3 -c "import importlib, spellcheckpy; print(spellcheckpy.__file__)" 2>/dev/null
# Then inspect: ls <package_dir>/resources/ — look for eu.json.gz with unexpected size

# Check for C2 domain in network logs
grep -i "updatenet" /var/log/syslog /var/log/dns*.log 2>/dev/null
# macOS:
log show --last 7d | grep -i "updatenet.work" 2>/dev/null
# Windows:
ipconfig /displaydns | findstr "updatenet"

# Check for Python processes making outbound connections
lsof -i -n -P 2>/dev/null | grep -E "python|python3" | grep ESTABLISHED
# Windows (PowerShell):
Get-NetTCPConnection -State Established | Where-Object {$_.OwningProcess -in (Get-Process python*).Id}

# Verify legitimate pyspellchecker if installed (safe package)
pip show pyspellchecker | grep -E "Name|Version"
# The legitimate package is 'pyspellchecker', not 'spellcheckpy'/'spellcheckerpy'
```

---

## Remediation

1. **Remove the malicious packages**:
   ```bash
   pip uninstall spellcheckpy spellcheckerpy -y
   ```

2. **Check package lockfiles** (`requirements.txt`, `Pipfile.lock`, `pyproject.toml`) and remove any references to `spellcheckpy` or `spellcheckerpy`. Replace with the legitimate `pyspellchecker` if spell-checking functionality is needed.

3. **Assume compromise if installed**: If either package was imported and the `SpellChecker` class was instantiated (especially v1.2.0 of `spellcheckpy`), a full Python RAT was executed. Treat the host as compromised:
   - Rotate all credentials, API keys, and SSH keys accessible from the machine
   - Review for persistence mechanisms (cron jobs, startup scripts, systemd units, LaunchAgents)
   - Check for unexpected outbound connections to `updatenet[.]work`

4. **Block the C2 domain** at DNS/firewall: `updatenet[.]work` and `*.updatenet.work`.

5. **Audit CI/CD pipelines**: If the package was installed during an automated build process, treat all secrets accessible to that build environment as compromised and rotate them.

---

## Lessons Learned

- **Compressed resource files are blind spots for many scanners**: Embedding a payload inside a `.gz` archive within a data directory bypasses most static analysis tools that focus on Python source files.
- **Maintainer identity impersonation is easy and effective**: PyPI does not currently verify that new packages are truly authored by the claimed maintainer. Typosquatting with a stolen identity adds credibility that name-only typosquats lack.
- **Staged deployment for infrastructure testing**: Publishing non-executing initial versions demonstrates deliberate testing tradecraft — attackers confirmed their download/decode pipeline worked before enabling execution.
- **Execution on instantiation, not import**: Triggering the RAT via `SpellChecker()` rather than at import time means that only code that actually uses the library fires the payload — reducing suspicious import-time side effects that some scanners detect.
- **Nation-state hosting infrastructure signals**: C2 domains hosted by providers with documented links to state-sponsored threat actors (Cloudzy/RouterHosting LLC) warrant elevated concern about the sophistication and persistence of the threat.

---

## Related Incidents

- [./2026-01-gwagon-npm-infostealer.md](./2026-01-gwagon-npm-infostealer.md) — Concurrent npm infostealer campaign (same week, January 2026)
- [./2026-01-dydx-npm-pypi.md](./2026-01-dydx-npm-pypi.md) — January 2026 cross-ecosystem credential stealer
- [./2026-03-telnyx-pypi-wav.md](./2026-03-telnyx-pypi-wav.md) — WAV steganography credential stealer injected into legitimate PyPI package
- [./2026-03-litellm-pypi-stealer.md](./2026-03-litellm-pypi-stealer.md) — Credential stealer hidden in a .pth file within a popular PyPI wheel
