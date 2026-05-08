# Index Page Changelog

This is the change history for `index.html` (the dashboard / homepage).
Newest revisions on top. Times are UTC.

This file also covers cross-cutting site changes that affect multiple pages — most non-core pages link here for their changelog rather than maintaining their own.

---

## Rev 3.26 — 2026-05-08

Bug fix — broken tile cascade.

### Root cause
A `TypeError: Cannot read properties of null (reading 'toLocaleString')` was crashing `fetchLiveOnChainData()` partway through, leaving 7+ stat tiles permanently spinning (Mint Status, Broken Status, DAO Treasury, DAO TLA Deposits, DAO TLA VP, NFT Backing tiles, Unminted NFT Backing). The crash happened at `mintedCount.toLocaleString()` because `mintedCount` was always null.

The Rev 3.21 "honest data display" cleanup removed snapshot fallbacks but left two read paths (`dashboardData.statusSliders.mint.minted` and `.unMinted`) reading from values that were no longer being populated by anything. They stayed at their initialization value of `null` forever.

### Fix
- **Derive `mintedCount` and `unmintedCount` from the live `nfts` array** instead of the dead static-data path. Strategy: NFTs owned by the DAO main wallet (`terra1sffd4efk2jpdt894r04qwmtjqrrjfc52tmj6vkzjxqhd8qqu2drs3m5vzm`) are unminted (held by the DAO, not yet distributed). If the API ever exposes an explicit `minted` boolean per NFT, the code prefers that. Console logs the derived counts for verification.
- **Added null-safety helper `fmt(v)`** that returns `'—'` instead of crashing when a value is null. Applied to all toLocaleString calls in the supply / unminted-modal block.

### Other infrastructure
- **`site.webmanifest`** restored to repo root (was 404'ing — PWA manifest)
- **`favicon.ico`** still needs to be added at repo root (browser auto-requests `/favicon.ico`; the GitHub-hosted favicons in `<link>` tags don't catch this fallback request). Action item for the user — copy the file from `aDAO-Image-Files/favicon.ico` into the site repo.

---

## Rev 3.25 — 2026-05-08

Duplicate header cleanup pass — extending the Rev 3.23 fix to the 10 pages chrome'd in 3.24.

### Pages with simple "logo + page title + logo" duplicate row removed entirely
The page-specific second header was redundant with the shared header (which already provides logos). Removed:
- `tutorials.html` (1.4 → 1.5) — "Community Tutorials" title
- `rarity-explained.html` (1.2 → 1.3) — "NFT Rarity Explained" title
- `release-history.html` (1.3 → 1.4) — "NFT Release History" title
- `links.html` (1.3 → 1.4) — "Official & Helpful Links" title
- `alliances.html` (1.3 → 1.4) — "Our Alliances" title
- `ally.html` (3.3 → 3.4) — "ALLY Rewards Explained" title

### tools.html (1.3 → 1.4)
The page-specific second header had the OLD top nav (4-tab: NFT Explorer dropdown / TLA Stats / DAO) — redundant with the shared 5-tab nav above. Removed entirely. The CoinGecko price ticker that lived just below the old nav is preserved.

### Sticky functional headers cleaned (kept functional bits, dropped Dashboard/title)
Same treatment as TLA Stats / DAO got in earlier revs:
- `dao_treasury.html` (2.2 → 2.3) — removed Dashboard backlink and "DAO Treasury" title; kept Live data indicator
- `dao_tla_deposits.html` (2.2 → 2.3) — removed Dashboard backlink and "TLA Deposits" title; kept Treasury cross-link, period buttons, and epoch badge

### tla-docs (1.2 → 1.3)
Removed the small `<header class="header">` block containing "TLA Documentation" title + tagline. Page content makes purpose clear.

---

## Rev 3.24 — 2026-05-08

Phase 2 cross-page chrome rollout + site-wide favicons + Vercel analytics + deving.zone API URL update.

### deving.zone API URL update
The deving.zone API path migrated from `/en/nfts/alliance_daos.json` to `/nfts/alliance_daos.json`. Updated all 7 occurrences across 5 files: `index.html` (2x), `ally.html`, `links.html` (browser link), `tla_tool.html` (2x), `nft-explorer-app.js`. The API now refreshes hourly instead of every 6 hours, so NFT status data on the site will be more current going forward.

### Per-page chrome rollout (12 pages)
Brought the unified chrome system (5-tab top nav, mobile bottom nav, footer with Rev / Changelog button, page-specific changelog modal) to the remaining user-facing pages. All fetch this file (`index-log.md`) on changelog open since site-wide changes typically affect multiple pages and a single shared log is easier to maintain than 12 separate ones.

| Page | Starting rev |
|---|---|
| `tools.html` | 1.3 |
| `ally.html` | 3.3 |
| `tutorials.html` | 1.4 |
| `alliances.html` | 1.3 |
| `links.html` | 1.3 |
| `rarity-explained.html` | 1.2 |
| `release-history.html` | 1.3 |
| `tla-docs.html` | 1.2 |
| `dao_treasury.html` | 2.2 |
| `dao_tla_deposits.html` | 2.2 |
| `ampcapa-tool.html` | 1.3 |
| `fuel-tool.html` | 1.3 |

Page-specific headers and content preserved on all 12 — the shared header was inserted ABOVE existing headers, never replacing them. `tools.html` had a duplicate "social + Tutorials/Official Links/NFT Contract/Contract Audit" footer that was redundant with the shared footer, so it was removed.

### Site-wide favicons
Added the standard 4-link favicon block to all 9 user-facing pages that were missing it:
- `adao-lore.html`, `alliances.html`, `ally.html`, `dao_tla_deposits.html`, `dao_treasury.html`, `rarity-explained.html`, `tla-docs.html`, `tutorials.html`
- Plus `dao_governance_tool.html` (admin page, favicon only)

Now every user-facing page shows the aDAO logo as the browser tab icon.

### Site-wide Vercel Web Analytics
Added the Vercel `_vercel/insights/script.js` snippet to 5 pages that were missing it:
- `dao.html`, `dao_governance_tool.html`, `dao_tla_deposits.html`, `dao_treasury.html`, `tla-docs.html`

All 19 user-facing pages (everything except the Google verification stub) now report analytics.

### Admin pages — partial treatment
`tla_tool.html`, `tla-tool_ext.html`, `dao_governance_tool.html` already had favicons + Vercel analytics from earlier work, but did NOT get the chrome rollout. They're internal admin tools where the public 5-tab navigation would be misleading context. This is a deferred decision logged in CHANGES_PENDING.

---

## Rev 3.23 — 2026-05-08

Quick fix after Rev 3.22 went live.

### What changed
- Fixed changelog modal — was fetching from `/main/logs/index-log.md` (404), now fetches from `/main/index-log.md` to match where the file actually lives in `website-adao-core`. Same fix applied to all 4 core tab pages (NFT Explorer, aDAO Lore, TLA Stats, DAO)

### Convention now codified
Log files in `defipatriot/website-adao-core` live at the **root** of the repo, not in a `/logs/` subdirectory. PROJECT_KNOWLEDGE.md updated to reflect this.

---

## Rev 3.22 — 2026-05-08

Cross-page consistency rollout (phase 1) — top-level navigation pages.

### What changed
- **4 files renamed for cleaner URLs:**
  - `planet-map.html` → `adao-lore.html`
  - `capa_lp_converter.html` → `ampcapa-tool.html`
  - `fuel_tracker.html` → `fuel-tool.html`
  - `dao_governance.html` → `dao.html`
- **Top nav now has 5 tabs** including a new Home tab (was 4 — Home now shows on every page including index)
- **Active page highlighting** — current page's tab gets cyan styling in both desktop top nav and mobile bottom nav
- All internal references in `index.html` and `tools.html` updated for the renamed files
- `sitemap.xml` rewritten to reflect current page list with new names

### Cross-page rollout (separate file pushes, listed here for visibility)
Four core tab pages now have unified header + 5-tab top nav + mobile bottom nav + footer + per-page changelog system, with their original page-specific controls preserved:
- `nft-explorer-index.html` — starting rev 4.12 (Collection/Wallet/Map view toggles preserved)
- `adao-lore.html` — starting rev 2.8 (galaxy map content preserved)
- `tla-stats.html` — starting rev 1.14 (epoch selector + all charts preserved)
- `dao.html` — starting rev 1.4 (Members/Proposals tabs + audit tool link preserved)

Each of those 4 pages fetches its own changelog. All other pages (when added in a future rollout phase) will fetch `index-log.md` since most site-wide changes happen on the homepage anyway.

---

## Rev 3.21 — 2026-05-07

This revision consolidates a wide-ranging modernization of the dashboard. Major themes: SEO/PWA readiness, mobile redesign, more honest data display, navigation cleanup.

### SEO & discovery
- Added canonical URL, full Open Graph + Twitter Card meta tags, robots directive, theme-color
- Improved meta description (~150 chars)
- Sitemap rewritten — removed 7 dead pages, added 8 active ones, re-prioritized live dashboards

### PWA / install support
- `site.webmanifest` upgraded with categories, maskable icons, 3 quick-launch shortcuts
- App install instructions modal (iPhone / Android steps)
- App-launch default-page selector (only visible when running as installed app)
- Mobile bottom tab bar (Home / NFTs / Lore / TLA / DAO) — fixed-position, native-app feel, auto-highlights active page

### Navigation overhaul
- Top nav restructured to 4 equal-width tabs: NFT Explorer · aDAO Lore · TLA Stats · DAO
- Galaxy Map promoted out of dropdown to its own "aDAO Lore" tab
- Mobile labels shortened: Explorer · Lore · TLA · DAO
- DAO Home link fixed (removed `?url=...erisprotocol.com` query param)
- Top info-cards grid expanded to **9 tiles** with two new dropdown-style tiles:
  * **DAO Links** (Main DAO + Council DAO sections, 10 destinations)
  * **Contract** (NFT Contract + Contract Audit)
- ALLY Rewards displays as a tall tile spanning 2 rows on desktop, normal tile on mobile

### Honest data display
- Removed all "snapshot fallback" data — tiles no longer show stale values when fetches fail
- Replaced em-dashes and "stale data" text with consistent loading spinners
- Removed red snapshot indicator dot — tiles either show live data (green pulse dot) or a spinner
- Treasury tile only displays a value when ALL assets successfully priced
- TLA Deposits / TLA VP use epoch-based staleness check (not date-based)
- "Please Note" disclaimer banner removed (no longer needed)

### Mobile redesign
- Single coherent mobile CSS block (replaced ~370 lines of layered overrides)
- Top info-cards: 3-col grid on mobile, all 9 tiles uniform compact size
- Stat tiles: tighter padding, smaller fonts, chart icons no longer overlap titles
- Live Activity rows: 2-line layout (NFT info on row 1, price + time on row 2)
- Marketplace collection tabs: segmented control style with proper sizing
- Listings filter: replaced 3-button toggle with single-select dropdown using collection logos
- Mobile-only filter dropdowns for Condition + Marketplace (replaces inline button groups)
- CoinGecko ticker thinner on mobile
- Tile titles wrap to 2 lines instead of truncating with ellipsis
- Tile text properly centered (CSS specificity fix)

### Cleanup
- Removed dead pages from repo: `graphs.html`, `news.html`, `rampt.html`, `on-ramp.html`, `off-ramp.html`, `alliance-dao-docs.html`, `test-page.html`
- Removed `Logos` modal trigger from footer
- Removed yellow disclaimer banner
- Vercel domain redirects fixed to 308 permanent (`theadao.com`, `www.theadao.com` → `www.thealliancedao.com`)

### Footer additions
- Rev number + Changelog link added to footer (this changelog)

---

## Rev 3.20 and earlier

History prior to Rev 3.21 was not formally tracked. Major prior revisions:
- **Rev 3.x** — multiple iterations of the unified dashboard view (current architecture)
- **Rev 2.x** — separate-pages era, before main dashboard consolidation
- **Rev 1.x** — initial release with basic NFT links / info hub layout

Going forward, each push will get its own rev entry here.
