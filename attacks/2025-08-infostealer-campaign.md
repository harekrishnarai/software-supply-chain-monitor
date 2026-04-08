# Info-Stealer Campaign (August 2025)

**Date:** August 2025
**Ecosystem:** npm
**Severity:** High
**Type:** Credential theft / Chrome browser infostealer
**Sources:** [sonatype.com](https://www.sonatype.com), [jfrog.com](https://jfrog.com)

---

## Summary

Eight sophisticated npm packages were discovered delivering **Chrome browser information stealers** targeting developer machines. The packages were designed to masquerade as legitimate React, Solana, and SDK-related utilities, making them plausible installs for frontend and Web3 developers.

Unlike simpler malware that targets environment variables or config files, this campaign specifically extracted credentials stored inside the **Chrome browser** — including saved passwords, session cookies, and authentication tokens — significantly broadening the credential theft surface beyond the development environment itself.

---

## Compromised Packages

| Package | Malicious Version(s) |
|---------|---------------------|
| `react-sxt` | 2.4.1 |
| `react-sdk-solana` | 2.4.1 |
| `react-native-control` | 2.4.1 |
| `revshare-sdk-api` | 2.4.1 |
| `revshare-sdk-apii` | 2.4.1 |
| `toolkdvv` | 1.0.0, 1.1.0 |

> Note: All five `2.4.1` packages share an identical version number, a strong indicator of a coordinated simultaneous publish by the same threat actor.

---

## How It Worked

### Package Design

The packages were constructed to appear credible:
- Names mimicked real ecosystem patterns (`react-sdk-*`, `react-native-*`, `revshare-*`)
- Packages contained partial legitimate functionality to avoid immediate suspicion
- The infostealer payload was obfuscated or encoded within the package's install scripts or main entry point

### Chrome Infostealer Mechanism

On installation or first execution, the payload:

1. Located Chrome's user data directory (platform-specific):
   - **Linux:** `~/.config/google-chrome/`
   - **macOS:** `~/Library/Application Support/Google/Chrome/`
   - **Windows:** `%LOCALAPPDATA%\Google\Chrome\User Data\`

2. Extracted:
   - **Saved passwords** from the `Login Data` SQLite database
   - **Session cookies** from the `Cookies` SQLite database
   - **Browser history** for reconnaissance
   - **Local storage** — often contains auth tokens for web apps

3. The Chrome encryption key (stored in `Local State`) was decrypted using the OS keychain or DPAPI (Windows), allowing the attacker to read plaintext credentials even from an encrypted Chrome profile.

4. Exfiltrated the collected data to an attacker-controlled endpoint.

### Why Chrome Specifically?

Developers frequently store credentials for:
- GitHub, GitLab, npm, and other developer platforms
- Cloud provider consoles (AWS, GCP, Azure)
- Internal tooling and staging environments

Browser-stored credentials often survive even after developers rotate API keys stored in dotfiles, making Chrome a high-value secondary target.

---

## Detection

```bash
# Check for compromised packages
npm ls react-sxt
npm ls react-sdk-solana
npm ls react-native-control
npm ls revshare-sdk-api
npm ls revshare-sdk-apii
npm ls toolkdvv
```

Check for unexpected outbound connections at the time of install:
```bash
# On Linux/macOS, monitor network during npm install
sudo tcpdump -i any -n 'not port 443 and not port 80' &
npm install <package>
```

---

## Remediation

1. **Remove all listed packages immediately**
2. **Change browser-saved passwords** for all developer accounts (GitHub, npm, cloud consoles, etc.)
3. **Revoke and rotate** all active sessions and API tokens for services you were logged into in Chrome
4. Consider using a **dedicated browser profile** for development credentials, separate from personal browsing
5. Review Chrome's saved passwords (`chrome://settings/passwords`) and clear any that are sensitive
6. Enable hardware security keys (FIDO2) where possible — session cookies stolen from Chrome cannot be replayed against FIDO2-protected accounts

---

## Lessons Learned

- **Install scripts are a primary attack vector.** Audit `preinstall`/`postinstall` scripts before running `npm install` on unfamiliar packages
- **Version number coordination is a red flag.** Multiple packages releasing the same version simultaneously, especially `x.y.1` or `1.0.0`, warrants scrutiny
- **Browser credentials are developer credentials.** Treat Chrome's stored passwords with the same security posture as your `.env` files

---

## Related Incidents

- [GlassWorm / CanisterWorm (March 2026)](./2026-03-glassworm-canisterworm.md) — similar credential harvesting objectives
- [The Great npm Heist (September 2025)](./2025-09-great-npm-heist.md) — same threat period
