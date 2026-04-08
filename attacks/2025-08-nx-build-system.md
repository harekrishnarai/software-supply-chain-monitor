# NX Build System Compromise

**Date:** August 26–27, 2025
**Ecosystem:** npm, VS Code Extensions
**Severity:** Critical
**Type:** Postinstall hook credential theft / GitHub token abuse / IDE extension vector
**Sources:**
- [semgrep.dev — Security Alert | NX Compromised to Steal Wallets and Credentials](https://semgrep.dev/blog/2025/security-alert-nx-compromised-to-steal-wallets-and-credentials/)
- [stepsecurity.io — Supply Chain Security Alert: NX Build System Compromised with Data-Stealing Malware](https://www.stepsecurity.io/blog/supply-chain-security-alert-popular-nx-build-system-package-compromised-with-data-stealing-malware)
- [Official advisory: GHSA-cxm3-wv7p-598c](https://github.com/nrwl/nx/security/advisories/GHSA-cxm3-wv7p-598c)

---

## Summary

On August 26, 2025, 8 malicious versions of `nx` — the popular monorepo build system used by **2.5 million developers daily** — were published to npm. The compromised versions contained a `postinstall` hook that stole crypto wallets, API keys, npm tokens, and GitHub tokens from developers' machines, then exfiltrated them as a double-Base64 encoded file into a **public GitHub repository** named `s1ngularity-repository` created on the victim's own account.

At least **1,400 developers** were immediately impacted. The attack was compounded by a second wave where the stolen GitHub tokens were used to **set private repositories to public**, dramatically widening exposure of confidential codebases.

The attack also spread via the **NX Console VS Code extension** (`nrwl.angular-console`), which auto-updated to the malicious nx version on IDE launch — meaning developers who never ran `npm install` themselves were still compromised simply by opening VS Code.

---

## Compromised Artifacts

| Package | Malicious Versions |
|---------|-------------------|
| `nx` | 20.9.0, 20.10.0, 20.11.0, 20.12.0, 21.5.0, 21.6.0, 21.7.0, 21.8.0 |
| NX Console VS Code extension (`nrwl.angular-console`) | 18.6.30 – 18.65.1 |
| All `@nx/*` scoped packages | Corresponding versions |

---

## How It Worked

### Initial Compromise Vector

The root cause was a **GitHub Actions workflow injection vulnerability** exploited via **outdated PR branches**. An attacker was able to inject malicious content into an nx CI/CD pipeline through an old, unmerged PR branch whose workflow definition ran in a privileged context. This gave the attacker the ability to modify and publish npm packages under the `nrwl` account.

### Postinstall Payload (`telemetry.js`)

The malicious `postinstall` script (`telemetry.js`) ran automatically on `npm install` and performed a multi-stage credential harvest:

**Stage 1 — Credential Harvesting**

The script scanned the local filesystem for:
- Crypto wallet files (keystore files, `*.key`, `wallet`, `metamask`, `electrum`, `ledger`, `trezor`, `exodus`, `phantom`, `solflare`)
- `.npmrc` — contains npm auth tokens
- Environment variables — AWS keys, GitHub tokens, API keys
- `~/.ssh/` — SSH private keys
- Any `.env` files in the accessible path

**Stage 2 — AI Agent Exploitation Attempt**

The script checked whether **Claude Code**, **Gemini CLI**, or **Amazon Q CLI** was installed on the machine. If found, it attempted to send a prompt instructing the AI agent to recursively search the filesystem for wallet-related files and write paths to `/tmp/inventory.txt`.

Claude refused to assist:
> *"I can't help with creating tools to search for and inventory wallet files, private keys, or other sensitive security materials on a system."*

Regardless, the script fell back to its own filesystem scanning when AI agents refused or were unavailable.

**Stage 3 — Exfiltration**

All harvested data was:
1. Encoded as a double Base64 string
2. Written to a file named `results.b64`
3. Pushed to a **newly created public GitHub repository** named `s1ngularity-repository` on the victim's own GitHub account

The attacker then simply searched GitHub for any repo named `s1ngularity-repository`, fetched `results.b64`, and decoded it.

**Stage 4 — Persistence Modification**

The script appended `sudo shutdown -h 0` to `~/.zshrc` and `~/.bashrc`. When the developer next opened a terminal session, they were prompted for their sudo password, and upon entering it — their machine would be shut down. (This also conveniently attempts to cover traces.)

**Stage 5 — Second Wave via Stolen Tokens**

The stolen GitHub tokens were subsequently used (by an automated script) to **change private repositories to public** — exposing confidential source code, secrets embedded in repos, and internal documentation.

### VS Code Extension Vector

The NX Console extension (`nrwl.angular-console`) called `npx -y nx@latest --version` on IDE activation — an auto-update check that silently installed the latest nx version. Developers who:
- Had the extension installed
- Opened VS Code between **August 26, 6:37 PM EDT – 10:44 PM EDT**

...were compromised even without ever running `npm install` for nx themselves.

---

## Timeline

| Time | Event |
|------|-------|
| Aug 26, ~6:00 PM PDT | 8 malicious nx versions published |
| Aug 26, ~8:30 PM PDT | First user reports suspicious `s1ngularity-repository` on GitHub |
| Aug 26, ~10:44 PM PDT | npm removes compromised versions |
| Aug 26, ~11:45 PM PDT | nrwl removes compromised npm account |
| Aug 27, ~1:00 AM PDT | Additional `@nx/` scoped packages added to scope of compromise |
| Aug 27, ~2:00 AM PDT | GitHub begins making `s1ngularity-repository` repos private and deindexing them |
| Aug 27, ~3:20 AM PDT | npm removes other affected packages |
| Aug 27, ~5:30 AM PDT | New NX Console VS Code extension released without auto-update |
| Aug 27, ~8:50 AM PDT | All nx npm packages require 2FA; npm token publishing disabled (Trusted Publisher only) |
| Aug 27, ~10:55 AM PDT | Malicious commit linked to GitHub Actions workflow injection |
| Aug 27, ~11:50 AM PDT | Attack reproduced — outdated PR branches confirmed as the injection method |
| Aug 27, ~12:10 PM PDT | All outdated branches rebased to remove vulnerable pipeline |
| Aug 28, ~12:43 PM PDT | Second wave reported: stolen GitHub tokens used to set private repos to public |

---

## Detection

```bash
# Check if you are running a malicious nx version
npm ls nx

# Affected versions: 20.9.0, 20.10.0, 20.11.0, 20.12.0, 21.5.0, 21.6.0, 21.7.0, 21.8.0

# Check npm logs for postinstall execution
ls ~/.npm/_logs/
# Look for entries running: node telemetry.js

# Check if the malicious postinstall hash is present
find . -type f -name "*.js" -exec sha256sum {} \; | grep -E \
  "de0e25a3e6c1e1e5998b306b7141b3dc4c0088da9d7bb47c1c00c91e6e4f85d6"

# Check for the shutdown backdoor in shell profiles
grep "shutdown" ~/.zshrc ~/.bashrc

# Check for s1ngularity-repository creation in your GitHub security log
# https://github.com/settings/security-log?q=operation%3AcreateRepo

# Check if any repos were renamed (sign of second wave)
# https://github.com/settings/security-log?q=action%3Arepo.rename

# If you use Claude Code, check for the malicious prompt in your project logs
grep -r "Recursively search local paths on Linux/macOS" --include="*.jsonl" ~/.claude/projects

# Check VS Code extension usage
# Verify if nrwl.angular-console is/was installed
ls ~/.vscode/extensions/ | grep nrwl
```

**Semgrep Rule** (open source, MIT licensed):
```bash
semgrep --config r/oqUk5lJ/semgrep.ssc-mal-resp-2025-08-nx-build-compromised
```

---

## Remediation

1. **Rotate all credentials immediately** if you were using an affected nx version:
   - npm tokens: https://www.npmjs.com/settings/~/tokens
   - GitHub tokens: https://github.com/settings/tokens (rotate and audit for unauthorized repo visibility changes)
   - AWS/GCP/Azure keys
   - Any API keys present in environment variables

2. **Update nx:** `npm uninstall nx && npm install nx@latest`

3. **Clear npm cache:** `npm cache clean --force`

4. **Remove the shutdown backdoor from shell profiles:**
   ```bash
   # Remove any 'sudo shutdown' lines added by the malware
   grep -v "sudo shutdown" ~/.zshrc > ~/.zshrc.tmp && mv ~/.zshrc.tmp ~/.zshrc
   grep -v "sudo shutdown" ~/.bashrc > ~/.bashrc.tmp && mv ~/.bashrc.tmp ~/.bashrc
   ```

5. **Audit GitHub** for any private repositories that were made public — check your organization's audit log

6. **Update NX Console VS Code extension** to the latest version (the auto-update feature has been removed)

---

## Lessons Learned

- **IDE extensions are a supply chain attack surface.** Extensions that auto-update or call `npx <package>@latest` on activation can pull in compromised packages without any developer action.
- **Outdated PR branches can be exploited for workflow injection.** Any GitHub Actions workflow that checks out code from a long-lived or old PR branch in a privileged context (`pull_request_target`) can be a vector. Clean up stale PR branches and audit workflow trigger conditions.
- **AI agents are being targeted as exfiltration amplifiers.** The NX malware attempted to leverage Claude/Gemini/Q CLI to recursively enumerate wallet files. As AI coding agents gain filesystem and shell access, they become high-value targets for social engineering at scale.
- **Double-layer exfiltration via the victim's own GitHub account is stealthy.** Rather than exfiltrating to an attacker-owned server, the malware created a public repo in the victim's account — blending into normal developer activity and bypassing outbound domain-based network monitoring.
- **Token theft enables cascading impact.** Stolen GitHub tokens don't just expose the immediate host — they can expose every private repository, secret, and teammate in the organization.

---

## Related Incidents

- [Info-Stealer Campaign (Aug 2025)](./2025-08-infostealer-campaign.md) — concurrent month; similar credential theft objective
- [Shai-Hulud Worm (Sep–Nov 2025)](./2025-late-shai-hulud-worm.md) — followed days later; uses the same `s1ngularity`-style exfiltration to GitHub public repos
- [The Great npm Heist (Sep 2025)](./2025-09-great-npm-heist.md) — same threat period; foundational npm packages compromised
