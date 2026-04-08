# Contributing a New Attack Entry

When a new supply chain incident is confirmed, add it to this repository using the standard template below. The goal is to make each entry immediately actionable — not just informational.

---

## File Naming Convention

```
attacks/YYYY-MM-short-name.md
```

Examples:
- `attacks/2026-03-trivy-github-actions.md`
- `attacks/2026-05-lodash-compromise.md`
- `attacks/2025-09-great-npm-heist.md`

Use the month the attack was **disclosed or detected**, not when it may have started silently.

---

## Standard Attack Template

Copy and fill in the template below for each new entry:

```markdown
# [Attack Name]

**Date:** [Month YYYY]
**Ecosystem:** [npm / PyPI / GitHub Actions / RubyGems / etc.]
**Severity:** [Critical / High / Medium]
**Type:** [e.g. Tag hijacking / Malicious package / Worm / Credential theft]
**Sources:** [linked source names]

---

## Summary

[2–4 sentence overview: what happened, who was affected, what was the goal.]

---

## Compromised Packages / Artifacts

| Package / Action | Malicious Version(s) |
|-----------------|---------------------|
| `package-name`  | x.y.z               |

---

## How It Worked

[Technical breakdown of the attack mechanism. Be specific — delivery, payload, execution, exfiltration.]

---

## Detection

[Exact commands or queries a developer can run RIGHT NOW to check if they're affected.]

---

## Remediation

[Numbered, specific steps to take if compromised.]

---

## Lessons Learned

[1–3 takeaways that generalize beyond this specific incident.]

---

## Related Incidents

[Links to other entries in this repo or external reports that share techniques or threat actors.]
```

---

## Quality Standards

**Do include:**
- Exact malicious version numbers (not ranges like "all versions before x")
- Copy-pasteable detection commands
- Specific file paths, env vars, or endpoints referenced in the payload
- Links to primary sources (security firm blogs, CVEs, GitHub issues, researcher disclosures)
- The attack mechanism at a technical level — not just "it stole credentials"

**Do not include:**
- Unconfirmed version numbers (mark uncertain information clearly)
- Attribution to specific threat actors unless officially confirmed by a credible source
- Speculation presented as fact

---

## Updating the README Timeline

After adding a new attack file, add a row to the timeline table in `README.md`:

```markdown
| [Month YYYY] | [Attack Name](./attacks/YYYY-MM-short-name.md) | [Ecosystem] | [One-line impact summary] |
```

Keep the table sorted with the most recent incidents at the top.

---

## Sources

Primary sources are preferred in this order:
1. Security firm original research (Socket, Sonatype, JFrog, Snyk, StepSecurity, etc.)
2. CVE database entries
3. Official maintainer postmortems
4. Reputable security journalism (The Hacker News, Bleeping Computer, etc.)

Always link to the original disclosure, not an article that summarizes another article.
