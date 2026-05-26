# Laravel-Lang Supply Chain Attack — Every Tag Across 4 Composer Packages Rewritten to Steal CI Secrets

**Date:** May 2026
**Ecosystem:** Packagist / Composer (PHP)
**Severity:** Critical
**Type:** Tag Poisoning / Credential Stealer / Autoloader Hijack
**Sources:**
- [StepSecurity — Laravel-Lang Supply Chain Attack: Every Tag Across Multiple Composer Packages Rewritten to Steal CI Secrets](https://www.stepsecurity.io/blog/laravel-lang-supply-chain-attack)
- [Aikido — Supply Chain Attack Targets Laravel-Lang Packages with Credential Stealer](https://www.aikido.dev/blog/supply-chain-attack-targets-laravel-lang-packages-with-credential-stealer)

---

## Summary

On May 22, 2026, a threat actor with org-wide push access to the **Laravel-Lang** GitHub organisation rewrote every existing git tag across four popular Composer packages — including the flagship `laravel-lang/lang` with 502 tags — within a 90-minute window (22:32–00:00 UTC). Rather than publishing a new suspicious version, the attacker pointed every historical tag (including versions years old) at new malicious commits, so any project running `composer update` or a fresh install would silently receive the backdoored code under a version it had pinned or trusted.

The attack exploits a fundamental property of Packagist: the registry does not host tarballs but instead indexes VCS repository URLs. A malicious git tag causes a Packagist webhook to fire within minutes, and the poisoned version is immediately served globally. The malicious commits add `src/helpers.php` to Composer's `autoload.files` map — the **eager autoloader**. Unlike PSR-4 or classmap, files listed in `autoload.files` are `require`d the instant `vendor/autoload.php` is called. Because every Laravel application calls this on startup, and because Symfony, PHPUnit, and most other PHP frameworks do the same, the payload fires on first boot with no class instantiation required.

The two-stage payload is sophisticated: a 30-line PHP dropper fetches a ~5,900-line second-stage credential stealer from C2 domain `flipboxstudio.info` (a typosquat of the legitimate `flipboxstudio.com`). The stealer is organised into 15 specialist collector modules and harvests cloud credentials, browser passwords, SSH keys, developer tokens, cryptocurrency wallets, Windows Credential Manager entries, VPN configs, and more — then AES-256-encrypts the results and POSTs them to the C2 before deleting itself from disk. StepSecurity confirmed end-to-end exploitation by detonating `laravel-lang/http-statuses v3.4.5` in an isolated runner. Packagist responded immediately by taking down the malicious versions and temporarily unlisting the affected packages.

---

## Compromised Artifacts

| Package | Tags Affected | Notes |
|---------|--------------|-------|
| `laravel-lang/lang` | All 502 tags | Flagship Laravel translations package; rewrites span 22:32–23:24 UTC |
| `laravel-lang/http-statuses` | All tags v1.0.0–v3.4.5 | Detonated and confirmed by StepSecurity |
| `laravel-lang/actions` | All 46 tags (1.0.0–1.12.2) | Rewrites complete by ~00:00 UTC |
| `laravel-lang/attributes` | All 86 tags | Same pattern; same fake author identity |

---

## How It Worked

### Entry Point

The attacker used a single compromised credential with org-wide push access to the Laravel-Lang GitHub organisation to force-push every tag in all four repositories to new malicious commits. All malicious commits share the same fake author identity (`Your Name`, `you@example.com`) and each modifies only two files: `composer.json` and `src/helpers.php`. The use of `Your Name` / `you@example.com` — a common placeholder left in place when no real identity is required — is a tell for automated or hastily scripted tag rewrites.

GitHub's interface flags these commits with the banner "This commit does not belong to any branch on this repository", because each commit is reachable only via a tag ref, not from any branch. This is the primary in-repo indicator of the attack.

The Aikido report notes that tags were pointed at commits from a fork of the same repository, exploiting GitHub's allowance for version tags to reference commits from forks.

### Autoloader Hijack (Stage 1 Trigger)

The malicious `composer.json` adds `src/helpers.php` to the `autoload.files` array. This is the eager autoloader: every file listed here is `require`d synchronously the first time `vendor/autoload.php` is included. Since `require __DIR__.'/vendor/autoload.php'` is the first line of virtually every Laravel application, Symfony app, PHPUnit test suite, and PHP framework project, **the payload fires on first boot, first test run, or first CI step that loads the project** — with no special trigger required.

### Stage 1 — PHP Dropper (`src/helpers.php`)

The dropper performs one-time-execution control (writing a marker file to `/tmp/.laravel_locale/<md5_hash>` to prevent re-firing), hides the C2 domain in an integer array decoded at runtime to evade static scanners, then fetches the second stage from `https://flipboxstudio.info/payload`. On Linux/macOS it executes the payload in a detached background process via `exec()`. On Windows it drops a `.vbs` launcher and runs the payload silently via `cscript`.

Harden-Runner captured the full process tree during detonation of `laravel-lang/http-statuses v3.4.5` (run 26318135547):

```
00:17:45 php (pid 2804) — autoload step
  └─ sh (pid 2805)
     └─ php (pid 2806, ppid=1) [reparented to init] — loader
        └─ sh (pid 2813)
           └─ nohup -> /tmp/.480dc608 (pid 2814, ppid=1) [reparented to init] — ELF stealer
              ├─ rm (pid 2816) — deletes PHP loader
              └─ rm (pid 2817) — deletes ELF binary
```

The entire compromise — from autoload to disk self-deletion of both artifacts — takes **3.16 seconds**. PIDs 2806 and 2814 are reparented to `init`, making them survive the workflow step boundary and appear unattributed to the runner user.

Two HTTPS calls to the C2 were captured:
- `GET https://flipboxstudio.info/payload` — fetches the stage 2 stealer
- `POST https://flipboxstudio.info/exfil` — exfiltrates the harvested data

### Stage 2 — PHP Credential Stealer (~5,900 lines, 15 collector modules)

The second stage is a comprehensive multi-platform credential stealer. After collection it AES-256-encrypts the results, POSTs them to `flipboxstudio.info/exfil`, and then deletes itself from disk.

**What it collects:**

- **Cloud credentials**: AWS keys + session tokens (env, `~/.aws/credentials`, EC2 IMDS); GCP application default credentials and token databases; Azure access tokens and MSAL cache; DigitalOcean, Heroku, Vercel, Netlify, Railway, Fly.io auth tokens
- **Infrastructure**: All kubeconfig files incl. `/etc/kubernetes/admin.conf`; HashiCorp Vault tokens; Helm repo configs; Docker `config.json`
- **Developer credentials**: SSH private keys; `.git-credentials`, `.gitconfig`; `.npmrc`, `.yarnrc`, `.pypirc`, `.gem/credentials`, `.composer/auth.json`; GitHub CLI, GitLab CLI, Hub CLI auth tokens; shell history files (bash, zsh, psql, mysql, python, node); all `.env` files; `wp-config.php`, `settings.py`, `docker-compose.yml`, `secrets.yaml`
- **Browsers and password managers**: Saved passwords from 17 Chromium-based browsers (Chrome, Edge, Brave, Opera, Vivaldi, Yandex, etc.) — on Windows a bundled helper `.exe` decrypts Chrome's DPAPI-protected login database; Firefox and Thunderbird `logins.json` and `key4.db`; KeePass `.kdbx`/`.kdb` files; 1Password and Bitwarden local vault files
- **Cryptocurrency wallets**: Bitcoin, Ethereum, Monero, Litecoin, Dash, Dogecoin, Zcash wallet files; Electrum, Exodus, Atomic, Ledger Live, Trezor, Wasabi, Sparrow; browser extension wallets by extension ID (MetaMask, Phantom, Trust Wallet, Ronin, Keplr, Solflare, Rabby)
- **Windows-specific**: Windows Credential Manager and Vault entries; PuTTY and WinSCP saved sessions (WinSCP passwords actively decrypted); `.rdp` files; Outlook registry profiles, OST/PST inventory, Credential Manager for Microsoft services
- **Communication platforms**: Slack tokens; Discord bot tokens; Telegram bot tokens
- **VPN configs**: NordVPN, ExpressVPN, ProtonVPN, CyberGhost, Private Internet Access, Windscribe, Mullvad, Surfshark, WireGuard, OpenVPN

---

## Timeline

| Date / Time (UTC) | Event |
|-------------------|-------|
| 2026-05-22 22:32 | First malicious tag rewrites begin on `laravel-lang/lang` (502 tags) |
| 2026-05-22 22:32–23:24 | `laravel-lang/lang` all 502 tags rewritten |
| 2026-05-22 ~23:xx | `laravel-lang/http-statuses` and `laravel-lang/attributes` tags rewritten |
| 2026-05-23 00:00 | `laravel-lang/actions` rewrites complete; attacker window closes |
| 2026-05-23 | StepSecurity files security issues in all four repositories (issues #8295, #277, #1193, #1085) |
| 2026-05-23 | Aikido detects the attack and reports to Packagist |
| 2026-05-23 | Packagist takes down malicious versions and temporarily unlists affected packages |
| 2026-05-23 | StepSecurity and Aikido publish public advisories |

---

## Detection

```bash
# Check if any of the four affected packages are installed
composer show laravel-lang/lang laravel-lang/http-statuses laravel-lang/actions laravel-lang/attributes 2>/dev/null

# Check composer.lock for malicious commit SHAs
# Malicious commits for laravel-lang/http-statuses include:
# bba2e443dc7ff1f8704f52a5375383e3f4f643b8 (v3.4.5)
# 26c233e1a0d4fd2331e8e0f175e18f8eed904aa3 (v3.4.0)
# db0c3ef246103fd0f6c318e0d48f26b5289044c3 (v3.0.0)
# See full list in the StepSecurity advisory
grep -A2 "laravel-lang" composer.lock | grep '"source"'

# Check if composer.lock was regenerated on or after 2026-05-22T22:32 UTC
stat composer.lock

# Check for the injection in installed vendor files
grep -r "flipboxstudio\|autoload.files.*helpers" vendor/laravel-lang/ 2>/dev/null
cat vendor/laravel-lang/http-statuses/src/helpers.php 2>/dev/null

# Check for orphaned processes left by the stealer
ps auxf | grep -E "ppid=1|php.*tmp"
# Look for php or unnamed ELF processes with ppid=1

# Check for marker/artifact files in /tmp (may be gone within seconds of execution)
ls -la /tmp/.laravel_locale/ 2>/dev/null
find /tmp -name "*.php" -path "*laravel*" 2>/dev/null
find /tmp -maxdepth 1 -name ".[0-9a-f]*" 2>/dev/null

# Check network logs for C2 traffic
# DNS / HTTPS requests to: flipboxstudio.info
# Endpoints: flipboxstudio.info/payload  (GET — dropper fetch)
#            flipboxstudio.info/exfil    (POST — credential exfiltration)

# Check git tag timestamps on the affected repos (all should be pre-2026-05-22)
# If any historical tag shows "created X hours ago" around 2026-05-22, it was rewritten
git ls-remote --tags https://github.com/laravel-lang/http-statuses
```

---

## Remediation

1. **Stop running `composer update` or `composer install` without a verified lockfile** for any project depending on `laravel-lang/lang`, `laravel-lang/http-statuses`, `laravel-lang/actions`, or `laravel-lang/attributes` until maintainers confirm all tags have been restored to their original commits
2. **Inspect `composer.lock`**: if the locked commit SHA matches any imposter commit listed in the StepSecurity advisory, or if the lockfile was regenerated on or after 2026-05-22 22:32 UTC, treat the environment as compromised
3. **Rotate all secrets** accessible to any environment that ran `composer install` of these packages since 22:32 UTC on 2026-05-22. The stealer targets: CI/CD tokens, cloud provider credentials (AWS/GCP/Azure), GitHub PATs, SSH keys, Docker registry creds, database credentials, npm/PyPI tokens, browser-stored passwords, and any `.env` secrets in the working directory
4. **Audit runner and developer machines** for orphaned `php` or unnamed ELF processes with `ppid=1` (`ps auxf`); check `/tmp/.laravel_locale/` for marker or payload files (these self-delete within seconds, so check quickly or use a live runner snapshot)
5. **Check DNS/TLS logs** for connections to `flipboxstudio.info`; this is the most durable forensic artifact since dropped files self-delete within 3 seconds of execution
6. **Add `flipboxstudio.info` to egress blocklists**, firewall rules, or DNS sinkholes
7. **If you are a Laravel-Lang maintainer**: force-update every tag back to its original commit (original tarballs are available on the Packagist dist mirror); revoke all PATs and OAuth app tokens with push access to the org; re-enforce 2FA on all org accounts; audit remaining repositories for similar rewrites

---

## Lessons Learned

- Tag poisoning is more dangerous than a new malicious version: it silently compromises projects that pinned to specific version ranges or even specific tags — including versions years old — with no new advisory to catch
- Composer's `autoload.files` (eager autoloader) is a zero-friction execution primitive: code added to `files` runs on every `require vendor/autoload.php` call, meaning no trigger, no class reference, no user interaction is required
- Self-deleting payloads that complete in under 4 seconds produce almost no forensic artifacts on disk; egress monitoring (DNS/HTTPS traffic logging) is the only reliable post-execution evidence source
- The 15-module credential stealer's breadth (cloud, browsers, wallets, VPNs, Windows Credential Manager, Slack/Discord/Telegram) demonstrates that attackers now treat PHP developer machines as full-spectrum credential vaults, not just CI/CD targets
- Packagist's architecture (indexing VCS URLs rather than hosting tarballs) means there is no pre-publish quarantine gate — once a GitHub tag is force-pushed, the poisoned version is globally available within minutes
- A single compromised org-level credential enabled 730+ tag rewrites across 4 repositories in 90 minutes; minimal credential privilege and audit logging on git push access are critical controls for open-source maintainer organisations

---

## Related Incidents

- [./2026-04-intercom-php-packagist.md](./2026-04-intercom-php-packagist.md) — Previous supply chain attack on a Packagist package via compromised maintainer credentials
- [./2026-05-actions-cool-issues-helper.md](./2026-05-actions-cool-issues-helper.md) — Tag-moving attack on GitHub Actions repo; same "rewrite all tags" technique
- [./2026-03-xygeni-action.md](./2026-03-xygeni-action.md) — GitHub Actions tag poisoning via stolen maintainer credentials
- [./2026-03-bittensor-pypi.md](./2026-03-bittensor-pypi.md) — Multi-stage credential stealer with self-deletion via a different ecosystem
