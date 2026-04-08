# ForceMemo: GitHub Python Repo Compromise via Blockchain C2

**Date:** March 2026
**Ecosystem:** GitHub / Python
**Severity:** High
**Type:** Account takeover / Force-push injection / Blockchain C2 / Infostealer
**Sources:**
- [StepSecurity — ForceMemo: Hundreds of GitHub Python Repos Compromised via Account Takeover and Force-Push](https://www.stepsecurity.io/blog/forcememo-hundreds-of-github-python-repos-compromised-via-account-takeover-and-force-push)

---

## Summary

Starting March 8, 2026, an attacker compromised hundreds of GitHub developer accounts and injected identical obfuscated malware into hundreds of Python repositories using a stealth technique StepSecurity named **ForceMemo**. The attacker takes the latest legitimate commit from each compromised repo, rebases it with malware appended to a key Python file (`setup.py`, `main.py`, `app.py`, etc.), and force-pushes — preserving the original commit message, author name, and author date. The only forensic trace is a discrepancy between the author date and committer date, sometimes spanning years.

The malware uses the **Solana blockchain as a C2 channel**: it queries a specific Solana wallet address for transaction memos containing base64-encoded payload URLs, making C2 instructions immutable and censorship-resistant. The Solana C2 address (`BjVeAjPrSKFiingBn4vZvghsGj9KCE8AJVtbc9S8o8SC`) is identical to the one used by the concurrent GlassWorm campaign, confirming both attacks are operated by the same threat actor.

The malware includes CIS country exclusion (skips execution on Russian locale/timezone) and Russian-language code comments — consistent with Eastern European cybercrime operations.

---

## Compromised Artifacts

Over 240+ GitHub Python repositories injected, including:

| Repository | Stars | Files Targeted |
|-----------|-------|---------------|
| `amirasaran/django-restful-admin` | 70 | `setup.py` |
| `BierOne/relation-vqa` | — | `setup.py` |
| `BierOne/bottom-up-attention-vqa` | — | `setup.py` |
| `biodatlab/siriraj-assist` | — | `main.py` |
| `KeithSloan/ImportNURBS` | — | `setup.py` |
| `KeithSloan/GDML` | — | — |

GitHub code search for marker variable: `lzcdrtfxyqiplpd`

---

## How It Worked

### Phase 1: Account Takeover via GlassWorm Credential Theft

Compromised developers were previously infected by GlassWorm malware through compromised VS Code and Cursor extensions. GlassWorm's stage-3 payload harvests GitHub tokens from:
- `git credential fill`
- VS Code extension storage
- `~/.git-credentials`
- `GITHUB_TOKEN` environment variable

With these tokens, the attacker gained access to all repositories under each victim account — evidenced by multiple repos per account being simultaneously compromised (e.g., `BierOne`: 6 repos, `wecode-bootcamp-korea`: 6 repos).

### Phase 2: Stealth Injection via Force-Push

The attacker:
1. Takes the latest legitimate commit on the default branch
2. Rebases it, appending obfuscated malware to a key Python file
3. Force-pushes with the original commit message and author preserved
4. Sets committer email to the literal string `"null"` — a fingerprint of the attacker's tooling

The committer date in git log is the only giveaway, sometimes showing a 9-year discrepancy (author date: 2017, committer date: 2026).

### Phase 3: Solana Blockchain C2

The obfuscated payload (base64 + zlib + XOR key 134) decodes to malware that:
1. Checks if the system is Russian (locale, timezone, UTC offset) — **aborts if so**
2. Queries Solana address `BjVeAjPrSKFiingBn4vZvghsGj9KCE8AJVtbc9S8o8SC` via 9 RPC fallback endpoints
3. Decodes the latest memo transaction to get the current payload URL
4. Downloads and executes the second-stage payload

The Solana address has 50+ transactions going back to November 27, 2025, showing the attacker has been rotating payload servers for months before pivoting to GitHub repo injection.

### Phase 4: Payload Execution

1. Downloads Node.js v22.9.0 from `nodejs.org` to the user's home directory
2. Fetches the AES-encrypted JavaScript payload (decryption key delivered in HTTP response headers)
3. Writes `~/i.js` and executes via the downloaded Node binary
4. Creates `~/init.json` with a 2-day recheck timer to avoid repeated execution

The final payload is consistent with crypto wallet stealer / infostealer campaigns (CIS exclusion, browser extension targeting, Node.js runtime, AES encryption).

---

## Timeline

| Date | Event |
|------|-------|
| 2025-11-27 | Earliest Solana C2 address activity; attacker begins testing with payload server `45.32.151.157` |
| Dec 2025 – Feb 2026 | Attacker rotates through payload servers; ~40 memo transactions |
| 2026-02-25 | Direct C2 configuration posted on-chain revealing `c2server: http://217.69.11.99:5000` |
| 2026-03-08 | Earliest GitHub repo injections: `metalogico/issued`, `uknfire/tsmpy`, `gnlxpy/*`, `wecode-bootcamp-korea/*` |
| 2026-03-10 | Major wave: `amirasaran/*`, `KeithSloan/ImportNURBS`, `watercrawl/*` |
| 2026-03-12 | `BierOne/ood_coverage` (34-star ICLR paper) compromised |
| 2026-03-13 | Latest wave: `BierOne/*`, `biodatlab/siriraj-assist`, `HydroRoll-Team/*` |
| 2026-03-14 | First repos begin reverting; StepSecurity publishes findings and files security issues |

---

## Detection

```bash
# Search for the malware marker variable in Python files
grep -r "lzcdrtfxyqiplpd" . --include="*.py"

# Check for the persistence file
ls ~/init.json 2>/dev/null && echo "INFECTED: persistence file found"

# Check for downloaded Node.js runtime
ls ~/node-v22* 2>/dev/null && echo "SUSPICIOUS: Node.js downloaded to home directory"

# Check for the payload JS file
find ~ -name "i.js" -newer ~/Downloads 2>/dev/null

# Detect author/committer date discrepancies in git log
git log --format="%H %ae %ai %ci %s" | awk -F' ' '
  {
    author_date = $3
    commit_date = $4
    if (author_date != commit_date) {
      print "DATE MISMATCH in commit:", $1, "\n  Author:", author_date, "\n  Committer:", commit_date
    }
  }'

# Check for outbound Solana RPC connections in network logs
grep -E "mainnet-beta\.solana\.com|getblock\.us|blockeden\.xyz" \
  /var/log/syslog /var/log/auth.log 2>/dev/null

# Check for connections to known C2 IPs
for ip in 45.32.151.157 45.32.150.97 217.69.11.57 217.69.11.99 217.69.0.159 45.76.44.240; do
  netstat -an | grep "$ip" && echo "ALERT: connection to ForceMemo C2 IP $ip"
done
```

---

## Remediation

1. **Search all recently-cloned Python repos** for the marker variable `lzcdrtfxyqiplpd`.
2. **Delete persistence artifacts**: `~/init.json`, `~/i.js`, `~/node-v22*`.
3. **Rotate GitHub tokens** for any account that has installed a VS Code or Cursor extension since October 2025.
4. **Verify git history** of Python repos you install from GitHub: compare committer date vs author date; inspect large discrepancies.
5. **Block Solana RPC endpoints** in CI/CD environments (they should never be contacted by Python build tooling).
6. **Block known C2 IPs** at firewall: `45.32.151.157`, `45.32.150.97`, `217.69.11.57`, `217.69.11.99`, `217.69.0.159`, `45.76.44.240`.
7. **Never `pip install` directly from a GitHub URL** without first inspecting the diff against the last known-good commit.

---

## Lessons Learned

- The Solana blockchain is a viable, censorship-resistant C2 channel — defenders cannot take down the instructions, only block access to the RPC endpoints.
- Force-push injection that preserves the original author/message is nearly invisible to casual review; automated committer-date anomaly detection is needed.
- Account-level compromise (all repos under one account compromised simultaneously) is the clearest early signal of token theft feeding a mass-injection campaign.
- Shared C2 infrastructure (same Solana wallet as GlassWorm) allows cross-campaign attribution and indicator reuse.
- The CIS exclusion pattern and Russian code comments are operational security indicators pointing to Eastern European cybercrime, consistent with many crypto-focused infostealers.

---

## Related Incidents

- [./2026-03-glassworm-returns.md](./2026-03-glassworm-returns.md) — GlassWorm March 2026 wave; shares identical Solana C2 address
- [./2026-03-glassworm-canisterworm.md](./2026-03-glassworm-canisterworm.md) — Earlier GlassWorm wave; same actor
