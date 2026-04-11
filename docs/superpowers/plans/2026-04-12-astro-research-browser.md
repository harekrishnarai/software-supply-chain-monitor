# Astro Research Browser Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a rich, research-grade Astro static site in `site/` that turns the Markdown corpus into a searchable, faceted browser deployable to GitHub Pages.

**Architecture:** Create a self-contained Astro app that reads repository Markdown content at build time, normalizes incident metadata into a static index, and renders editorial, browsing, pattern, and incident-detail pages from that index. Keep the repository Markdown files as the source of truth and avoid any server-side runtime so the output works on GitHub Pages.

**Tech Stack:** Astro, TypeScript, Node.js, static JSON/content generation, client-side search/filter UI, GitHub Actions Pages deployment

---

## File Structure

### New Files

- `site/package.json` for Astro dependencies and scripts
- `site/astro.config.mjs` for static build and GitHub Pages base config
- `site/tsconfig.json` for TypeScript config
- `site/src/env.d.ts` for Astro typing
- `site/src/styles/global.css` for design tokens and site-wide styling
- `site/src/layouts/BaseLayout.astro` for global shell, metadata, nav, and theme loading
- `site/src/components/Hero.astro` for homepage hero section
- `site/src/components/MetricGrid.astro` for summary metrics
- `site/src/components/IncidentCard.astro` for reusable incident result cards
- `site/src/components/FilterPanel.astro` for incidents browser filters
- `site/src/components/SearchBox.astro` for query input
- `site/src/components/RelatedIncidents.astro` for detail-page related records
- `site/src/components/MetadataRail.astro` for detail-page structured metadata
- `site/src/components/PatternCard.astro` for pattern entry points
- `site/src/components/TOC.astro` for markdown table of contents rendering
- `site/src/lib/content/readRepoMarkdown.ts` for file system readers
- `site/src/lib/content/parseAttack.ts` for extracting structured attack metadata
- `site/src/lib/content/parsePage.ts` for `README.md` and `resources.md`
- `site/src/lib/content/normalize.ts` for record normalization and tag derivation
- `site/src/lib/content/index.ts` for content-loading entry points
- `site/src/lib/search/buildSearchIndex.ts` for static search index generation
- `site/src/lib/utils/date.ts` for date formatting helpers
- `site/src/lib/utils/urlState.ts` for query/filter URL sync helpers
- `site/src/pages/index.astro` for homepage
- `site/src/pages/incidents/index.astro` for incidents browser
- `site/src/pages/incidents/[slug].astro` for generated incident pages
- `site/src/pages/patterns/index.astro` for pattern overview
- `site/src/pages/patterns/[slug].astro` for generated pattern pages
- `site/src/pages/resources.astro` for promoted resources page
- `site/src/pages/methodology.astro` for repo curation and metadata methodology
- `site/src/pages/404.astro` for Pages-safe not found handling
- `site/public/favicon.svg` for a simple site icon
- `.github/workflows/deploy-site.yml` for GitHub Pages deployment
- `site/README.md` for local development instructions

### Existing Files To Modify

- `.gitignore` to ignore `.worktrees/`
- `README.md` only if needed for a link to the hosted site later

## Task 1: Safe Worktree Setup

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: Confirm `.worktrees/` is ignored**

Run: `git check-ignore -v .worktrees`
Expected: output showing `.gitignore` rule for `.worktrees/`

- [ ] **Step 2: Commit the ignore and planning docs baseline**

```bash
git add .gitignore docs/superpowers/specs/2026-04-12-astro-research-browser-design.md docs/superpowers/plans/2026-04-12-astro-research-browser.md
git commit -m "chore: add astro site design and planning docs"
```

- [ ] **Step 3: Create isolated feature worktree**

```bash
git worktree add .worktrees/feature-astro-research-browser -b feature/astro-research-browser
```

- [ ] **Step 4: Verify branch and clean baseline in the worktree**

Run: `git status --short --branch`
Expected: `## feature/astro-research-browser` with no pending changes

## Task 2: Scaffold Astro Site

**Files:**
- Create: `site/package.json`
- Create: `site/astro.config.mjs`
- Create: `site/tsconfig.json`
- Create: `site/src/env.d.ts`
- Create: `site/public/favicon.svg`
- Create: `site/README.md`

- [ ] **Step 1: Write the failing scaffold expectation**

Create `site/README.md` with:

```md
# Site Workspace

Run `npm install` and `npm run build` from this directory.

Expected routes:
- `/`
- `/incidents`
- `/patterns`
- `/resources`
- `/methodology`
```

- [ ] **Step 2: Run the missing-package check**

Run: `test -f site/package.json`
Expected: exit status non-zero because the Astro app has not been scaffolded yet

- [ ] **Step 3: Write the minimal Astro scaffold**

Create `site/package.json` with:

```json
{
  "name": "supply-chain-attacks-site",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview"
  },
  "dependencies": {
    "astro": "^5.7.0",
    "gray-matter": "^4.0.3",
    "marked": "^15.0.7",
    "reading-time": "^1.5.0"
  }
}
```

Create `site/astro.config.mjs` with:

```js
import { defineConfig } from "astro/config";

const repo = "software-supply-chain-monitor";
const isGithubPages = process.env.GITHUB_ACTIONS === "true";

export default defineConfig({
  output: "static",
  site: "https://harekrishnarai.github.io/" + repo,
  base: isGithubPages ? `/${repo}` : "/"
});
```

Create `site/tsconfig.json` with:

```json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "baseUrl": "."
  }
}
```

Create `site/src/env.d.ts` with:

```ts
/// <reference types="astro/client" />
```

Create `site/public/favicon.svg` with:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="12" fill="#101726"/>
  <path d="M16 20h32v8H16zm0 16h24v8H16z" fill="#7dd3fc"/>
  <circle cx="46" cy="40" r="6" fill="#f59e0b"/>
</svg>
```

- [ ] **Step 4: Install dependencies and verify Astro is available**

Run: `cd site && npm install`
Expected: install completes and creates `package-lock.json`

- [ ] **Step 5: Commit**

```bash
git add site/package.json site/package-lock.json site/astro.config.mjs site/tsconfig.json site/src/env.d.ts site/public/favicon.svg site/README.md
git commit -m "chore: scaffold astro site workspace"
```

## Task 3: Build Content Ingestion Layer

**Files:**
- Create: `site/src/lib/content/readRepoMarkdown.ts`
- Create: `site/src/lib/content/parseAttack.ts`
- Create: `site/src/lib/content/parsePage.ts`
- Create: `site/src/lib/content/normalize.ts`
- Create: `site/src/lib/content/index.ts`
- Test: `site/src/pages/incidents/index.astro`

- [ ] **Step 1: Write the failing ingestion expectation**

Create the initial incidents page that imports `getAllIncidents()` before that function exists:

```astro
---
import { getAllIncidents } from "../../lib/content";
const incidents = await getAllIncidents();
---
<pre>{JSON.stringify(incidents.slice(0, 1), null, 2)}</pre>
```

- [ ] **Step 2: Run build to verify ingestion is missing**

Run: `cd site && npm run build`
Expected: FAIL with module resolution or missing export errors for `../../lib/content`

- [ ] **Step 3: Write minimal content readers**

Create `site/src/lib/content/readRepoMarkdown.ts` with:

```ts
import fs from "node:fs/promises";
import path from "node:path";

const repoRoot = path.resolve(process.cwd(), "..");
const attacksDir = path.join(repoRoot, "attacks");

export async function readAttackFiles() {
  const names = (await fs.readdir(attacksDir)).filter((name) => name.endsWith(".md"));
  return Promise.all(
    names.map(async (name) => ({
      name,
      filePath: path.join(attacksDir, name),
      raw: await fs.readFile(path.join(attacksDir, name), "utf8")
    }))
  );
}

export async function readRepoPage(name: "README.md" | "resources.md") {
  const filePath = path.join(repoRoot, name);
  return {
    filePath,
    raw: await fs.readFile(filePath, "utf8")
  };
}
```

Create `site/src/lib/content/parseAttack.ts` with:

```ts
type ParsedAttack = {
  slug: string;
  title: string;
  summary: string;
  dateLabel: string;
  year: number;
  month: string;
  ecosystems: string[];
  severity: string | null;
  attackTypes: string[];
  impactTags: string[];
  sourceLinks: { label: string; href: string }[];
  body: string;
  filePath: string;
};

function firstMatch(raw: string, pattern: RegExp) {
  return raw.match(pattern)?.[1]?.trim() ?? "";
}

function parseList(value: string) {
  return value
    .split("/")
    .map((item) => item.trim())
    .filter(Boolean);
}

export function parseAttack(name: string, raw: string, filePath: string): ParsedAttack {
  const title = firstMatch(raw, /^#\s+(.+)$/m);
  const dateLabel = firstMatch(raw, /^\*\*Date:\*\*\s+(.+)$/m);
  const severity = firstMatch(raw, /^\*\*Severity:\*\*\s+(.+)$/m) || null;
  const ecosystems = parseList(firstMatch(raw, /^\*\*Ecosystem:\*\*\s+(.+)$/m));
  const typeValue = firstMatch(raw, /^\*\*Type:\*\*\s+(.+)$/m);
  const attackTypes = typeValue ? typeValue.split("/").map((item) => item.trim()).filter(Boolean) : [];
  const summary = firstMatch(raw, /## Summary\s+([\s\S]*?)(?:\n## |\n---|$)/m).replace(/\n+/g, " ").trim();
  const year = Number((dateLabel.match(/\b(20\d{2})\b/) || [])[1] || name.slice(0, 4));
  const month = dateLabel.split(" ")[0] || "Unknown";
  const sourceLinks = Array.from(raw.matchAll(/\[([^\]]+)\]\((https?:\/\/[^)]+)\)/g)).map((match) => ({
    label: match[1],
    href: match[2]
  }));
  const impactTags = [...new Set([...attackTypes, ...ecosystems].map((item) => item.toLowerCase()))];

  return {
    slug: name.replace(/\.md$/, ""),
    title,
    summary,
    dateLabel,
    year,
    month,
    ecosystems,
    severity,
    attackTypes,
    impactTags,
    sourceLinks,
    body: raw,
    filePath
  };
}
```

Create `site/src/lib/content/parsePage.ts` with:

```ts
export function parseRepoPage(raw: string) {
  const title = raw.match(/^#\s+(.+)$/m)?.[1]?.trim() ?? "";
  return { title, body: raw };
}
```

Create `site/src/lib/content/normalize.ts` with:

```ts
import { marked } from "marked";

export async function renderMarkdown(markdown: string) {
  return marked.parse(markdown) as Promise<string>;
}

export function slugify(value: string) {
  return value
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-|-$/g, "");
}
```

Create `site/src/lib/content/index.ts` with:

```ts
import { readAttackFiles, readRepoPage } from "./readRepoMarkdown";
import { parseAttack } from "./parseAttack";
import { parseRepoPage } from "./parsePage";

export async function getAllIncidents() {
  const files = await readAttackFiles();
  return files
    .map((file) => parseAttack(file.name, file.raw, file.filePath))
    .sort((a, b) => (a.dateLabel < b.dateLabel ? 1 : -1));
}

export async function getIncidentBySlug(slug: string) {
  const incidents = await getAllIncidents();
  return incidents.find((incident) => incident.slug === slug) ?? null;
}

export async function getRepoPage(name: "README.md" | "resources.md") {
  const page = await readRepoPage(name);
  return parseRepoPage(page.raw);
}
```

- [ ] **Step 4: Run build to verify ingestion compiles**

Run: `cd site && npm run build`
Expected: build gets past content imports and fails later because layouts/pages are not complete yet

- [ ] **Step 5: Commit**

```bash
git add site/src/lib/content site/src/pages/incidents/index.astro
git commit -m "feat: add markdown ingestion pipeline"
```

## Task 4: Create Site Shell and Visual System

**Files:**
- Create: `site/src/styles/global.css`
- Create: `site/src/layouts/BaseLayout.astro`

- [ ] **Step 1: Write the failing layout import**

Update `site/src/pages/incidents/index.astro` to import `BaseLayout` before it exists:

```astro
---
import BaseLayout from "../../layouts/BaseLayout.astro";
import { getAllIncidents } from "../../lib/content";
const incidents = await getAllIncidents();
---
<BaseLayout title="Incidents">
  <pre>{JSON.stringify(incidents.slice(0, 1), null, 2)}</pre>
</BaseLayout>
```

- [ ] **Step 2: Run build to verify missing layout**

Run: `cd site && npm run build`
Expected: FAIL with missing layout import

- [ ] **Step 3: Write the layout and global design system**

Create `site/src/styles/global.css` with:

```css
:root {
  --bg: #07111f;
  --bg-soft: #0f1a2d;
  --panel: rgba(14, 23, 38, 0.72);
  --panel-strong: #111f35;
  --text: #e7edf7;
  --muted: #8fa4bf;
  --line: rgba(143, 164, 191, 0.18);
  --accent: #7dd3fc;
  --accent-2: #f59e0b;
  --danger: #fb7185;
  --shadow: 0 20px 70px rgba(0, 0, 0, 0.35);
  --radius: 24px;
  --max: 1240px;
}

* {
  box-sizing: border-box;
}

html {
  color-scheme: dark;
}

body {
  margin: 0;
  font-family: "Space Grotesk", "Segoe UI", sans-serif;
  color: var(--text);
  background:
    radial-gradient(circle at top, rgba(125, 211, 252, 0.14), transparent 30%),
    radial-gradient(circle at right, rgba(245, 158, 11, 0.12), transparent 25%),
    linear-gradient(180deg, #07111f 0%, #091423 100%);
  min-height: 100vh;
}

a {
  color: inherit;
  text-decoration: none;
}

.shell {
  width: min(calc(100% - 32px), var(--max));
  margin: 0 auto;
}

.panel {
  background: var(--panel);
  border: 1px solid var(--line);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  backdrop-filter: blur(12px);
}

.eyebrow {
  text-transform: uppercase;
  letter-spacing: 0.16em;
  color: var(--accent);
  font-size: 0.74rem;
}
```

Create `site/src/layouts/BaseLayout.astro` with:

```astro
---
import "../styles/global.css";

interface Props {
  title: string;
  description?: string;
}

const { title, description = "Research-grade browser for software supply chain attacks." } = Astro.props;
const pageTitle = `${title} | Supply Chain Attack Monitor`;
---
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>{pageTitle}</title>
    <meta name="description" content={description} />
    <link rel="icon" href={`${Astro.site?.pathname ?? "/"}favicon.svg`} />
  </head>
  <body>
    <header class="shell" style="padding: 24px 0 12px;">
      <nav class="panel" style="display:flex;align-items:center;justify-content:space-between;padding:16px 20px;">
        <a href={Astro.url.pathname.startsWith("/software-supply-chain-monitor") ? `${Astro.site?.pathname ?? "/"}`
          : "/"} style="font-weight:700;">Supply Chain Attack Monitor</a>
        <div style="display:flex;gap:16px;color:var(--muted);">
          <a href={`${Astro.site?.pathname ?? "/"}incidents`}>Incidents</a>
          <a href={`${Astro.site?.pathname ?? "/"}patterns`}>Patterns</a>
          <a href={`${Astro.site?.pathname ?? "/"}resources`}>Resources</a>
          <a href={`${Astro.site?.pathname ?? "/"}methodology`}>Methodology</a>
        </div>
      </nav>
    </header>
    <main>
      <slot />
    </main>
  </body>
</html>
```

- [ ] **Step 4: Run build to verify shell compiles**

Run: `cd site && npm run build`
Expected: build gets past shell setup and fails later on missing homepage/detail routes

- [ ] **Step 5: Commit**

```bash
git add site/src/styles/global.css site/src/layouts/BaseLayout.astro site/src/pages/incidents/index.astro
git commit -m "feat: add astro site shell and design tokens"
```

## Task 5: Build Homepage and Core Components

**Files:**
- Create: `site/src/components/Hero.astro`
- Create: `site/src/components/MetricGrid.astro`
- Create: `site/src/components/PatternCard.astro`
- Create: `site/src/components/IncidentCard.astro`
- Create: `site/src/pages/index.astro`

- [ ] **Step 1: Write the failing homepage import**

Create `site/src/pages/index.astro` with:

```astro
---
import BaseLayout from "../layouts/BaseLayout.astro";
import Hero from "../components/Hero.astro";
---
<BaseLayout title="Home">
  <Hero />
</BaseLayout>
```

- [ ] **Step 2: Run build to verify homepage component is missing**

Run: `cd site && npm run build`
Expected: FAIL with missing `Hero.astro`

- [ ] **Step 3: Write minimal homepage components**

Create `site/src/components/Hero.astro` with:

```astro
<section class="shell" style="padding: 40px 0 24px;">
  <div class="panel" style="padding:40px;display:grid;gap:18px;">
    <p class="eyebrow">Threat Research Corpus</p>
    <h1 style="font-size:clamp(2.7rem,7vw,5.5rem);margin:0;line-height:0.95;">Research-grade browsing for software supply chain attacks.</h1>
    <p style="max-width:60ch;color:var(--muted);font-size:1.1rem;">
      Explore real-world incidents across npm, PyPI, GitHub Actions, IDE ecosystems, and adjacent package supply chains through a structured static browser built from this repository.
    </p>
    <div style="display:flex;gap:12px;flex-wrap:wrap;">
      <a class="panel" style="padding:14px 18px;background:var(--accent);color:#07111f;font-weight:700;" href="/incidents">Browse incidents</a>
      <a class="panel" style="padding:14px 18px;" href="/patterns">Explore patterns</a>
    </div>
  </div>
</section>
```

Create `site/src/components/MetricGrid.astro` with:

```astro
---
interface Metric {
  label: string;
  value: string;
}

const { metrics } = Astro.props as { metrics: Metric[] };
---
<section class="shell" style="padding: 8px 0 24px;">
  <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(180px,1fr));gap:16px;">
    {metrics.map((metric) => (
      <article class="panel" style="padding:20px;">
        <div style="font-size:2rem;font-weight:700;">{metric.value}</div>
        <div style="color:var(--muted);">{metric.label}</div>
      </article>
    ))}
  </div>
</section>
```

Create `site/src/components/PatternCard.astro` with:

```astro
---
const { title, description, href } = Astro.props as { title: string; description: string; href: string };
---
<a class="panel" href={href} style="padding:20px;display:grid;gap:8px;">
  <strong>{title}</strong>
  <span style="color:var(--muted);">{description}</span>
</a>
```

Create `site/src/components/IncidentCard.astro` with:

```astro
---
const { incident } = Astro.props as {
  incident: {
    slug: string;
    title: string;
    dateLabel: string;
    summary: string;
    ecosystems: string[];
    severity: string | null;
  };
};
---
<a class="panel" href={`/incidents/${incident.slug}`} style="padding:20px;display:grid;gap:12px;">
  <div style="display:flex;justify-content:space-between;gap:12px;align-items:start;">
    <div>
      <div class="eyebrow">{incident.dateLabel}</div>
      <h3 style="margin:8px 0 0;">{incident.title}</h3>
    </div>
    {incident.severity && <span style="color:var(--accent-2);font-weight:700;">{incident.severity}</span>}
  </div>
  <p style="margin:0;color:var(--muted);">{incident.summary}</p>
  <div style="display:flex;gap:8px;flex-wrap:wrap;">
    {incident.ecosystems.map((ecosystem) => <span class="panel" style="padding:6px 10px;border-radius:999px;">{ecosystem}</span>)}
  </div>
</a>
```

Update `site/src/pages/index.astro` with:

```astro
---
import BaseLayout from "../layouts/BaseLayout.astro";
import Hero from "../components/Hero.astro";
import MetricGrid from "../components/MetricGrid.astro";
import IncidentCard from "../components/IncidentCard.astro";
import PatternCard from "../components/PatternCard.astro";
import { getAllIncidents } from "../lib/content";

const incidents = await getAllIncidents();
const metrics = [
  { label: "Tracked incidents", value: String(incidents.length) },
  { label: "Years covered", value: String(new Set(incidents.map((item) => item.year)).size) },
  { label: "Ecosystems", value: String(new Set(incidents.flatMap((item) => item.ecosystems)).size) }
];

const featured = incidents.slice(0, 3);
const patterns = [
  { title: "Maintainer compromise", description: "Account takeover and poisoned publishes.", href: "/patterns/maintainer-compromise" },
  { title: "Tag poisoning", description: "Mutable version references used for runner compromise.", href: "/patterns/tag-poisoning" },
  { title: "Typosquatting", description: "Near-match package names hiding malware.", href: "/patterns/typosquatting" }
];
---
<BaseLayout title="Home">
  <Hero />
  <MetricGrid metrics={metrics} />
  <section class="shell" style="padding: 8px 0 24px;">
    <div style="display:grid;gap:16px;">
      <p class="eyebrow">Featured Incidents</p>
      <div style="display:grid;gap:16px;">
        {featured.map((incident) => <IncidentCard incident={incident} />)}
      </div>
    </div>
  </section>
  <section class="shell" style="padding: 8px 0 64px;">
    <div style="display:grid;gap:16px;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));">
      {patterns.map((pattern) => <PatternCard {...pattern} />)}
    </div>
  </section>
</BaseLayout>
```

- [ ] **Step 4: Run build to verify homepage renders**

Run: `cd site && npm run build`
Expected: homepage builds, with remaining failures only from unimplemented detail and pattern routes

- [ ] **Step 5: Commit**

```bash
git add site/src/components/Hero.astro site/src/components/MetricGrid.astro site/src/components/PatternCard.astro site/src/components/IncidentCard.astro site/src/pages/index.astro
git commit -m "feat: add editorial astro homepage"
```

## Task 6: Build Incidents Browser with Search and Filters

**Files:**
- Create: `site/src/components/SearchBox.astro`
- Create: `site/src/components/FilterPanel.astro`
- Modify: `site/src/pages/incidents/index.astro`
- Create: `site/src/lib/search/buildSearchIndex.ts`
- Create: `site/src/lib/utils/urlState.ts`

- [ ] **Step 1: Write the failing filter component import**

Update `site/src/pages/incidents/index.astro` to import `SearchBox` and `FilterPanel` before they exist:

```astro
---
import BaseLayout from "../../layouts/BaseLayout.astro";
import SearchBox from "../../components/SearchBox.astro";
import FilterPanel from "../../components/FilterPanel.astro";
import IncidentCard from "../../components/IncidentCard.astro";
import { getAllIncidents } from "../../lib/content";
const incidents = await getAllIncidents();
---
<BaseLayout title="Incidents">
  <section class="shell" style="padding:32px 0 64px;">
    <SearchBox />
    <FilterPanel />
    <div style="display:grid;gap:16px;">
      {incidents.map((incident) => <IncidentCard incident={incident} />)}
    </div>
  </section>
</BaseLayout>
```

- [ ] **Step 2: Run build to verify missing browser components**

Run: `cd site && npm run build`
Expected: FAIL with missing incidents browser component imports

- [ ] **Step 3: Write the incidents browser**

Create `site/src/components/SearchBox.astro` with:

```astro
<label class="panel" style="display:block;padding:14px 16px;margin-bottom:16px;">
  <span class="eyebrow">Search</span>
  <input id="incident-search" type="search" placeholder="Search incidents, ecosystems, attack types..." style="margin-top:8px;width:100%;background:transparent;border:none;color:var(--text);font:inherit;outline:none;" />
</label>
```

Create `site/src/components/FilterPanel.astro` with:

```astro
---
const { years = [], ecosystems = [], severities = [] } = Astro.props as {
  years?: number[];
  ecosystems?: string[];
  severities?: string[];
};
---
<div class="panel" style="padding:16px;margin-bottom:16px;display:grid;gap:12px;">
  <div class="eyebrow">Filters</div>
  <div style="display:flex;flex-wrap:wrap;gap:8px;">
    {years.map((year) => <button type="button" data-filter-group="year" data-filter-value={String(year)} class="panel" style="padding:8px 12px;cursor:pointer;">{year}</button>)}
    {ecosystems.map((ecosystem) => <button type="button" data-filter-group="ecosystem" data-filter-value={ecosystem} class="panel" style="padding:8px 12px;cursor:pointer;">{ecosystem}</button>)}
    {severities.map((severity) => <button type="button" data-filter-group="severity" data-filter-value={severity} class="panel" style="padding:8px 12px;cursor:pointer;">{severity}</button>)}
  </div>
</div>
```

Create `site/src/lib/search/buildSearchIndex.ts` with:

```ts
export function buildSearchText(incident: {
  title: string;
  summary: string;
  ecosystems: string[];
  attackTypes: string[];
  impactTags: string[];
}) {
  return [
    incident.title,
    incident.summary,
    incident.ecosystems.join(" "),
    incident.attackTypes.join(" "),
    incident.impactTags.join(" ")
  ]
    .join(" ")
    .toLowerCase();
}
```

Create `site/src/lib/utils/urlState.ts` with:

```ts
export const FILTER_PARAM = "filters";
export const QUERY_PARAM = "q";
```

Update `site/src/pages/incidents/index.astro` with:

```astro
---
import BaseLayout from "../../layouts/BaseLayout.astro";
import SearchBox from "../../components/SearchBox.astro";
import FilterPanel from "../../components/FilterPanel.astro";
import IncidentCard from "../../components/IncidentCard.astro";
import { buildSearchText } from "../../lib/search/buildSearchIndex";
import { getAllIncidents } from "../../lib/content";

const incidents = await getAllIncidents();
const years = [...new Set(incidents.map((incident) => incident.year))].sort((a, b) => b - a);
const ecosystems = [...new Set(incidents.flatMap((incident) => incident.ecosystems))].sort();
const severities = [...new Set(incidents.map((incident) => incident.severity).filter(Boolean))] as string[];
const browserIncidents = incidents.map((incident) => ({ ...incident, searchText: buildSearchText(incident) }));
---
<BaseLayout title="Incidents">
  <section class="shell" style="padding:32px 0 64px;">
    <div style="display:grid;grid-template-columns:minmax(0,280px) minmax(0,1fr);gap:20px;align-items:start;">
      <aside>
        <SearchBox />
        <FilterPanel years={years} ecosystems={ecosystems} severities={severities} />
      </aside>
      <div id="incident-results" style="display:grid;gap:16px;">
        {browserIncidents.map((incident) => (
          <div
            data-incident-card
            data-title={incident.title}
            data-search-text={incident.searchText}
            data-year={incident.year}
            data-ecosystems={incident.ecosystems.join("|")}
            data-severity={incident.severity ?? ""}
          >
            <IncidentCard incident={incident} />
          </div>
        ))}
      </div>
    </div>
  </section>
  <script>
    const search = document.querySelector("#incident-search");
    const buttons = Array.from(document.querySelectorAll("[data-filter-group]"));
    const cards = Array.from(document.querySelectorAll("[data-incident-card]"));
    const active = new Map();
    function applyFilters() {
      const query = String(search?.value || "").toLowerCase().trim();
      const params = new URLSearchParams(window.location.search);
      params.set("q", query);
      params.delete("filters");
      for (const [group, values] of active.entries()) {
        for (const value of values) params.append("filters", `${group}:${value}`);
      }
      history.replaceState({}, "", `${window.location.pathname}?${params.toString()}`);
      for (const card of cards) {
        const searchText = card.getAttribute("data-search-text") || "";
        const matchesQuery = !query || searchText.includes(query);
        const matchesYear = !active.get("year")?.size || active.get("year").has(card.getAttribute("data-year"));
        const matchesSeverity = !active.get("severity")?.size || active.get("severity").has(card.getAttribute("data-severity"));
        const ecosystems = new Set((card.getAttribute("data-ecosystems") || "").split("|").filter(Boolean));
        const matchesEcosystem = !active.get("ecosystem")?.size || [...active.get("ecosystem")].some((value) => ecosystems.has(value));
        card.style.display = matchesQuery && matchesYear && matchesSeverity && matchesEcosystem ? "" : "none";
      }
    }
    for (const button of buttons) {
      button.addEventListener("click", () => {
        const group = button.getAttribute("data-filter-group");
        const value = button.getAttribute("data-filter-value");
        if (!group || !value) return;
        const values = active.get(group) || new Set();
        values.has(value) ? values.delete(value) : values.add(value);
        active.set(group, values);
        button.style.borderColor = values.has(value) ? "var(--accent)" : "var(--line)";
        applyFilters();
      });
    }
    search?.addEventListener("input", applyFilters);
    applyFilters();
  </script>
</BaseLayout>
```

- [ ] **Step 4: Run build to verify incidents browser works**

Run: `cd site && npm run build`
Expected: incidents browser page builds, with remaining failures only for unimplemented detail/resources/pattern pages

- [ ] **Step 5: Commit**

```bash
git add site/src/components/SearchBox.astro site/src/components/FilterPanel.astro site/src/lib/search/buildSearchIndex.ts site/src/lib/utils/urlState.ts site/src/pages/incidents/index.astro
git commit -m "feat: add searchable incidents browser"
```

## Task 7: Generate Incident Detail Pages

**Files:**
- Create: `site/src/components/MetadataRail.astro`
- Create: `site/src/components/RelatedIncidents.astro`
- Create: `site/src/components/TOC.astro`
- Modify: `site/src/pages/incidents/[slug].astro`

- [ ] **Step 1: Write the failing dynamic route expectation**

Create `site/src/pages/incidents/[slug].astro` with:

```astro
---
import { getStaticPaths } from "astro";
---
```

- [ ] **Step 2: Run build to verify detail route is incomplete**

Run: `cd site && npm run build`
Expected: FAIL because `getStaticPaths` is not a valid import or route implementation is incomplete

- [ ] **Step 3: Write detail route and supporting components**

Create `site/src/components/MetadataRail.astro` with:

```astro
---
const { incident } = Astro.props as { incident: any };
---
<aside class="panel" style="padding:20px;display:grid;gap:12px;position:sticky;top:24px;">
  <div>
    <div class="eyebrow">Date</div>
    <div>{incident.dateLabel}</div>
  </div>
  <div>
    <div class="eyebrow">Ecosystems</div>
    <div style="display:flex;flex-wrap:wrap;gap:8px;">{incident.ecosystems.map((item: string) => <span class="panel" style="padding:6px 10px;border-radius:999px;">{item}</span>)}</div>
  </div>
  {incident.severity && (
    <div>
      <div class="eyebrow">Severity</div>
      <div>{incident.severity}</div>
    </div>
  )}
</aside>
```

Create `site/src/components/RelatedIncidents.astro` with:

```astro
---
import IncidentCard from "./IncidentCard.astro";
const { incidents } = Astro.props as { incidents: any[] };
---
{incidents.length > 0 && (
  <section style="display:grid;gap:16px;">
    <p class="eyebrow">Related Incidents</p>
    <div style="display:grid;gap:16px;">
      {incidents.map((incident) => <IncidentCard incident={incident} />)}
    </div>
  </section>
)}
```

Create `site/src/components/TOC.astro` with:

```astro
---
const { headings } = Astro.props as { headings: { slug: string; text: string; depth: number }[] };
---
{headings.length > 0 && (
  <nav class="panel" style="padding:20px;display:grid;gap:10px;">
    <div class="eyebrow">On This Page</div>
    {headings.map((heading) => <a href={`#${heading.slug}`} style={`padding-left:${(heading.depth - 2) * 12}px;color:var(--muted);`}>{heading.text}</a>)}
  </nav>
)}
```

Update `site/src/pages/incidents/[slug].astro` with:

```astro
---
import BaseLayout from "../../layouts/BaseLayout.astro";
import MetadataRail from "../../components/MetadataRail.astro";
import RelatedIncidents from "../../components/RelatedIncidents.astro";
import TOC from "../../components/TOC.astro";
import { getAllIncidents, getIncidentBySlug } from "../../lib/content";
import { renderMarkdown, slugify } from "../../lib/content/normalize";

export async function getStaticPaths() {
  const incidents = await getAllIncidents();
  return incidents.map((incident) => ({
    params: { slug: incident.slug }
  }));
}

const { slug } = Astro.params;
const incident = slug ? await getIncidentBySlug(slug) : null;
if (!incident) throw new Error(`Incident not found: ${slug}`);

const html = await renderMarkdown(incident.body);
const headings = Array.from(incident.body.matchAll(/^##+\s+(.+)$/gm)).map((match) => ({
  text: match[1],
  depth: match[0].startsWith("###") ? 3 : 2,
  slug: slugify(match[1])
}));
const related = (await getAllIncidents())
  .filter((item) => item.slug !== incident.slug)
  .filter((item) => item.ecosystems.some((ecosystem) => incident.ecosystems.includes(ecosystem)))
  .slice(0, 3);
---
<BaseLayout title={incident.title} description={incident.summary}>
  <section class="shell" style="padding:24px 0 64px;display:grid;gap:24px;">
    <div style="display:grid;gap:12px;">
      <p class="eyebrow">{incident.dateLabel}</p>
      <h1 style="margin:0;font-size:clamp(2.1rem,5vw,4rem);">{incident.title}</h1>
      <p style="margin:0;max-width:70ch;color:var(--muted);font-size:1.1rem;">{incident.summary}</p>
    </div>
    <div style="display:grid;grid-template-columns:minmax(0,220px) minmax(0,1fr) minmax(0,280px);gap:20px;align-items:start;">
      <TOC headings={headings} />
      <article class="panel" style="padding:28px;" set:html={html} />
      <MetadataRail incident={incident} />
    </div>
    <RelatedIncidents incidents={related} />
  </section>
</BaseLayout>
```

- [ ] **Step 4: Run build to verify incident pages generate**

Run: `cd site && npm run build`
Expected: dynamic incident pages build successfully, with remaining failures only from missing pattern/resources/methodology pages

- [ ] **Step 5: Commit**

```bash
git add site/src/components/MetadataRail.astro site/src/components/RelatedIncidents.astro site/src/components/TOC.astro site/src/pages/incidents/[slug].astro
git commit -m "feat: add generated incident detail pages"
```

## Task 8: Add Pattern, Resources, and Methodology Pages

**Files:**
- Create: `site/src/pages/patterns/index.astro`
- Create: `site/src/pages/patterns/[slug].astro`
- Create: `site/src/pages/resources.astro`
- Create: `site/src/pages/methodology.astro`

- [ ] **Step 1: Write the failing auxiliary pages expectation**

Create `site/src/pages/patterns/index.astro` with:

```astro
---
import BaseLayout from "../../layouts/BaseLayout.astro";
---
<BaseLayout title="Patterns">
  <MissingPatterns />
</BaseLayout>
```

- [ ] **Step 2: Run build to verify missing page implementation**

Run: `cd site && npm run build`
Expected: FAIL with `MissingPatterns` undefined

- [ ] **Step 3: Write pattern, resources, and methodology pages**

Update `site/src/pages/patterns/index.astro` with:

```astro
---
import BaseLayout from "../../layouts/BaseLayout.astro";
import PatternCard from "../../components/PatternCard.astro";
const patterns = [
  { slug: "maintainer-compromise", title: "Maintainer compromise", description: "Credential theft or social engineering leads to malicious publishes." },
  { slug: "tag-poisoning", title: "Tag poisoning", description: "Mutable release tags abused to compromise CI pipelines." },
  { slug: "typosquatting", title: "Typosquatting", description: "Look-alike package names bait installs and imports." }
];
---
<BaseLayout title="Patterns">
  <section class="shell" style="padding:32px 0 64px;display:grid;gap:16px;">
    <p class="eyebrow">Recurring Tactics</p>
    <div style="display:grid;gap:16px;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));">
      {patterns.map((pattern) => <PatternCard title={pattern.title} description={pattern.description} href={`/patterns/${pattern.slug}`} />)}
    </div>
  </section>
</BaseLayout>
```

Create `site/src/pages/patterns/[slug].astro` with:

```astro
---
import BaseLayout from "../../layouts/BaseLayout.astro";
import IncidentCard from "../../components/IncidentCard.astro";
import { getAllIncidents } from "../../lib/content";

const patternMap = {
  "maintainer-compromise": ["hijacked", "maintainer", "account takeover"],
  "tag-poisoning": ["tag poisoning", "github actions", "poisoned version tags"],
  typosquatting: ["typosquat", "typosquatting"]
} as const;

export async function getStaticPaths() {
  return Object.keys(patternMap).map((slug) => ({ params: { slug } }));
}

const { slug } = Astro.params;
const needles = patternMap[slug as keyof typeof patternMap] ?? [];
const incidents = (await getAllIncidents()).filter((incident) => {
  const haystack = `${incident.title} ${incident.summary} ${incident.attackTypes.join(" ")}`.toLowerCase();
  return needles.some((needle) => haystack.includes(needle));
});
---
<BaseLayout title={`Pattern: ${slug}`}>
  <section class="shell" style="padding:32px 0 64px;display:grid;gap:16px;">
    <p class="eyebrow">Pattern View</p>
    <h1 style="margin:0;text-transform:capitalize;">{String(slug).replace(/-/g, " ")}</h1>
    <div style="display:grid;gap:16px;">
      {incidents.map((incident) => <IncidentCard incident={incident} />)}
    </div>
  </section>
</BaseLayout>
```

Create `site/src/pages/resources.astro` with:

```astro
---
import BaseLayout from "../layouts/BaseLayout.astro";
import { getRepoPage } from "../lib/content";
import { renderMarkdown } from "../lib/content/normalize";

const page = await getRepoPage("resources.md");
const html = await renderMarkdown(page.body);
---
<BaseLayout title="Resources">
  <section class="shell" style="padding:32px 0 64px;">
    <article class="panel" style="padding:28px;" set:html={html} />
  </section>
</BaseLayout>
```

Create `site/src/pages/methodology.astro` with:

```astro
---
import BaseLayout from "../layouts/BaseLayout.astro";
---
<BaseLayout title="Methodology">
  <section class="shell" style="padding:32px 0 64px;">
    <article class="panel" style="padding:28px;display:grid;gap:16px;">
      <p class="eyebrow">Curation Rules</p>
      <h1 style="margin:0;">Methodology</h1>
      <p style="margin:0;color:var(--muted);">
        This site is generated from the repository Markdown corpus. Incidents are included when there is credible, sufficiently detailed reporting and enough technical information to make the entry operationally useful.
      </p>
      <p style="margin:0;color:var(--muted);">
        Metadata is inferred from the existing file structure and the standardized attack template where possible. Where fields are inconsistent across historical entries, the site degrades gracefully instead of failing the build.
      </p>
    </article>
  </section>
</BaseLayout>
```

- [ ] **Step 4: Run build to verify all major routes compile**

Run: `cd site && npm run build`
Expected: PASS with generated routes for home, incidents, incident detail pages, patterns, resources, and methodology

- [ ] **Step 5: Commit**

```bash
git add site/src/pages/patterns/index.astro site/src/pages/patterns/[slug].astro site/src/pages/resources.astro site/src/pages/methodology.astro
git commit -m "feat: add pattern resources and methodology pages"
```

## Task 9: Add GitHub Pages Deployment Workflow

**Files:**
- Create: `.github/workflows/deploy-site.yml`

- [ ] **Step 1: Write the failing workflow check**

Run: `test -f .github/workflows/deploy-site.yml`
Expected: exit status non-zero because the Pages workflow does not exist yet

- [ ] **Step 2: Write the workflow**

Create `.github/workflows/deploy-site.yml` with:

```yaml
name: Deploy Astro Site

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: site/package-lock.json

      - name: Install dependencies
        working-directory: site
        run: npm ci

      - name: Build site
        working-directory: site
        run: npm run build

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: site/dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

- [ ] **Step 3: Run a workflow syntax sanity check**

Run: `sed -n '1,220p' .github/workflows/deploy-site.yml`
Expected: workflow shows `upload-pages-artifact` and `deploy-pages` steps with `site/dist`

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/deploy-site.yml
git commit -m "ci: add github pages deployment workflow"
```

## Task 10: Verify Production Build and Document Usage

**Files:**
- Modify: `site/README.md`
- Test: `site/dist`

- [ ] **Step 1: Write the final usage docs**

Update `site/README.md` to:

```md
# Supply Chain Attack Monitor Site

## Local development

```bash
cd site
npm install
npm run dev
```

## Production build

```bash
cd site
npm run build
```

## Deployment

The GitHub Actions workflow in `.github/workflows/deploy-site.yml` builds the Astro app and publishes `site/dist` to GitHub Pages when `main` is updated.
```

- [ ] **Step 2: Run production verification**

Run: `cd site && npm run build`
Expected: PASS and generated static files under `site/dist`

- [ ] **Step 3: Inspect generated output**

Run: `find site/dist -maxdepth 2 -type f | sort | head -n 40`
Expected: includes `index.html`, `incidents/index.html`, `patterns/index.html`, `resources/index.html`, `methodology/index.html`

- [ ] **Step 4: Commit**

```bash
git add site/README.md site/dist
git commit -m "docs: add astro site usage notes"
```

## Self-Review

- Spec coverage check:
  - Rich homepage: covered in Task 5
  - Incidents research browser: covered in Task 6
  - Generated detail pages: covered in Task 7
  - Pattern browsing: covered in Task 8
  - Resources and methodology: covered in Task 8
  - GitHub Pages deployment: covered in Task 9
  - Verification and docs: covered in Task 10
- Placeholder scan:
  - No `TODO`, `TBD`, or deferred placeholders remain
- Type consistency:
  - Incident fields are defined consistently across ingestion, browser, and detail tasks
