# Index Page Changelog

This is the change history for `index.html` (the dashboard / homepage).
Newest revisions on top. Times are UTC.

This file also covers cross-cutting site changes that affect multiple pages — most non-core pages link here for their changelog rather than maintaining their own.

---

## Rev 3.30 — 2026-05-08

Stale-data UX overhaul. Three problems fixed in one pass: (1) DAO TLA Deposits and DAO TLA VP tiles spinning forever despite epoch-182 data being available, (2) red bordered banner inside the rewards card was redundant once you got a popup explaining the same thing, (3) staleness check happened way too late in the page lifecycle (after the 15s LCD calls finished).

### Diagnosis — why two TLA tiles were stuck

`buildTlaDeposits()` and `buildTlaVp()` had a `if (!hasValidTlaData || dataIsStale)` guard that swapped the entire render path for a spinner whenever data was stale. Original intent was good — Design Principle #1 says blank > stale. But with this rev's modal + red dot, "stale" is now visibly marked, so showing the actual numbers is fine. Hiding behind a perpetual spinner was just confusing because it looks indistinguishable from "still loading" forever.

Also discovered: **`adao_json_storage` has zero `adao-snapshot_*_end.json` files** — the entire pattern Rev 3.29's `fetchTlaFromGitHub()` was querying never existed. The actual data the dashboard tiles need (treasury, tla_deposits, etc.) lives in `tla_json_storage/tla-data-epoch-X-end.json` under the `dashboard.*` keys, and the existing `fetchTlaData()` already fetches it correctly. The `buildTlaDeposits()` function already had a fallback path to consume that data — it just wasn't being reached because the staleness check bailed first.

### Changes

**1. Tile builders render stale data instead of bailing**
- `buildTlaVp()`: changed `if (!tlaData || meta.isStale)` to just `if (!tlaData)` — only spin when data is genuinely missing
- `buildTlaDeposits()`: same change to `if (!hasValidTlaData)`

**2. New `applyTlaStaleness()` helper — single source of truth for staleness UI**
Lives near `tlaDataMeta` declaration (so it has scope access). Reads the meta, sets:
- 🟢 green `pulse-dot` on `dao-tla-title` + `dao-tla-vp-title` if fresh, 🔴 red `static-dot-red` if stale
- Small "Stale Data" pill (`unclaimed-stale-warning`) in the rewards card header — visible
- Status text (`unclaimed-data-status`) showing "Epoch X data" — visible
- Big red bordered banner (`unclaimed-stale-banner`) — explicitly hidden (replaced by modal)
- Modal popup if stale AND `sessionStorage.getItem('aDAO_stale_modal_dismissed')` is unset

Idempotent — safe to call multiple times. Called from three places: end of `updateUI()` (early), inside `fetchLiveOnChainData`'s finally block, and would be added to tile builders if needed (currently not — early `updateUI()` call is enough).

**3. New stale-data modal**
HTML at `<div id="staleDataModal">` near the other modals. Shows on launch when stale; user clicks "Got it — show me the dashboard" to dismiss; sessionStorage flag prevents re-show within the tab session. Standard modal pattern (click outside, Escape, X to close).

**4. Banner display removed from two places**
- The `if (meta.isStale)` branch in the rewards-card stale handler — now only sets the pill + status + yellow tint, no banner
- The Rev 3.29 finally-block code in `fetchLiveOnChainData` — replaced with single `applyTlaStaleness()` call. The banner HTML element itself is left in place (zero footprint when hidden) for now; can be deleted in a cleanup pass.

**5. Early staleness check — popup appears within ~500ms of load**
`updateUI()` now kicks off `fetchTlaData()` in parallel with `fetchLiveOnChainData()`. Since fetchTlaData is internally cached, all the downstream callers (buildTlaVp, buildTlaDeposits, etc.) reuse the same in-flight promise — no duplicate network calls. The `.then(() => applyTlaStaleness())` fires the modal as soon as the snapshot resolves, regardless of how slow the on-chain calls are.

### What this didn't fix

- **DAO Broken/Held NFTs tile is still spinning.** This one isn't actually a TLA snapshot issue — the tile element (`#dao-broken-total`) has zero live-data writes anywhere in the codebase. Only the chart icon attaches to it. It's missing implementation, not a stale-data problem. Probably wants `nfts.filter(n => n.broken && n.owner === DAO_MAIN_WALLET).length` populating it from the live `nfts` array, but that's a design call. Logged in `CHANGES_PENDING.md`.
- **Rev 3.29's `fetchTlaFromGitHub` is now redundant.** It still runs and harmlessly returns null (since `adao_json_storage` is empty). The downstream builders already fall back to `fetchTlaData` correctly. Worth deleting in a cleanup pass — logged.

---

## Rev 3.29 — 2026-05-08

TLA snapshot fetch: parallel epoch-fallback + staleness UI. Closes the gap that Rev 3.27 left — the dashboard's TLA tiles (DAO TLA Deposits, DAO TLA VP) were spinning forever any time the latest snapshot was missing, even when older snapshots existed.

### Background
Two separate fetches consume snapshot data on this site:
- `fetchTlaData()` (line ~7700) reads `tla_json_storage` for the **TLA Stats page** — already had epoch fallback
- `fetchTlaFromGitHub()` (line ~3529) reads `adao_json_storage` for the **dashboard tiles** — had no fallback

When the user misses a Sunday-night snapshot capture (e.g. site outage prevented it), the second function's `currentEpoch-1` fetch 404s and the dashboard tiles never recover. Older snapshots are sitting right there in the repo, but the code never tries them.

### What changed

**`fetchTlaFromGitHub()` rewrite — parallel epoch walk:**
- Tries 6 candidate epochs simultaneously: `[currentEpoch+1, currentEpoch, currentEpoch-1, currentEpoch-2, currentEpoch-3, currentEpoch-4]`
- `Promise.allSettled` — total wall-time is one round-trip regardless of which one wins (vs. up to 6 sequential round-trips in the old TLA Stats pattern)
- 5-second per-fetch timeout via `AbortController` so a hung GitHub CDN can't stall the dashboard
- In-memory cache keyed by `currentEpoch` so re-renders within a session reuse the result; auto-invalidates when the epoch number rolls Sunday 23:59 UTC

**Staleness convention:**
- Found at `currentEpoch+1` / `currentEpoch` / `currentEpoch-1` → **fresh** (the healthy expected state)
- Found at `currentEpoch-2` or older → **stale**, with `epochsBehind` count
- Returned as `{data, epoch, currentEpoch, epochsBehind, isStale, timestamp}`

**`updateTlaFromGitHub()`** now returns the same metadata so the caller can render the right state instead of just a boolean.

**Caller (the `finally` block in `fetchLiveOnChainData`)** now does three things on success:
1. **Green pulse dot** on `dao-tla-title` and `dao-tla-vp-title` when fresh
2. **Red `static-dot-red`** + tooltip explaining the lag when stale
3. **Reveals the existing `unclaimed-stale-banner`** with epoch numbers populated, so users see the same warning the TLA Stats page already shows when data is behind. No new modal — reuses what was already there.

On total failure (all 6 epochs unreachable), no dot is added — spinner stays so the user knows the state is genuinely unresolved rather than being shown stale data dressed up as fresh.

### Why parallel matters for speed
The previous TLA Stats fallback walks epochs sequentially. If the user missed 2 epochs, that's 3 sequential fetches before finding data — easily 1–2 seconds of avoidable wait time. Parallel collapses that to one round-trip. The new dashboard fetch will load fast even in the worst-case stale scenario.

### Open items not addressed in this rev
- **Apply the same parallel pattern to `fetchTlaData()`** (TLA Stats page) — it works correctly today but is unnecessarily slow on stale fallback. Easy port; logged for a follow-up.
- **Per-fetch timeout in `fetchLiveOnChainData`** for the slow Terra LCD calls — 14.67s load time on yesterday's session is mostly from there, not from TLA. Logged.

---

## Rev 3.28 — 2026-05-08

Mint Status slider — was permanently spinning. Bug existed since the slider was added; only became visible once Rev 3.27 fixed the cascade and the rest of the dashboard started loading reliably.

### Root cause
The slider's three text spans (`#unminted-count`, `#minted-count`, `#minted-percent`) and the bar (`#mint-slider`) were only touched by `updateUI()` at startup. That function reads `dashboardData.statusSliders.mint.percentMinted` — but that field is hardcoded `null` (line 3373) and nothing in the codebase ever writes to it. So the read failed the null check, the slider was left untouched, and the initial spinner HTML stayed forever. By contrast, the Broken Status slider next to it has a parallel block in `fetchLiveOnChainData()` that writes to its DOM elements directly using the live `brokenCount` — that's why one worked and the other didn't.

### Fix
Added a Mint Status slider update block in `fetchLiveOnChainData()` immediately after the existing Broken Status block. Uses the same `mintedCount` / `unmintedCount` values already derived in Rev 3.26's null-safety patch (line ~6164), so no new data fetching needed. Color convention matches the Broken slider: cyan-blue = the highlighted half ("Minted" here, "Broken" there), gray = the remaining half. All formatters use the Rev 3.27 `safeLocale` / `safeFix` helpers, so a missing upstream value shows `—` instead of crashing.

---

## Rev 3.27 — 2026-05-08

Comprehensive null-safety fix — the same `TypeError: Cannot read properties of null (reading 'toLocaleString')` from Rev 3.26 was still crashing `fetchLiveOnChainData()`, just at different sites that 3.26 didn't touch.

### What was still broken after Rev 3.26

Rev 3.26 fixed the `mintedCount.toLocaleString()` crash by adding `fmt(v)` and applying it to the supply / unminted-modal block (lines ~6313–6318, 6346–6347). But the function has **15+ other** `.toLocaleString()` and `.toFixed()` calls that were still unguarded:

- **Broken-count slider** (lines 6325–6331): `brokenCount.toLocaleString()`, `(totalNftsForBrokenCalc - brokenCount).toLocaleString()`, `(brokenCount / totalNftsForBrokenCalc * 100).toFixed(2)`
- **Backing tile updates** (lines 6333–6338): `liveAmpLunaBacking.toFixed(4)`, `ampLunaToLunaRate.toFixed(4)`, `backingInLunaLive.toFixed(4)`, `lunaPriceUSD.toFixed(4)`, `backingInUSD.toFixed(2)`, `unmintedBackingLuna.toLocaleString(...)`, `unmintedBackingUSD.toLocaleString(...)`
- **Unminted modal text content** (lines 6348–6351): `backingInLunaLive.toFixed(4)`, `backingInUSD.toFixed(4)`, `unmintedBackingLuna.toLocaleString(...)`, `unmintedBackingUSD.toLocaleString(...)`
- **Catch block** (lines 6368–6377): the *recovery* path called `keyMetrics.daodaoStaked.toLocaleString()`, `keyMetrics.daoMembers.toLocaleString()`, `keyMetrics.enterpriseStakedHolder.toLocaleString()`, `keyMetrics.backingInLuna.toFixed(2)` — but those keyMetrics fields are hardcoded `null` at line 3370. So the catch crashed on its first line and **never applied any of its 'Error' fallbacks**, leaving downstream tiles permanently stuck on the initial spinner.

### Today's trigger

`terra.publicnode.com` returned HTTP 500 on a treasury balance query (the contract balance for `terra1sffd…3m5vzm`). That null propagated into the unguarded format calls and triggered the cascade. It "worked yesterday" simply because the LCD was up yesterday — the underlying bug has been latent since Rev 3.21's honest-data cleanup removed the static fallbacks that masked it.

### Fix

- **Hoisted null-safe formatters** above the `try` block so they're visible in both `try` and `catch`:
  - `safeLocale(v, opts)` — null-safe `.toLocaleString('en-US', opts)`, returns `'—'` for null/undefined/NaN
  - `safeFix(v, d=2)` — null-safe `.toFixed(d)`, returns `'—'` for null/undefined/NaN
- **Made backing calcs null-aware** so a missing upstream value propagates as `null` instead of being silently coerced to `0` (which would display as "0 LUNA" — a Design Principle #1 violation, since "0" looks like real data):
  ```js
  // Was: const backingInLunaLive = liveAmpLunaBacking * ampLunaToLunaRate;  // null * num = 0
  // Now:
  const backingInLunaLive = (liveAmpLunaBacking != null && ampLunaToLunaRate != null)
    ? liveAmpLunaBacking * ampLunaToLunaRate : null;
  ```
  Same treatment for `liveAmpLunaBacking`, `backingInUSD`, `unmintedBackingLuna`, `unmintedBackingUSD`.
- **Replaced every unguarded format call** in the affected lines (6325–6351 in try, 6368–6377 in catch) with `safeLocale` / `safeFix`.
- **Reset broken-count slider in catch** so it doesn't stay on its initial spinner if recovery runs.

### Why the catch had the same bug

This was a textbook "the recovery path was never tested" scenario. The catch was written assuming `keyMetrics.X` was always populated with sensible defaults — but Rev 3.21's honest-data cleanup left those at `null`. Since the catch only fires on errors, and errors on prod were rare until today's LCD outage, nobody noticed the recovery itself was broken.

### Open items not addressed in this rev

- **Per-fetch try/catch isolation** would be a stronger fix — currently one failed treasury balance fetch poisons every downstream calculation. Logged for future revisit; not in scope here.
- **Fallback LCD endpoint** — both `terra.publicnode.com` and `terra-lcd.publicnode.com` returned 500s today. A retry against a different public LCD on 5xx would significantly improve resilience. Logged for future revisit.
- **The `fmt` helper from Rev 3.26 is now redundant with `safeLocale`** but left in place for stability — can be deduplicated in a future cleanup pass.

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
