# Resources — Supply Chain Security

A curated list of tools, monitoring services, and reading material for staying ahead of supply chain attacks.

---

## Detection & Auditing Tools

### Package Scanning
- **[Socket.dev](https://socket.dev)** — Real-time supply chain threat detection for npm, PyPI, and Go. Flags malicious packages before they're installed. Integrates with GitHub and CI/CD.
- **[Snyk](https://snyk.io)** — Vulnerability and license scanning across npm, pip, maven, and more. Includes transitive dependency analysis.
- **[JFrog Xray](https://jfrog.com/xray/)** — Deep recursive scanning of binaries and packages; strong on behavioral analysis.
- **[Sonatype Nexus IQ](https://www.sonatype.com/products/open-source-security-intelligence)** — Policy enforcement and component intelligence across the build pipeline.

### GitHub Actions Hardening
- **[StepSecurity Harden Runner](https://www.stepsecurity.io)** — Monitors outbound network traffic from GitHub Actions runners; detects unexpected exfiltration in real time.
- **[Actionlint](https://github.com/rhysd/actionlint)** — Static analysis for GitHub Actions workflows.
- **[pin-github-action](https://github.com/mheap/pin-github-action)** — CLI tool that replaces tag references in workflow files with pinned commit SHAs.

### npm-Specific
- **[npm audit](https://docs.npmjs.com/cli/v10/commands/npm-audit)** — Built-in vulnerability scanner. Run `npm audit` in any project.
- **[better-npm-audit](https://github.com/jeemok/better-npm-audit)** — Enhanced output and filtering for `npm audit`.
- **[lockfile-lint](https://github.com/lirantal/lockfile-lint)** — Validates your `package-lock.json` against policy rules (e.g., only allow packages from the official registry).
- **[is-website-vulnerable](https://github.com/lirantal/is-website-vulnerable)** — Checks a live website for known vulnerable JavaScript libraries.

---

## Monitoring & Alerting

- **[Socket Threat Intel](https://socket.dev/supply-chain-attacks)** — Real-time feed of detected supply chain attacks across open source ecosystems.
- **[OSV (Open Source Vulnerabilities)](https://osv.dev)** — Google's open database of vulnerabilities across open source packages. API-accessible.
- **[deps.dev](https://deps.dev)** — Google's package insights tool — dependency graphs, license info, and security advisories.
- **[npm security advisories](https://www.npmjs.com/advisories)** — Official npm advisory feed.
- **[PyPI malware reports](https://pypi.org/security/)** — Python package index security page.

---

## Hardening Your Pipeline

### GitHub Actions Best Practices

```yaml
# Pin actions to a full commit SHA — not a tag
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

# Limit token permissions to least privilege
permissions:
  contents: read

# Pin your runner OS version
runs-on: ubuntu-24.04  # not ubuntu-latest
```

### npm Best Practices

```bash
# Enforce lockfile usage in CI (never auto-update)
npm ci

# Audit before install in CI
npm audit --audit-level=high && npm ci

# Require 2FA for publish
npm profile enable-2fa auth-and-writes

# Use scoped tokens with publish restrictions
npm token create --cidr-whitelist=<YOUR_CI_IP>/32
```

### General Dependency Hygiene

- **Prefer SHA pinning over tag pinning** in GitHub Actions — tags are mutable
- **Review `preinstall`/`postinstall` scripts** before installing unfamiliar packages: `npm install --ignore-scripts` then review manually
- **Enable Dependabot or Renovate** for automated security updates — but review PRs before auto-merging
- **Keep a software bill of materials (SBOM)** — know what's in your build at all times

---

## Incident Response Quick Reference

If you suspect a compromised package:

```bash
# 1. Check what version is installed
npm ls <package-name>

# 2. View the package's install scripts before running
npm pack <package-name> && tar -tf <package-name>-*.tgz

# 3. Revoke and rotate npm tokens
npm token list
npm token revoke <token-id>

# 4. Check recent publishes on packages you own
npm view <your-package> time --json

# 5. Audit git history for unexpected CI/CD changes
git log --all --oneline -- .github/workflows/
```

---

## Learning & Background Reading

### Research & Blogs
- [Socket Research Blog](https://socket.dev/blog) — Original supply chain attack disclosures
- [Sonatype State of the Software Supply Chain](https://www.sonatype.com/state-of-the-software-supply-chain) — Annual report; excellent longitudinal data
- [JFrog Security Research](https://jfrog.com/blog/category/security-research/)
- [StepSecurity Blog](https://www.stepsecurity.io/blog)
- [Checkmarx Supply Chain Security](https://checkmarx.com/blog/)

### Standards & Frameworks
- **[SLSA (Supply-chain Levels for Software Artifacts)](https://slsa.dev)** — Google's framework for incrementally hardening your supply chain. Start at Level 1.
- **[SSDF (Secure Software Development Framework)](https://csrc.nist.gov/projects/ssdf)** — NIST's guidelines for secure software development practices.
- **[OpenSSF Scorecard](https://securityscorecards.dev)** — Automated checks for open source project security practices.
- **[CISA Software Supply Chain Security Guidance](https://www.cisa.gov/resources-tools/resources/software-supply-chain-security-guidance)**

### Threat Intelligence Feeds (Primary Sources Used in This Repo)
- [paloaltonetworks.com](https://www.paloaltonetworks.com/blog/category/threat-research/)
- [blog.checkpoint.com](https://blog.checkpoint.com)
- [blog.qualys.com](https://blog.qualys.com)
- [aikido.dev](https://www.aikido.dev/blog)
- [upwind.io](https://www.upwind.io/blog)
- [stepsecurity.io](https://www.stepsecurity.io/blog)
- [thehackernews.com](https://thehackernews.com)
- [securityboulevard.com](https://securityboulevard.com)
- [sonatype.com](https://www.sonatype.com/blog)
- [jfrog.com/blog](https://jfrog.com/blog/)
- [kodemsecurity.com](https://www.kodemsecurity.com)
- [betterstack.com](https://betterstack.com/community/security/)
