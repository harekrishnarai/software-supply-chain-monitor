# Mini Shai-Hulud Expands to PHP — intercom/intercom-php@5.0.2 Packagist Compromise

**Date:** April 2026
**Ecosystem:** Packagist / Composer (PHP)
**Severity:** High
**Type:** Account compromise / Composer plugin injection / Worm campaign cross-ecosystem expansion
**Sources:**
- [Semgrep — Malicious Intercom PHP Package Spreads Mini Shai-Hulud Attack to Packagist via Composer Plugin](https://semgrep.dev/blog/2026/malicious-intercom-php-package-spreads-mini-shai-hulud-attack-to-packagist-via-composer-plugin)

---

## Summary

On April 30, 2026 — the same day the Mini Shai-Hulud campaign compromised the `lightning` PyPI package and the `intercom-client` npm package — attackers expanded the campaign into the PHP ecosystem by overwriting `intercom/intercom-php` version `5.0.2` on Packagist with malicious code. This marks the first documented instance of the Mini Shai-Hulud worm family crossing from npm/PyPI into the PHP/Composer ecosystem, and demonstrates a deliberate same-day, multi-registry blitz targeting all official Intercom SDK packages simultaneously.

The attack exploited a fundamental property of the Packagist architecture: because Packagist does not host package tarballs but instead indexes VCS repository URLs, there is no pre-publish quarantine gate. Once an attacker compromises a maintainer's GitHub account and pushes a malicious tag, a webhook fires within minutes and Packagist immediately serves the new version to any developer running `composer update`. The malicious version converted `intercom/intercom-php` into a Composer plugin — PHP's install-time execution mechanism — causing the payload to run automatically whenever any project installs or updates the package.

The payload is functionally identical to the npm/PyPI variants: it downloads the Bun JavaScript runtime, executes a heavily obfuscated `router_runtime.js` credential stealer, and exfiltrates stolen secrets (GitHub tokens, SSH keys, cloud credentials, environment variables) to the C2 domain `zero.masscan.cloud`. The `intercom/intercom-php` and `intercom-client` npm packages together see approximately 400,000 weekly downloads, and are commonly installed in backend services, developer environments, and CI/CD pipelines — making them high-value targets for credential theft.

---

## Compromised Artifacts

| Package | Ecosystem | Malicious Version | Weekly Downloads |
|---------|-----------|-------------------|-----------------|
| `intercom/intercom-php` | Packagist (PHP/Composer) | `5.0.2` | ~400K (combined with intercom-client npm) |

---

## How It Worked

### Entry Point: Composer Plugin Injection

The Packagist ecosystem has no upload-time malware scanning, unlike npm and PyPI (which introduced such pipelines after prior high-profile incidents). The attacker pushed a malicious `5.0.2` tag to the `intercom/intercom-php` GitHub repository after compromising the maintainer's account. The Packagist webhook fired within minutes, making the malicious version immediately available.

The core mechanism was converting the package into a **Composer plugin** via `src/composerPlugin.php`. A Composer plugin is a special package type that hooks into Composer's event system:

```json
// composer.json (malicious)
{
  "type": "composer-plugin",
  "extra": {
    "class": "Intercom\\ComposerPlugin"
  }
}
```

When any project runs `composer install` or `composer update` with this package in scope, Composer automatically invokes the plugin class. This provides install-time code execution equivalent to npm's `postinstall` hook — but without the same level of scrutiny, since Composer plugins are a legitimate and widely-used mechanism.

### Payload Mechanics: Bun + Obfuscated JS

`src/composerPlugin.php` invoked `setup-intercom.sh`, a shell script that:

1. Downloaded the Bun JavaScript runtime binary from a remote host
2. Executed `router_runtime.js` — an 11+ MB heavily obfuscated JavaScript payload shared with the npm and PyPI variants of the Mini Shai-Hulud campaign
3. Wrote a lock file to `/tmp/tmp.987654321.lock` to prevent double-execution on the same machine

The use of a cross-runtime JavaScript payload (Bun) running inside a PHP project installation is a deliberate evasion choice: PHP-focused security tooling is not instrumented to detect JavaScript execution during Composer operations.

### Credential Exfiltration

The `router_runtime.js` payload — identical to the one deployed in the SAP npm, intercom-client npm, and lightning PyPI attacks — swept the victim environment for:

- GitHub tokens (environment variables and `~/.config/gh/hosts.yml`)
- SSH private keys (`~/.ssh/`)
- Cloud provider credentials (AWS `~/.aws/credentials`, GCP `~/.config/gcloud/`, Azure `~/.azure/`)
- Environment variables (full `process.env` dump)
- Kubernetes tokens (`~/.kube/config`)
- npm publishing tokens

Stolen data was encrypted and POSTed to `zero.masscan.cloud`.

### AI Agent and IDE Persistence

Consistent with other Mini Shai-Hulud variants, the payload also installed persistence artifacts targeting AI development tooling:

- `.claude/settings.json` — modified to hijack Claude Code MCP configuration
- `.claude/router_runtime.js` and `.claude/setup.mjs` — persistent re-execution hooks
- `.vscode/tasks.json` and `.vscode/setup.mjs` — VS Code workspace task persistence

### How Packagist Differs from npm/PyPI

The Semgrep write-up includes a precise architectural explanation of the attack surface:

- **No tarball hosting:** Packagist serves metadata and indexes GitHub/GitLab/Bitbucket repo URLs. The code itself comes from the VCS. There is no tarball upload step that could be scanned.
- **No pre-publish quarantine:** npm and PyPI introduced malware-scanning pipelines after prior high-profile incidents; Packagist has not.
- **Transitive exposure:** If a popular library depends on the compromised package, every downstream project picks it up on `composer update` without ever having explicitly required it. Replication is a feature of Composer's dependency resolution, not of the attack itself.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Apr 29, 2026 | SAP npm packages (@cap-js/sqlite, @cap-js/postgres, @cap-js/db-service, mbt) compromised — Mini Shai-Hulud wave begins |
| Apr 30, 2026 (morning) | `lightning` PyPI package versions 2.6.2 and 2.6.3 compromised |
| Apr 30, 2026 | `intercom-client` npm package version 7.0.4 compromised (~361K weekly downloads) |
| Apr 30, 2026 | `intercom/intercom-php@5.0.2` overwritten on Packagist with malicious Composer plugin code |
| Apr 30, 2026 | Semgrep publishes disclosure for the Packagist compromise |
| Apr 30, 2026 | Packagist removes/supersedes malicious version |

---

## Detection

```bash
# Check installed version of intercom/intercom-php
composer show intercom/intercom-php 2>/dev/null | grep versions

# Check lock file for the malicious version
grep -A2 '"intercom/intercom-php"' composer.lock 2>/dev/null | grep '"version"'
# Malicious version: 5.0.2

# Check for the Composer plugin type in the package's composer.json
cat vendor/intercom/intercom-php/composer.json 2>/dev/null | grep '"type"'
# Malicious if: "type": "composer-plugin"

# Check for the malicious plugin class
ls vendor/intercom/intercom-php/src/composerPlugin.php 2>/dev/null && echo "MALICIOUS FILE PRESENT"

# Check for Bun runtime download artifacts
ls /tmp/tmp.987654321.lock 2>/dev/null && echo "LOCK FILE FOUND - PAYLOAD MAY HAVE EXECUTED"

# Check for persistence artifacts left by the payload
ls .claude/router_runtime.js .claude/setup.mjs .claude/settings.json 2>/dev/null
ls .vscode/setup.mjs .vscode/tasks.json 2>/dev/null

# Check for results (exfiltrated data staging)
ls results/results-*.json 2>/dev/null

# Check for outbound connections to C2
grep -r "zero\.masscan\.cloud" /var/log/ ~/.npm/_logs/ 2>/dev/null

# Check for suspicious shell script in vendor directory
ls vendor/intercom/intercom-php/setup-intercom.sh 2>/dev/null && echo "MALICIOUS SCRIPT PRESENT"

# If Composer audit is available (Composer 2.4+)
composer audit 2>/dev/null | grep intercom
```

---

## Remediation

1. **Update or remove the package immediately:**
   ```bash
   composer update intercom/intercom-php
   # Verify you are on a clean version (5.0.3+ or the latest non-5.0.2 release):
   composer show intercom/intercom-php | grep versions
   ```

2. **If version 5.0.2 was installed, assume full credential compromise.** Rotate all of the following immediately:
   - GitHub personal access tokens and OAuth apps
   - SSH private keys (regenerate keypairs; remove old public keys from GitHub/GitLab/Bitbucket)
   - AWS IAM access keys (`aws iam list-access-keys` and rotate via `aws iam create-access-key`)
   - GCP service account keys (`gcloud iam service-accounts keys list`)
   - Azure credentials (`az account list` and rotate affected service principals)
   - Kubernetes kubeconfig credentials
   - npm publish tokens
   - Any `.env` / environment variable secrets present in the working directory

3. **Remove persistence artifacts:**
   ```bash
   rm -f .claude/router_runtime.js .claude/setup.mjs
   rm -f .vscode/setup.mjs .vscode/tasks.json
   rm -f /tmp/tmp.987654321.lock
   rm -f results/results-*.json package-updated.tgz
   # Review .claude/settings.json for unauthorized MCP server entries
   ```

4. **Check CI/CD pipelines:** If `composer install` ran in any CI pipeline with version 5.0.2, treat all secrets injected into that pipeline's environment as compromised. Review pipeline logs for the install step and any unusual outbound network calls.

5. **Review `composer.lock` in all projects** for the malicious version before it was removed from Packagist, as cached/vendored copies may persist in Docker images or deployment artifacts.

6. **Audit transitive dependencies:** Run `composer why intercom/intercom-php` to identify all packages that depend on it transitively and ensure they have been updated.

---

## Lessons Learned

- **Packagist's VCS-indexing architecture has no pre-publish scanning gate** — unlike npm and PyPI, which added malware-scanning pipelines after prior incidents; this makes the PHP ecosystem particularly vulnerable to compromised-maintainer attacks
- **Composer plugin abuse is PHP's equivalent to npm `postinstall`** — any package with `"type": "composer-plugin"` in `composer.json` executes code at install time; security tools should flag unexpected plugin-type changes in previously non-plugin packages
- **Cross-ecosystem simultaneous campaigns are now operational** — the attackers hit npm (SAP packages + intercom-client), PyPI (lightning), and Packagist (intercom-php) within a 24-hour window, requiring defenders to monitor all registries in parallel
- **Shared payloads across ecosystems lower attacker development costs** — the same `router_runtime.js` and Bun-based execution chain was reused across npm, PyPI, and now PHP; defenders can use this for cross-ecosystem IOC correlation
- **AI tooling is now a primary persistence target** — `.claude/settings.json` and `.vscode/tasks.json` modification for persistent MCP hijacking has now appeared in npm, PyPI, and PHP attacks; treat these config files as high-value artifacts during incident response

---

## IOCs

| Indicator | Type | Notes |
|-----------|------|-------|
| `intercom/intercom-php@5.0.2` | Malicious package | Packagist/PHP; overwritten with Composer plugin dropper |
| `zero.masscan.cloud` | C2 domain | Exfiltration endpoint; shared with Mini Shai-Hulud npm/PyPI variants |
| `setup-intercom.sh` | Malicious file | Shell dropper in vendor directory |
| `src/composerPlugin.php` | Malicious file | Composer plugin entry point |
| `router_runtime.js` | Malicious payload | 11+ MB obfuscated JS credential stealer (shared with npm/PyPI variants) |
| `/tmp/tmp.987654321.lock` | Execution artifact | Lock file indicating payload has run |
| `.claude/router_runtime.js` | Persistence artifact | AI tooling persistence |
| `.claude/setup.mjs` | Persistence artifact | AI tooling re-execution hook |
| `.claude/settings.json` | Persistence artifact | MCP server config hijack target |
| `.vscode/setup.mjs` | Persistence artifact | VS Code workspace persistence |
| `.vscode/tasks.json` | Persistence artifact | VS Code workspace task persistence |
| `results/results-*.json` | Exfiltration staging | Staged credential dumps |
| `package-updated.tgz` | Propagation artifact | Modified npm tarball for downstream spread |

---

## Related Incidents

- [Mini Shai-Hulud Wave — SAP npm Packages + intercom-client Multi-Cloud Worm (Apr 2026)](./2026-04-mini-shai-hulud-sap-npm.md) — Same-day campaign; intercom-client npm compromise and SAP packages
- [lightning PyPI Compromise — Mini Shai-Hulud in AI/ML Ecosystem (Apr 2026)](./2026-04-lightning-pypi-shai-hulud.md) — Same-day campaign; PyPI vector
- [Bitwarden CLI Shai-Hulud Third Coming (Apr 2026)](./2026-04-bitwarden-cli-shai-hulud.md) — Same worm family; shared Bun runtime + obfuscation toolchain
