# Astro Research Browser Design

## Goal

Build a rich, research-grade static website for this repository using Astro and deploy it to GitHub Pages. The Markdown corpus in this repository remains the source of truth. The site should make the incident collection significantly easier to browse, filter, search, compare, and study than the raw repository does today.

## Why This Exists

The repository already has strong incident content, but it currently behaves like a set of standalone documents with a long timeline page. That format is acceptable for storage and authoring, but it is not the best presentation layer for researchers, defenders, or readers trying to identify patterns across incidents.

The site should solve that by adding:
- rich faceted browsing
- full-text search
- generated incident detail pages
- cross-incident pattern navigation
- a stronger editorial homepage

## Non-Goals

This first version will not include:
- any server-side runtime
- user accounts
- comments or annotations
- external database storage
- live ingestion from threat feeds
- dashboards that depend on perfect metadata normalization

## Delivery Constraints

- The site must deploy as static files on GitHub Pages.
- Existing Markdown files in `attacks/`, `README.md`, and `resources.md` must remain the source of truth.
- The implementation should live on a new feature branch in an isolated worktree.
- The first version should ship value without requiring a full metadata rewrite of all attack entries.

## Proposed Stack

- Astro for static site generation
- TypeScript for ingestion and site utilities
- Astro content collections or a thin custom ingestion layer for the Markdown corpus
- Client-side search and faceted filtering backed by a generated static index
- GitHub Actions for Pages build and deployment

## Repository Layout

Add a dedicated site application under `site/` so the repository remains easy to browse as a content repo.

Planned structure:

```text
/
├── attacks/
├── docs/
│   └── superpowers/
│       └── specs/
├── resources.md
├── README.md
└── site/
    ├── src/
    │   ├── components/
    │   ├── layouts/
    │   ├── pages/
    │   ├── styles/
    │   └── lib/
    ├── public/
    ├── astro.config.*
    ├── package.json
    └── tsconfig.json
```

## Information Architecture

### Home

Purpose:
- present the project clearly
- surface key stats and recent incidents
- provide strong entry points into the corpus

Content:
- hero section with site positioning
- summary metrics such as incident count, ecosystems covered, time span
- featured or recent incidents
- ecosystem cards
- pattern cards
- links to methodology and resources

### Incidents

This is the primary research surface.

Capabilities:
- full-text search
- filters for year, ecosystem, severity, attack type, and impact tags
- URL-synced query/filter state
- sort by newest, oldest, and possibly relevance or severity
- result cards that balance density with readability

### Incident Detail

Each file in `attacks/*.md` becomes a detail page.

Detail pages should include:
- rendered Markdown body
- structured metadata rail
- summary and quick facts
- table of contents
- source links
- related incidents based on shared metadata

### Patterns

Generated pages for recurring themes across incidents, such as:
- malicious package publish
- maintainer compromise
- typosquatting
- GitHub Actions tag poisoning
- credential theft
- wallet theft
- persistence
- worm propagation

These pages should act as pre-filtered browsing surfaces over the incident corpus rather than long essays.

### Resources

Promote `resources.md` into the navigation as a first-class page.

### Methodology

Add a short methodology page that explains:
- what counts as an incident in this repo
- preferred source types
- how metadata is inferred
- current limitations of the dataset

## Data Model

The site should build a normalized incident index from Markdown at build time.

Each incident record should include:
- `slug`
- `title`
- `date`
- `year`
- `month`
- `ecosystems`
- `severity`
- `attackTypes`
- `impactTags`
- `summary`
- `sourceLinks`
- `filePath`

Optional structured fields when available:
- `maliciousArtifacts`
- `affectedPackages`
- `iocs`
- `relatedIncidents`

## Metadata Strategy

Use a hybrid ingestion strategy:

1. Parse structured fields already present inside the Markdown files.
2. Infer useful metadata from headings and body content where practical.
3. Introduce lightweight frontmatter only where it materially improves quality.

This avoids blocking delivery on a full manual metadata migration while still letting the browsing experience feel structured.

## Search and Filtering Strategy

Search should be static and client-side.

Requirements:
- search across title, summary, ecosystems, attack type labels, and main body text excerpt/index
- fast filtering with no server dependency
- support combined filters
- persist state in the URL so views are shareable

Expected filter dimensions:
- year
- ecosystem
- severity
- attack type
- impact tag

## UI Direction

The user explicitly wants a rich UI and research-grade browsing.

The design should therefore feel more like a modern intelligence interface than a documentation theme:
- bold editorial landing page
- dense but readable browsing views
- strong hierarchy and metadata treatment
- clear visual grouping by ecosystem and pattern
- polished motion used sparingly and purposefully

The interface should avoid looking like a generic docs template.

## Implementation Phases

### Phase 1: Foundation

- create a feature branch and isolated worktree
- scaffold Astro app in `site/`
- configure static output and GitHub Pages base path handling
- add shared layout, theme variables, and navigation

### Phase 2: Content Ingestion

- implement file readers/parsers for the existing Markdown corpus
- normalize incidents into a generated static index
- validate output against the current set of attack files

### Phase 3: Core Research Surfaces

- build homepage
- build incidents browser with search and filters
- build incident detail pages
- build resources and methodology pages

### Phase 4: Cross-Incident Discovery

- build generated pattern pages
- add related incident logic
- improve metadata display and quick facts

### Phase 5: Deployment

- add GitHub Pages workflow
- verify production build output
- document local development, build, and deployment steps

## Testing and Verification

Minimum verification before completion:
- local Astro build succeeds
- incident pages generate for the current corpus
- search/filter UI works in the browser
- GitHub Pages workflow is valid
- no broken internal links in the generated site

## Risks

### Inconsistent Metadata

The existing attack files may not expose all fields consistently. The ingestion layer must degrade gracefully and not fail the build just because one field is missing.

### Search Index Size

If full-text indexing is too heavy, the first version should prioritize title, summary, and selected metadata fields, then expand once real size constraints are measured.

### GitHub Pages Base Path

The Astro config must be correct for repository-based Pages hosting, or assets and routes will break in production.

## Recommendation

Proceed with Astro in a dedicated `site/` directory on a new feature branch/worktree. Build a normalized incident index from the existing Markdown corpus, then use that index to power a rich static research browser optimized for GitHub Pages.
