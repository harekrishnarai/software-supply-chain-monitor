# GlassWorm Returns: Invisible Unicode Mass Campaign

**Date:** March 2026
**Ecosystem:** GitHub, npm, VS Code / OpenVSX
**Severity:** Critical
**Type:** Invisible Unicode steganography / Infostealer / Worm / Blockchain C2
**Sources:**
- [Aikido Security — Glassworm Is Back: A New Wave of Invisible Unicode Attacks Hits Hundreds of Repositories](https://www.aikido.dev/blog/glassworm-returns-unicode-attack-github-npm-vscode)
- [Aikido Security — Glassworm Strikes Popular React Native Phone Number Packages](https://www.aikido.dev/blog/glassworm-strikes-react-packages-phone-numbers)
- [StepSecurity — Malicious npm Releases Found in Popular React Native Packages](https://www.stepsecurity.io/blog/malicious-npm-releases-found-in-popular-react-native-packages---130k-monthly-downloads-compromised)

---

## Summary

In March 2026, the GlassWorm threat actor launched a coordinated mass campaign across GitHub, npm, and the VS Code marketplace, representing the most prolific wave yet in a campaign that began in March 2025. At least **150+ GitHub repositories** were compromised between March 3–9, plus additional npm packages and VS Code extensions. The attack uses invisible Unicode characters (Private Use Area codepoints `U+FE00`–`U+FE0F` and `U+E0100`–`U+E01EF`) embedded in what appears to be empty JavaScript strings to hide a malicious decoder. When JavaScript evaluates the string, the invisible characters are decoded into a full payload and passed to `eval()`.

The campaign targets open source maintainers whose GitHub credentials were previously stolen by GlassWorm's own credential-theft module (delivered through compromised VS Code/Cursor extensions). The Solana C2 address used by this campaign — `BjVeAjPrSKFiingBn4vZvghsGj9KCE8AJVtbc9S8o8SC` — is also used by the concurrent ForceMemo campaign, confirming both waves share infrastructure and likely the same threat actor.

---

## Compromised Artifacts

**GitHub Repositories (sample — 150+ total, March 3–9, 2026):**

| Repository | Stars |
|-----------|-------|
| `pedronauck/reworm` | 1,460 |
| `pedronauck/spacefold` | 62 |
| `anomalyco/opencode-bench` | 56 |
| `doczjs/docz-plugin-css` | 39 |
| `uknfire/theGreatFilter` | 38 |
| `sillyva/rpg-schedule` | 37 |
| `wasmer-examples/hono-wasmer-starter` | 8 |

**npm Packages:**

| Package | Malicious Versions | Date |
|---------|-------------------|------|
| `@aifabrix/miso-client` | 4.7.2 | Mar 12, 2026 |
| `@iflow-mcp/watercrawl-watercrawl-mcp` | 1.3.0–1.3.4 | Mar 12, 2026 |

**React Native npm Packages (130K+ combined monthly downloads):**

| Package | Malicious Version |
|---------|-----------------|
| `@react-native-community/phone-number` | Compromised release |

**VS Code Extensions:**

| Extension | Malicious Version | Date |
|-----------|-----------------|------|
| `quartz.quartz-markdown-editor` | 0.3.0 | Mar 12, 2026 |

---

## How It Worked

### Invisible Unicode Injection

The core technique embeds an entire malicious payload as invisible Unicode variation selector characters inside what visually appears to be an empty template literal backtick string:

```javascript
const s = v => [...v].map(w => (
  w = w.codePointAt(0),
  w >= 0xFE00 && w <= 0xFE0F ? w - 0xFE00 :
  w >= 0xE0100 && w <= 0xE01EF ? w - 0xE0100 + 16 : null
)).filter(n => n !== null);
eval(Buffer.from(s(``)).toString('utf-8'));
```

The backtick string `\`\`` appears completely empty in every editor, terminal, diff view, and code review interface — but is packed with invisible characters that decode to a full malicious payload.

### Blockchain C2

The decoded payload queries the Solana blockchain address `BjVeAjPrSKFiingBn4vZvghsGj9KCE8AJVtbc9S8o8SC` via 9 different RPC endpoints for resilience, reads a base64-encoded payload URL from a transaction memo, downloads and executes the second stage (which steals tokens, credentials, and secrets using the same infrastructure as ForceMemo).

### Credential Theft → Account Takeover → Injection Loop

GlassWorm's stage-3 payload (delivered via compromised VS Code extensions in earlier waves) includes a credential theft module that harvests GitHub tokens from `git credential fill`, VS Code extension storage, `~/.git-credentials`, and the `GITHUB_TOKEN` environment variable. Stolen tokens are then used to inject the invisible Unicode payload into the victim's repositories — creating a self-propagating loop.

### Campaign History

| Period | Activity |
|--------|----------|
| March 2025 | Aikido first discovers malicious npm packages hiding payloads via PUA Unicode |
| May 2025 | Aikido publishes blog detailing invisible Unicode supply chain abuse |
| October 17, 2025 | Compromised extensions on Open VSX using same technique |
| October 31, 2025 | Pivot to GitHub repository injections |
| December 2025 | Wave 4 — pivots to macOS with trojanized crypto wallets (documented by Koi Security) |
| March 3–9, 2026 | Mass wave: 150+ GitHub repos; npm packages; VS Code marketplace |
| March 12–16, 2026 | Continued npm campaign targeting React Native packages (130K+ monthly downloads) |

---

## Timeline

| Date | Event |
|------|-------|
| 2026-03-03 | Earliest GitHub repository injections begin |
| 2026-03-08 | ForceMemo campaign begins using same Solana C2 address |
| 2026-03-09 | Last known GitHub repo injections in this wave |
| 2026-03-12 | npm packages and VS Code extension compromised |
| 2026-03-13 | Aikido publishes analysis; GitHub code search shows 151+ matches |
| 2026-03-16–18 | React Native phone number packages compromised (StepSecurity + Aikido) |
| Mar 17, 2026 | Aikido updates analysis with latest scope |

---

## Detection

```bash
# Search cloned repositories for the invisible Unicode decoder pattern
grep -rn "codePointAt\|FE00\|E0100\|E01EF" . --include="*.js" --include="*.ts" --include="*.mjs"

# GitHub code search for the specific decoder function (run in browser)
# https://github.com/search?q=codePointAt+0xFE00&type=code

# Scan for hidden Unicode in source files (requires specialized tooling)
# Using Aikido's SafeChain or similar malware scanner
npx @aikido/safechain scan .

# Check for the Solana C2 wallet address in decoded payloads
grep -r "BjVeAjPrSKFiingBn4vZvghsGj9KCE8AJVtbc9S8o8SC" . 2>/dev/null

# Check git log for suspicious committer dates vs author dates (ForceMemo overlap)
git log --format="%H %ai %ci %s" | awk '{if ($2 != $3) print "DATE MISMATCH:", $0}'

# Scan npm packages for invisible Unicode before install
# Using Aikido SafeChain wrapper around npm
```

---

## Remediation

1. **Scan all recently cloned/installed repositories** using a tool capable of detecting invisible Unicode characters (Aikido SafeChain, custom grep for `\xef\xb8\x80`–`\xef\xb8\x8f` byte ranges).
2. **Rotate GitHub tokens** if any VS Code or Cursor extension installed since October 2025 may have been compromised.
3. **Check for the persistence file** `~/init.json` and downloaded `~/node-v22*` from the second-stage payload.
4. **Remove and reinstall** any npm packages from the affected list at malicious versions.
5. **Enable 2FA on GitHub** and use hardware security keys where possible to make token theft insufficient for takeover.
6. **Block Solana RPC endpoints** from developer CI environments (these should never be contacted by build tooling): `api.mainnet-beta.solana.com`, `solana-mainnet.gateway.tatum.io`, etc.

---

## Lessons Learned

- Invisible Unicode attacks are undetectable by visual code review and most static linters — the only defense is active tooling that inspects the raw bytes of source files.
- A self-reinforcing loop (extension → credential theft → repo injection → more downloads) can spread a campaign indefinitely through the open source ecosystem with minimal ongoing attacker effort.
- Shared C2 infrastructure across multiple concurrent campaigns (GlassWorm + ForceMemo) indicates a single organized threat actor operating coordinated multi-vector supply chain attacks.
- GitHub's code search can be used by defenders to find infected repositories (`codePointAt 0xFE00`), but affected repos are deleted faster than the index updates.

---

## Related Incidents

- [./2026-03-glassworm-canisterworm.md](./2026-03-glassworm-canisterworm.md) — Earlier GlassWorm/CanisterWorm wave (same actor, same C2)
- [./2026-03-forcememo.md](./2026-03-forcememo.md) — Concurrent ForceMemo campaign sharing the same Solana C2 address
- [./2025-late-shai-hulud-worm.md](./2025-late-shai-hulud-worm.md) — Related worm campaign in the same ecosystem
