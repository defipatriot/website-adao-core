# Changes Pending — aDAO Website

> Rolling list of identified work. Active items at top, completed items moved to the changelog (`index-log.md`) once shipped.
> 
> **For Claude:** Update this file proactively. See "Tracking responsibilities" in `PROJECT_KNOWLEDGE.md` — when the user mentions a future feature in passing, when we notice tech debt while doing other work, when we leave an open question unresolved — log it here without waiting for the user to ask.

---

## 🟢 Just shipped (Rev 3.25)

Duplicate header cleanup pass — same treatment we gave the 4 core pages, now applied to the 10 pages that got chrome'd in Rev 3.24:
- 6 simple "logo + page title + logo" duplicate rows removed entirely (tutorials, rarity-explained, release-history, links, alliances, ally)
- `tools.html` had its OLD top nav (4-tab dropdown layout) removed — redundant with the shared 5-tab nav
- `dao_treasury` and `dao_tla_deposits` sticky headers cleaned: dropped Dashboard backlinks and titles, kept functional bits (Live indicator on Treasury, period buttons + epoch badge + Treasury cross-link on Deposits)
- `tla-docs` had its small H1+tagline header removed

All 10 pages got rev bumps. Index bumped to 3.25.

## 🟢 Just shipped (Rev 3.24)

Big batch — Phase 2 cross-page chrome rollout + site-wide favicons + Vercel analytics + deving.zone API URL update. See `index-log.md` Rev 3.24 entry for details.

- 12 user-facing pages got the unified header/nav/footer/changelog system: tools, ally, tutorials, alliances, links, rarity-explained, release-history, tla-docs, dao_treasury, dao_tla_deposits, ampcapa-tool, fuel-tool
- All user-facing pages now have favicons (9 added) and Vercel Web Analytics (5 added)
- Admin pages (`tla_tool`, `tla-tool_ext`, `dao_governance_tool`) got favicons + analytics but no chrome (intentional — admin context)
- `tools.html` had a duplicate footer cleaned up
- Sitemap pushed (was missing from repo)
- **deving.zone API URL** migrated from `/en/nfts/alliance_daos.json` → `/nfts/alliance_daos.json` (7 occurrences across 5 files). API now refreshes hourly instead of every 6 hours.

## 🟢 Just shipped (Rev 3.23 + per-page hotfixes)

See `index-log.md` and per-page logs for full details. High-level:
- **All 5 changelog modals fixed** — fetch URL was `/main/logs/<file>.md` (404), corrected to `/main/<file>.md` since logs live at the root of `website-adao-core`. Affects index + 4 core pages
- **NFT Explorer (rev 4.13):** removed dual logos from second header row, removed Map view entirely (Collection / Wallet only now)
- **aDAO Lore (rev 2.9):** removed duplicate header row (logos + "Galaxy Map & Lore" title)
- **TLA Stats (rev 1.15):** cleaned second header row (removed aDAO logo, "← Dashboard" backlink, "by The Alliance DAO" prefix; kept title and Eris TLA link). Cleaned footer (removed "Updated:" date, "Built by DeFi Patriot" credit, copyright notice; kept disclaimers and ecosystem links)
- **DAO (rev 1.5 → 1.6):** changelog URL fix; then matched TLA Stats cleanup — removed small aDAO logo, "← Dashboard" backlink, "by The Alliance DAO" subtitle from page-specific header

## 🟢 Previously shipped (Rev 3.22)

See `index-log.md`. High-level for Rev 3.22:
- 4 file renames: `planet-map` → `adao-lore`, `capa_lp_converter` → `ampcapa-tool`, `fuel_tracker` → `fuel-tool`, `dao_governance` → `dao`
- Home tab added to top nav (now 5 tabs everywhere)
- Active page highlighting in both top nav and bottom nav
- 4 core pages got the unified header / nav / footer / changelog system: NFT Explorer, aDAO Lore, TLA Stats, DAO
- 4 new log files created: `explorer-log.md`, `lore-log.md`, `tla-log.md`, `dao-log.md`

## 🟢 Previously shipped (Rev 3.21)

See `index-log.md`. High-level: SEO/PWA setup, mobile redesign, navigation overhaul, honest data display, changelog system itself.

---

## 🟡 Outstanding tech debt (not urgent, but worth knowing)

- **NFT Explorer JS still has Map view code** — `nft-explorer-app.js` still has `mapViewBtn`, `spaceCanvas`, `initializeStarfield`, `handleMapResize`, `switchView('map')` branch, and ~hundred lines of starfield rendering / pinch-zoom / pan logic. All gated by null-safe checks so nothing breaks, but it's dead weight (~? KB). Strip in a future cleanup pass when convenient.
- **`vercel.json` audit** — never confirmed whether file exists in `aDAO-links-site` repo. Need to verify and add 308 redirects for the 4 renamed files: `planet-map.html` → `adao-lore.html`, `capa_lp_converter.html` → `ampcapa-tool.html`, `fuel_tracker.html` → `fuel-tool.html`, `dao_governance.html` → `dao.html`. Without these, external links / Google index entries / bookmarks to old URLs will 404.
- **Admin page chrome decision** — `tla_tool.html`, `tla-tool_ext.html`, `dao_governance_tool.html` are intentionally without the public 5-tab nav since they're internal tools. If they ever need a way to navigate back to the public site, decide on a minimal admin-specific nav (probably just a "← Dashboard" link). Defer until a real need arises.

---

## 🔴 Active / next round

### High priority — SEO discoverability ⚠️

Site is not surfacing in Google for "the alliance dao" — `alliance.xyz` (unrelated startup accelerator) dominates. Tasks to address:

- [ ] Submit `sitemap.xml` to **Google Search Console** (https://search.google.com/search-console). Without this, Google won't reliably index/recrawl.
- [ ] Verify domain ownership in Search Console (DNS TXT or HTML file method).
- [ ] Add **JSON-LD structured data** to `index.html` so Google understands what we are — minimum: `Organization` and `WebSite` schemas with name "The Alliance DAO" (note: NOT "Alliance DAO" — that's the unrelated accelerator), URL, logo, description.
- [ ] Apply same SEO meta tag pattern to all other 18 pages (only `index.html` has full SEO tags right now).
- [ ] Submit also to **Bing Webmaster Tools** (lower priority, but covers Bing/DuckDuckGo).
- [ ] Get backlinks from related Terra ecosystem sites (Eris Protocol, BBL, Terra Hub, Terra Classic refugees, etc.) — biggest SEO factor we can't fix in code. Coordinate with partners.
- [ ] Wait 2–6 weeks for Google to recrawl and re-rank after sitemap submission.

### High priority — page work

- [ ] ~~**Add bottom tab bar to all other pages**~~ — superseded by Cross-page consistency rollout above (which includes this plus much more)
- [ ] **Delete duplicate Vercel project** `a-dao-links-site` (keep `a-dao-links-site-t6nu`).
- [ ] **rawgit.hack bug fix:** in `tla_tool.html` line 142, change ext-tab from `https://raw.githack.com/defipatriot/aDAO-links-site/main/tla-tool_ext.html` to local `tla-tool_ext.html`. Eliminates the "One more step" interstitial that pops up when users click the ext tab.

### Medium priority

- [ ] **Test on more devices** — currently only verified on desktop Chrome + iPhone 16 Chrome (browser + PWA). Need testing for: Safari, Android, iPad portrait/landscape, smaller iPhones (SE/mini), Pro Max sizes, Firefox.
- [ ] **Fix iPad portrait gap** (640–768px width). Either treat iPad as phone (extend bottom nav to 1024px) or improve desktop layout at small viewports. Currently untested but theoretically awkward — sees desktop nav with phone-tight tile grid.

### Low priority / cleanup

- [ ] **LST ratio default values** — `bLUNA || 1.6048`, `ampLUNA || 1.9015`, `arbLUNA || 2.6873` — ~10 places in code. Violates Design Principle #1 (honest data). Decide: remove or keep? Ratios drift slowly so fallbacks less misleading than data fallbacks, but still inconsistent with other tile behavior.
- [ ] **Remove dead Logos modal HTML** (~line 1745 area in index.html) — the trigger was removed but the modal markup is still in DOM.
- [ ] **Remove dead `'logo-modal-trigger'` JS mapping** (~line 5396 area in index.html).

---

## 🚧 Future projects (separate threads when started)

### Activity dashboard / aggregated changelog page 🆕

A dedicated page (e.g., `changelog.html` or `activity.html`) that aggregates per-page changelog data and GitHub activity. Helps the user see what was last touched and what's gone stale.

Features (rough scope):
- Reads each page's `logs/<page>-log.md` from the `website-adao-core` repo
- Shows a summary table: page name, current rev, last updated date, days since last update
- Highlights pages that haven't been updated in 60+ days as "stale" — useful for noticing forgotten pages
- Optionally: pulls GitHub commit/activity data (via GitHub API) to show recent commits, author activity feed, "last 10 changes across all pages"
- Could also surface: pages that exist in the repo but aren't in the changelog system, sitemap drift, etc.

Why useful:
- Single place to check "when was X last worked on?"
- Identifies stale content automatically
- Public-facing transparency — users/community can see active development
- For the user personally: helps remember what was being worked on after a break

Implementation sketch:
- New `activity.html` page in main repo
- Fetches all `logs/*.md` files from `website-adao-core` raw URLs (could use GitHub API to list files in `logs/`)
- Optionally calls `https://api.github.com/repos/defipatriot/aDAO-links-site/commits` for recent activity
- Could be linked from footer or main nav (decide later)

Dependencies: makes most sense after Cross-page consistency rollout is done (so logs exist for all pages).

### Service worker (PWA polish)
- Offline shell — show app UI even without network
- Faster repeat loads via asset caching
- Update detection — notify users when new version deployed
- Push notifications (deferred — needs cron jobs first to have something to notify about)

### TLA data collection automation
- **Vercel cron** (defined in `vercel.json`) pointing to a serverless function endpoint
- Cron schedule: end of each TLA epoch — Sunday 23:59 UTC (`59 23 * * 0`)
- Port browser-based TLA collection logic from `tla_tool.html` to a Node.js serverless function
- Function commits JSON snapshots back to the 3 storage repos via PAT (stored in Vercel env vars)
- Replaces manual Sunday-night data capture (currently has to be done by user every week)
- Notes: `epoch_1-300_date.json` from `website-adao-core` is useful reference. Logic is currently 100% browser-side — needs porting. Watch out for Vercel function execution time limits (10–60s depending on plan).

### Capa Protocol integration prep
- New pages/sections for Capa marketplace when partnership solidifies
- Lore integration (per the Lion DAO framework — DAOs become tribes in The Lattice)
- Will need to update `alliances.html` plus possibly its own page

### Static site generator migration (long-term, not urgent)
- Astro or Eleventy would let us template head meta tags + nav once instead of per-page
- Would simplify SEO Phase 2 and bottom-nav-everywhere tasks
- Big refactor — only worth it if we're going to keep adding pages
- Would also help with the file-size issue (index.html ~700KB, tla_tool.html ~621KB)

### Per-page changelogs (deferred)
- System is already designed to support this — each page can have its own `<page>-log.md`
- e.g., `tla-stats.html` → `logs/tla-stats-log.md`
- Implementation: copy changelog modal HTML+JS to those pages, change the URL
- Decide if/when this is worth it — probably wait until per-page changes accumulate

---

## 📝 Open questions / decisions

## 📝 Open questions / decisions

- **🆕 308 redirects for renamed files** — old URLs (`planet-map.html`, `capa_lp_converter.html`, `fuel_tracker.html`, `dao_governance.html`) need 308 redirects to their new equivalents in `vercel.json` so any external links (Google indexing, partner site backlinks, bookmarks) don't 404. Verify `vercel.json` exists and add redirect rules. **Action:** Add to next round.
- **Capa partnership** — when it goes live, own page or section in `alliances.html`?
- **NFT contract address** — currently footer-only, should it be more prominent somewhere?
- **iPad behavior** — phone-style with bottom nav up to 1024px, or desktop-style starting at 768px?
- **SEO branding** — do we lean into "The Alliance DAO" (full phrase, ambiguous with `alliance.xyz`) or "aDAO" (shorter, more unique) as the primary searchable term?
- **LST ratio fallbacks** — keep (slow-drifting, low harm) or remove (consistency with Design Principle #1)?
- **`alliance-dao-docs.html` content** — user has it saved separately for future reference (had on-chain address fetching logic). When/if needed, ask user for the file.

---

## Changelog rev numbering convention

- Format: `MAJOR.MINOR` (e.g., `3.21`)
- **MAJOR** bumps for big architectural shifts (entire dashboard redesign, new framework, etc.)
- **MINOR** bumps for any push that ships meaningful user-visible change
- Multiple commits between releases collapse into one rev number — we don't bump for every commit, only for "I just pushed this and it's the version users will see"
- Date format in log: `YYYY-MM-DD` (UTC)
- Log entries: newest at the top, older below
- **Doc-only changes (this file, PROJECT_KNOWLEDGE) do NOT bump the rev** — rev tracks site UI state only
