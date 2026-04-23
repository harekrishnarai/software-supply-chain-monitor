# Strapi CMS npm Typosquats — Redis/PostgreSQL RCE, Docker Escape, Persistent Implant

**Date:** April 2026
**Ecosystem:** npm
**Severity:** Critical
**Type:** Typosquatting / postinstall RCE / Database exploitation / Persistent implant
**Sources:**
- [The Hacker News — 36 Malicious npm Packages Exploited Redis, PostgreSQL to Deploy Persistent Implants](https://thehackernews.com/2026/04/36-malicious-npm-packages-exploited.html)
- [SafeDep — Malicious npm Strapi Plugin: Events C2 Agent](https://safedep.io/malicious-npm-strapi-plugin-events-c2-agent/)

---

## Summary

In early April 2026, 36 malicious npm packages were discovered masquerading as Strapi CMS community plugins. All packages were uploaded by four sock puppet accounts (`umarbek1233`, `kekylf12`, `tikeqemif26`, `umar_bektembiev1`) across a 13-hour window. The packages followed a uniform structure (three files: `package.json`, `index.js`, `postinstall.js`), used version `3.6.8` to simulate maturity as a Strapi v3 plugin, and adopted the `strapi-plugin-*` naming convention that developers naturally expect from the ecosystem — while legitimate Strapi plugins are scoped under `@strapi/`.

The attack evolved across eight distinct payload stages, beginning with aggressive Redis RCE and Docker container escape techniques, then pivoting to reconnaissance and credential harvesting when those approaches failed to land, and finally settling on direct PostgreSQL database exploitation using hard-coded credentials and deployment of a persistent reverse shell implant. The hard-coded credentials and hostname `prod-strapi` in later payloads strongly suggest this was a targeted attack against a specific cryptocurrency exchange platform — a target the attackers had likely already partially compromised through prior means.

---

## Compromised Artifacts

| Package | Notes |
|---------|-------|
| `strapi-plugin-cron` | |
| `strapi-plugin-config` | |
| `strapi-plugin-server` | |
| `strapi-plugin-database` | |
| `strapi-plugin-core` | |
| `strapi-plugin-hooks` | |
| `strapi-plugin-monitor` | |
| `strapi-plugin-events` | Primary C2 agent payload |
| `strapi-plugin-logger` | |
| `strapi-plugin-health` | |
| `strapi-plugin-sync` | |
| `strapi-plugin-seed` | |
| `strapi-plugin-locale` | |
| `strapi-plugin-form` | |
| `strapi-plugin-notify` | |
| `strapi-plugin-api` | |
| `strapi-plugin-sitemap-gen` | |
| `strapi-plugin-nordica-tools` | |
| `strapi-plugin-nordica-sync` | |
| `strapi-plugin-nordica-cms` | |
| `strapi-plugin-nordica-api` | |
| `strapi-plugin-nordica-recon` | |
| `strapi-plugin-nordica-stage` | |
| `strapi-plugin-nordica-vhost` | |
| `strapi-plugin-nordica-deep` | |
| `strapi-plugin-nordica-lite` | |
| `strapi-plugin-nordica` | |
| `strapi-plugin-finseven` | |
| `strapi-plugin-hextest` | |
| `strapi-plugin-cms-tools` | |
| `strapi-plugin-content-sync` | |
| `strapi-plugin-debug-tools` | |
| `strapi-plugin-health-check` | |
| `strapi-plugin-guardarian-ext` | Targets Guardarian (crypto exchange) specifically |
| `strapi-plugin-advanced-uuid` | |
| `strapi-plugin-blurhash` | |

All 36 packages version `3.6.8`. Legitimate Strapi plugins use the `@strapi/` namespace scope.

---

## How It Worked

### Entry Point: Postinstall Hook

The malicious code runs entirely through the `postinstall.js` script, triggered automatically on `npm install` with no user interaction. The hook executes with the same privileges as the installing user — gaining root-equivalent access in CI/CD runners and Docker containers.

### Eight Payload Stages (in evolution order)

**Stage 1 — Redis RCE via Crontab Injection:**
Injected a crontab entry via Redis `CONFIG SET dir` and `CONFIG SET dbfilename` to write a cron job that downloads and executes a shell script from a remote server every minute. The downloaded script:
- Wrote a PHP web shell and Node.js reverse shell to Strapi's public uploads directory
- Attempted disk scanning for secrets (Elasticsearch credentials, cryptocurrency wallet seed phrases)
- Exfiltrated a Guardarian API module

**Stage 2 — Redis + Docker Container Escape:**
Combined Redis crontab exploitation with Docker `CONFIG SET dir` pointing to the host filesystem (via `/proc/1/root` or similar escape path) to write shell payloads outside the container. Also launched a direct Python reverse shell on port 4444 and injected a reverse shell trigger into `node_modules/` via Redis replication.

**Stage 3 — Simplified Reverse Shell + Redis Persistence:**
Deployed a direct reverse shell and used Redis to write a persistent shell downloader to disk.

**Stage 4 — Reconnaissance / Credential Harvesting:**
Pivoted from active exploitation to reconnaissance after earlier approaches failed. Scanned environment variables and PostgreSQL database connection strings. Indicates the attacker recognized the active payloads weren't reaching their target.

**Stage 5 — Expanded Credential Harvester:**
Full reconnaissance sweep collecting: environment dumps, Strapi configuration files, Redis database extraction (`INFO`, `DBSIZE`, `KEYS` commands), network topology mapping, Docker/Kubernetes secrets, cryptographic keys, and cryptocurrency wallet files.

**Stage 6 — PostgreSQL Exploitation with Hard-Coded Credentials:**
Connected directly to the target's PostgreSQL database using hard-coded credentials. Queried Strapi-specific tables for secrets, dumped rows matching cryptocurrency-related patterns (`wallet`, `transaction`, `deposit`, `withdraw`, `hot`, `cold`, `balance`), and attempted connections to six Guardarian-specific databases. The possession of hard-coded credentials implies prior compromise of the target environment.

**Stage 7 — Persistent Implant Deployment:**
Deployed a persistent implant designed for long-term remote access to a specific target hostname (`prod-strapi`). The implant survived reboots and maintained a persistent remote shell.

**Stage 8 — Targeted Credential Theft + Persistent Shell:**
Final stage combined hard-coded path credential scanning with a persistent reverse shell, refining earlier approaches into a stable, targeted payload.

### Targeting Assessment

The evolution narrative tells a clear story: the attacker started aggressive (Redis RCE, Docker escape) but found those techniques weren't reliably landing, pivoted to reconnaissance and data collection, then used hard-coded PostgreSQL credentials for direct database access, and finally settled on persistent access with targeted credential theft. The hard-coded credentials, the focus on Guardarian-specific tables and databases, and the named target hostname `prod-strapi` all strongly indicate this was a targeted attack against a cryptocurrency exchange or processor — likely Guardarian — with the npm campaign serving as a secondary infection vector to reach adjacent developer environments.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Early Apr 2026 | 36 packages uploaded by 4 sock puppet accounts across ~13 hours |
| 2026-04-05 | SafeDep publishes analysis; The Hacker News reports on campaign |
| 2026-04-05 | npm registry removes all 36 packages |

---

## Detection

```bash
# Check for any of the 36 malicious packages in installed dependencies
npm ls 2>/dev/null | grep -E "strapi-plugin-(cron|config|server|database|core|hooks|monitor|events|logger|health|sync|seed|locale|form|notify|api|sitemap-gen|nordica|finseven|hextest|cms-tools|content-sync|debug-tools|health-check|guardarian-ext|advanced-uuid|blurhash)"

# Search lockfiles for these packages
grep -r "strapi-plugin-cron\|strapi-plugin-config\|strapi-plugin-nordica\|strapi-plugin-guardarian\|strapi-plugin-finseven" \
  --include="package-lock.json" \
  --include="yarn.lock" \
  --include="pnpm-lock.yaml" .

# Check for attacker sock-puppet npm account installations
# (packages published by: umarbek1233, kekylf12, tikeqemif26, umar_bektembiev1)
npm info strapi-plugin-events 2>/dev/null | grep "_npmUser"

# Check Redis for injected crontab configuration (Stage 1/2 indicator)
redis-cli CONFIG GET dir
redis-cli CONFIG GET dbfilename

# Check for PHP webshell in Strapi uploads directory
find ./public/uploads -name "*.php" -newer package.json 2>/dev/null

# Check for unexpected reverse shell processes
ss -tlnp | grep ":4444"
netstat -tlnp 2>/dev/null | grep ":4444"

# Check for unexpected crontab entries
crontab -l 2>/dev/null | grep -v "^#"

# Check CI/CD environment variables for Strapi DB credentials against known-bad patterns
env | grep -E "DATABASE|POSTGRES|PG_" | head -20

# Check for persistent implant (hostname-based target)
hostname | grep "prod-strapi"
```

---

## Remediation

1. **Remove all 36 packages immediately** from any environment where they were installed.
2. **Assume full compromise** if any of these packages were installed — rotate all credentials:
   - Database connection strings (PostgreSQL, Redis)
   - Strapi admin credentials
   - Cloud provider API keys (AWS, GCP, Azure)
   - npm tokens, GitHub PATs
   - Cryptocurrency wallet keys or exchange API credentials
3. **Audit Redis configuration** — check for injected crontab via `CONFIG GET dir` / `CONFIG GET dbfilename`; reset to safe defaults.
4. **Inspect the Strapi `public/uploads/` directory** for PHP web shells or unexpected Node.js files.
5. **Review PostgreSQL audit logs** for unexpected queries on Strapi tables, especially any containing wallet/transaction/financial data.
6. **Scan for the persistent implant:** check for unexpected services connecting to external hosts on non-standard ports, particularly from processes named `prod-strapi`.
7. **Lock down package namespaces** — add `@strapi/` scoped packages only to allowlists; block unscoped `strapi-plugin-*` packages via registry policy or `.npmrc`:
   ```
   @strapi:registry=https://registry.npmjs.org/
   ```
8. **Enable npm provenance** for your own Strapi plugin packages and require it for all installed dependencies where possible.

---

## Lessons Learned

- **Namespace impersonation is a persistent threat:** The absence of an official scoped namespace (`@strapi/`) for community plugins created a natural typosquat opportunity. Ecosystems should encourage or enforce scoped namespacing for plugin ecosystems.
- **Payload evolution within a campaign reveals attacker learning:** The eight-stage evolution from Redis RCE → Docker escape → reconnaissance → DB exploitation → persistence shows attackers adapting in real time. This iterative pattern is a signature of targeted, persistent threat actors rather than opportunistic script kiddies.
- **Hard-coded credentials in public packages are a forensic goldmine:** The presence of specific database credentials and a target hostname in Stage 6/7 payloads confirmed a prior breach of the target environment, enabling law enforcement and incident response to scope the larger intrusion.
- **postinstall hooks remain a high-risk attack surface:** Even in 2026, the npm `postinstall` hook executes arbitrary code with full user privileges on install. Organizations running Strapi or similar CMS ecosystems should disable install scripts in CI unless explicitly required and audited.
- **Sock-puppet account velocity is a detection signal:** 36 packages published by 4 accounts in 13 hours should trigger automated registry abuse detection. Account age + publication volume + package naming pattern are reliable heuristics.

---

## Related Incidents

- [./2026-02-npm-gambling-backdoor.md](./2026-02-npm-gambling-backdoor.md) — npm typosquats with env-gated activation
- [./2026-01-gwagon-npm-infostealer.md](./2026-01-gwagon-npm-infostealer.md) — Fake UI library targeting crypto wallets
- [./2025-12-neoshadow-npm.md](./2025-12-neoshadow-npm.md) — npm typosquatting campaign with MSBuild LOLbin C2
- [./2026-03-canisterworm-npm.md](./2026-03-canisterworm-npm.md) — npm campaign with Kubernetes wiper targeting cryptocurrency infrastructure
