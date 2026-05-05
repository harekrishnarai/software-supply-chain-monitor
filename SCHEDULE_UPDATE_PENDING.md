# Schedule Update — Pending Application

**Status:** Ready to apply — open a new Cowork session and ask Claude to apply this update to the `supply-chain-daily-monitor` scheduled task.

---

## Updated task description

```
Daily RSS scan of Aikido, StepSecurity, Semgrep, Wiz & Unit 42 for new supply chain attacks — auto-updates knowledge base
```

## Two new feeds being added

| Feed | URL | Notes |
|------|-----|-------|
| Wiz Research | `https://www.wiz.io/feed/rss.xml` | 500+ item archive — filtered to last 3 days only |
| Unit 42 (Palo Alto) | `https://unit42.paloaltonetworks.com/feed/` | 15 recent items, has category tags |

---

## Full updated prompt

````
You are maintaining a supply chain attacks knowledge base. Your job is to check 5 RSS feeds for new attack articles, then create files for any new attacks found.

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

## STEP 2 — Fetch the 5 RSS feeds via Chrome

Use the Chrome browser tools. Navigate each feed URL in a separate tab and extract items.

### Feed 1 — StepSecurity
URL: `https://www.stepsecurity.io/blog/rss.xml`

Navigate to it, then run:
```javascript
window._rss = document.body.innerText;
'ok: ' + window._rss.length
```
Then extract items:
```javascript
const matches = [...window._rss.matchAll(/<title>([^<]+)<\/title>|<link>([^<]+)<\/link>|<pubDate>([^<]+)<\/pubDate>/g)];
const items = []; let cur = {};
for (const m of matches) {
  if (m[1]) { if (cur.title) { items.push(cur); cur = {}; } cur.title = m[1]; }
  if (m[2] && cur.title && !cur.link) cur.link = m[2];
  if (m[3]) cur.date = m[3];
}
if (cur.title) items.push(cur);
items.filter(i => i.link && i.link.includes('/blog/')).map(i => i.date?.substring(0,16) + ' | ' + i.title + '\n  ' + i.link).join('\n\n')
```

### Feed 2 — Aikido Security
URL: `https://www.aikido.dev/blog/rss.xml`

Same approach — navigate, store in `window._rss`, run the same extraction script.

### Feed 3 — Semgrep
URL: `https://semgrep.dev/blog/rss/`

Navigate, store in `window._rss`, then use this CDATA-aware extractor:
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

### Feed 4 — Wiz Research
URL: `https://www.wiz.io/feed/rss.xml`

This feed contains 500+ items (full archive). Navigate to it, then use the DOM XML parser and filter to the last 3 days only:
```javascript
const cutoff = new Date(Date.now() - 3*24*60*60*1000);
const items = document.querySelectorAll('item');
Array.from(items).filter(item => {
  const pub = item.querySelector('pubDate')?.textContent;
  return pub && new Date(pub) > cutoff;
}).map(item => {
  const title = item.querySelector('title')?.textContent?.trim();
  const link = item.querySelector('link')?.textContent?.trim() || item.querySelector('guid')?.textContent?.trim();
  const date = item.querySelector('pubDate')?.textContent?.substring(0,16).trim();
  return date + ' | ' + title + '\n  ' + link;
}).join('\n\n')
```

Note: Wiz has no category tags — keyword filtering in STEP 3 is the only relevance filter.

### Feed 5 — Unit 42 (Palo Alto Networks)
URL: `https://unit42.paloaltonetworks.com/feed/`

Note: use `/feed/` not the articles page URL. Navigate, store in `window._rss`, then use this CDATA-aware extractor that also captures categories:
```javascript
const raw = document.body.innerText;
const itemBlocks = [...raw.matchAll(/<item>([\s\S]*?)<\/item>/g)].map(m => m[1]);
itemBlocks.slice(0,20).map(block => {
  const title = (block.match(/<title><!\[CDATA\[([^\]]+)\]\]>/) || block.match(/<title>([^<]+)<\/title>/))?.[1]?.trim();
  const link = block.match(/<link>([^<]+)<\/link>/)?.[1]?.trim() || block.match(/<guid[^>]*>([^<]+)<\/guid>/)?.[1]?.trim();
  const date = block.match(/<pubDate>([^<]+)<\/pubDate>/)?.[1]?.substring(0,16).trim();
  const cats = [...block.matchAll(/<category><!\[CDATA\[([^\]]+)\]\]><\/category>/g)].map(m=>m[1].trim()).join(', ');
  return date + ' | ' + title + (cats ? ' ['+cats+']' : '') + '\n  ' + link;
}).filter(s => !s.includes('undefined')).join('\n\n')
```

---

## STEP 3 — Identify NEW attack articles

From all 5 feeds combined, filter for items that:
1. Are NOT already in the README timeline (compare by title keywords and date)
2. Are attack/incident reports — look for these keywords in the title or URL: `compromised`, `backdoor`, `malware`, `supply chain`, `attack`, `hijack`, `poisoning`, `worm`, `infostealer`, `CVE`, `exploit`, `typosquat`, `hackerbot`

**Per-feed relevance guidance:**
- **StepSecurity & Aikido** (primary): Any attack/incident post is likely relevant. Include broadly.
- **Semgrep** (supplementary): Include attack/incident reports; skip general guides, opinion pieces, product news.
- **Wiz** (supplementary): Include posts about supply chain attacks, GitHub Actions compromises, malicious packages, RCE in developer tooling, CVEs in build infrastructure. Skip: product recaps, state-of-cloud reports, architecture guides, partner announcements, cloud cost/compliance content.
- **Unit 42** (supplementary): Prioritize posts with categories `Malware`, `Threat Research`, or `High Profile Threats` that also contain supply chain keywords. Skip: `Insights`/opinion pieces, regional geopolitical briefs, AI-defence think-pieces not tied to a specific incident.

If the same attack appears on multiple blogs, use the most detailed article as the primary source and list others under Sources. Do NOT create duplicate files.

Skip: product announcements, feature releases, opinion posts.

If an article pubDate is OLDER than the most recent already-documented attack of the same type, still include it if it's genuinely new content.

---

## STEP 4 — Read each new article in full

For each new attack article, open it in a Chrome tab:
1. Navigate to the URL
2. Try `get_page_text` first
3. If that fails (too large), use this JS extractor to get content in chunks:
```javascript
const allEls = Array.from(document.querySelectorAll('h1,h2,h3,h4,p,li,code,pre,td'));
let inArticle = false;
const lines = [];
for (const el of allEls) {
  if (el.tagName === 'H1') inArticle = true;
  if (inArticle) {
    const text = el.textContent.replace(/\s+/g,' ').trim();
    if (text.length > 5 && text.length < 3000) lines.push(el.tagName+': '+text);
  }
  if (inArticle && (el.textContent.includes('Secure your code') || el.textContent.includes('Similar Posts') || el.textContent.includes('Related Posts'))) break;
}
window._art = lines.join('\n');
window._art.length + ' chars'
```
Then retrieve in 4000-char chunks:
```javascript
window._art.substring(0, 4000)    // chunk 1
window._art.substring(4000, 8000) // chunk 2
// etc until you have all content
```

Extract from each article:
- Attack name, date, ecosystem (npm/PyPI/GitHub Actions/VS Code/OpenVSX)
- Which packages/actions/extensions were compromised and which versions
- How the attack worked (payload mechanics, obfuscation, C2, persistence, propagation)
- Timeline of events with UTC timestamps if available
- IOCs: domains, IPs, file hashes, wallet addresses, malicious package versions
- Detection commands (bash/grep/npm ls/hash checks)
- Remediation steps

---

## STEP 5 — Create attack files

For each new attack, create a markdown file at:
`$REPO_DIR/attacks/YYYY-MM-<short-slug>.md`

Naming examples: `2026-03-bittensor-pypi.md`, `2026-03-hackerbot-claw.md`

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
- Semgrep RSS: https://semgrep.dev/blog/rss/ ([N] items)
- Wiz RSS: https://www.wiz.io/feed/rss.xml ([N] recent items, filtered from full archive)
- Unit 42 RSS: https://unit42.paloaltonetworks.com/feed/ ([N] items)
```

If nothing new: `No new attacks found. Knowledge base is up to date as of [date].`

---

## CONSTRAINTS

- Prefer Aikido and StepSecurity as primary sources. Use Semgrep, Wiz, and Unit 42 as supplementary.
- Do NOT overwrite existing files — only create new ones.
- Do NOT document product announcements, feature releases, or opinion posts.
- If the same attack appears on multiple blogs, use the most detailed article as the primary source and list others under Sources.
- Every attack file MUST include detection commands — do not create a file without them.
- Always read existing attack files in attacks/ as style reference before writing new ones.
- For Wiz: filter to items published within the last 3 days to avoid processing the full 500+ item archive.
- For Unit 42: use `https://unit42.paloaltonetworks.com/feed/` (the RSS feed, not the articles page).
````
