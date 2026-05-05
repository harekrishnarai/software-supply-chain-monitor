---
name: supply-chain-daily-monitor
description: Daily RSS scan of Aikido, StepSecurity, Semgrep, Wiz & Unit 42 for new supply chain attacks — auto-updates knowledge base
---

You are maintaining a supply chain attacks knowledge base. Your job is to check 5 feeds for new attack articles, then create files for any new attacks found.

---

## STEP 0 — Discover the workspace path

The workspace folder path changes every session. Discover it dynamically before doing anything else:

```bash
REPO_DIR=$(ls -d /sessions/*/mnt/Supply\ chain\ attacks 2>/dev/null | head -1)
echo "Repo dir: $REPO_DIR"
```

Use this path for ALL file reads, writes, and git operations below. If the directory is not found, abort and report the error.

---

## STEP 1 — Read what's already documented

Read this file to get the current attack list:
`$REPO_DIR/README.md`

Note all attack names and dates already in the timeline table. You will use this to skip duplicates.

---

## STEP 2 — Fetch the 5 feeds

For each feed, attempt to use `mcp__workspace__web_fetch` on the RSS URL first. If that returns `[binary data]`, fall back to fetching the blog's main HTML listing page and extracting article titles/URLs from the page text. As a last resort, use `WebSearch` with `site:<domain>` to find recent articles.

### Feed 1 — StepSecurity (PRIMARY)
RSS URL: `https://www.stepsecurity.io/blog/rss.xml`
HTML fallback: `https://www.stepsecurity.io/blog`

### Feed 2 — Aikido Security (PRIMARY)
RSS URL: `https://www.aikido.dev/blog/rss.xml`
HTML fallback: `https://www.aikido.dev/blog`

### Feed 3 — Wiz Research (PRIMARY)
RSS URL: `https://www.wiz.io/feed/rss.xml`
HTML fallback: `https://www.wiz.io/blog`

Wiz is a top-tier source — first reporter on some incidents (e.g. prt-scan AI GitHub Actions campaign, Shai-Hulud wave analyses). Their supply chain articles cover npm, PyPI, GitHub Actions, and CI/CD attacks in depth. Give Wiz equal weight to Aikido and StepSecurity.

### Feed 4 — Semgrep (SUPPLEMENTARY)
RSS URL: `https://semgrep.dev/blog/rss/`

Use this CDATA-aware extractor if fetching raw XML via browser:
```javascript
const raw = window._rss;
const itemBlocks = [...raw.matchAll(/<item>([\s\S]*?)<\/item>/g)].map(m => m[1]);
itemBlocks.slice(0,30).map(block => {
  const title = (block.match(/<title><!\[CDATA\[([^\]]+)\]\]>/) || block.match(/<title>([^<]+)<\/title>/))?.[1]?.trim();
  const link = block.match(/<link>\s*(https:\/\/semgrep\.dev\/blog[^\s<]+)\s*<\/link>/)?.[1]?.trim();
  const date = block.match(/<pubDate>([^<]+)<\/pubDate>/)?.[1]?.substring(0,16).trim();
  return date + ' | ' + title + '\n  ' + link;
}).filter(s => !s.includes('undefined')).join('\n\n')
```

### Feed 5 — Unit 42 / Palo Alto Networks (SUPPLEMENTARY — strict filtering required)
RSS URL: `https://unit42.paloaltonetworks.com/feed/`
HTML fallback: `https://unit42.paloaltonetworks.com/unit-42-all-articles/`

**Important:** Unit 42 publishes a wide range of threat intel (APT groups, ransomware, nation-state, AI security, phishing). The majority of their articles are NOT relevant to this repo. Apply strict filtering — only include articles that specifically cover supply chain attacks targeting package registries (npm, PyPI, Maven, RubyGems, etc.), GitHub Actions, VS Code/OpenVSX extensions, Docker Hub images, or CI/CD pipeline compromise. Discard everything else.

Unit 42 adds unique value through deep threat actor attribution (CVE assignments, actor codenames like TeamPCP), cross-ecosystem campaign analysis, and IOC sets that often complement Aikido/StepSecurity coverage.

---

## STEP 3 — Identify NEW attack articles

From all 5 feeds combined, filter for items that:
1. Are NOT already in the README timeline (compare by title keywords and date)
2. Are attack/incident reports — look for these keywords in the title or URL: `compromised`, `backdoor`, `malware`, `supply chain`, `attack`, `hijack`, `poisoning`, `worm`, `infostealer`, `CVE`, `exploit`, `typosquat`, `hackerbot`
3. For Unit 42 specifically: must also involve a package registry, GitHub Action, CI/CD pipeline, or IDE extension — not just generic malware/APT articles

Skip: product announcements, feature releases, opinion pieces, general guides, award news, APT profiles unrelated to package ecosystems.

If the article pubDate is OLDER than the most recent already-documented attack of the same type, still include it if it's genuinely new content not yet in the repo.

---

## STEP 4 — Read each new article in full

For each new attack article, fetch it via `mcp__workspace__web_fetch`. If the result exceeds token limits (saved to a file path), read the file in chunks using the Read tool with offset/limit parameters.

If web_fetch is blocked or returns no useful content, try `get_page_text` via Chrome browser tools.

Extract from each article:
- Attack name, date, ecosystem (npm/PyPI/GitHub Actions/VS Code/OpenVSX/Docker Hub/etc.)
- Which packages/actions/extensions were compromised and which versions
- How the attack worked (payload mechanics, obfuscation, C2, persistence, propagation)
- Timeline of events with UTC timestamps if available
- IOCs: domains, IPs, file hashes, wallet addresses, malicious package versions
- Detection commands (bash/grep/npm ls/hash checks)
- Remediation steps

If the same attack has been reported by multiple feeds, read the most detailed article as the primary source and note the others as additional Sources.

---

## STEP 5 — Create attack files

For each new attack, create a markdown file at:
`$REPO_DIR/attacks/YYYY-MM-<short-slug>.md`

Naming examples: `2026-03-bittensor-pypi.md`, `2026-04-prt-scan-github-actions.md`

Use this exact structure (match the style of existing files in the attacks/ directory):

```markdown
# [Attack Name]

**Date:** [Month YYYY]
**Ecosystem:** [npm / PyPI / GitHub Actions / VS Code / OpenVSX / etc.]
**Severity:** Critical / High
**Type:** [e.g. Tag poisoning / Backdoor / Phishing / Worm / Infostealer]
**Sources:**
- [Source blog — Article title](URL)

---

## Summary

[2-3 paragraphs: what happened, how it was discovered, why it matters]

---

## Compromised Artifacts

[Table: Package/Action/Extension | Malicious Version(s)]

---

## How It Worked

[Subsections covering: entry point, payload mechanics, C2/exfiltration, persistence, propagation if applicable]

---

## Timeline

[Table: Date/Time (UTC) | Event]

---

## Detection

\`\`\`bash
[Detection commands, hash checks, log searches]
\`\`\`

---

## Remediation

[Numbered steps]

---

## Lessons Learned

[Bullet points of systemic takeaways — what this attack teaches defenders]

---

## Related Incidents

[Links to related files in the attacks/ directory using relative paths: ./filename.md]
```

---

## STEP 6 — Update README.md

Edit `$REPO_DIR/README.md`:

1. Add each new attack as a row in the timeline table, keeping reverse-chronological order:
```
| [Month YYYY] | [Attack Name](./attacks/FILENAME.md) | [Ecosystem] | [One-line impact summary] |
```

2. Add the new filename to the Repository Structure code block.

---

## STEP 7 — Output a run summary

Print this to the session output:

```
## Supply Chain Monitor — [Today's Date]

### New attacks documented: N
- [Attack name] ([date]) — [one-line description]
  Source: [blog name]

### Already documented (skipped): N
- [Any articles that matched but were already in the repo]

### Sources checked:
- StepSecurity RSS: https://www.stepsecurity.io/blog/rss.xml ([N] items)
- Aikido RSS: https://www.aikido.dev/blog/rss.xml ([N] items)
- Wiz RSS: https://www.wiz.io/feed/rss.xml ([N] items)
- Semgrep RSS: https://semgrep.dev/blog/rss/ ([N] items)
- Unit 42 RSS: https://unit42.paloaltonetworks.com/feed/ ([N] items total, [M] passed supply-chain filter)
```

If nothing new: `No new attacks found. Knowledge base is up to date as of [date].`

---

## CONSTRAINTS

- **Primary sources** (equal weight): Aikido, StepSecurity, Wiz. These three focus on supply chain attacks and should be checked thoroughly.
- **Supplementary sources**: Semgrep (useful for TeamPCP/Trivy chain context) and Unit 42 (useful for threat actor attribution and CVE tagging). Apply stricter filtering to both.
- Do NOT overwrite existing files — only create new ones.
- Do NOT document product announcements, feature releases, or opinion posts.
- Do NOT document Unit 42 articles about APT groups, ransomware, or nation-state attacks unless they specifically involve a compromised package registry, GitHub Action, or CI/CD pipeline.
- If the same attack appears on multiple blogs, use the most detailed article as the primary source and list others under Sources.
- Every attack file MUST include detection commands — do not create a file without them.
- Always read existing attack files in attacks/ as style reference before writing new ones.
