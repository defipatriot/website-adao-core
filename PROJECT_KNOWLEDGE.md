# aDAO Website Project Knowledge

> **Canonical context document.** Loaded at the start of every chat session so Claude can pick up where we left off without the user re-explaining anything.
> 
> **For Claude:** This file is yours to maintain. See "Tracking responsibilities" below — you are expected to update this file proactively as the project evolves, not only when explicitly asked.

---

## ⭐ Tracking responsibilities — for Claude

The user has explicitly stated they will forget things and lose track. **Claude is the primary record-keeper.** Don't wait for the user to say "remember this" — when any of the triggers below fire during a conversation, prepare an updated copy of this file (or `CHANGES_PENDING.md`) at the same time as the actual work being done.

### When to update PROJECT_KNOWLEDGE.md (this file)

| Trigger | Add it to | Why |
|---|---|---|
| User states a preference as a rule (e.g. "always do X", "I never want Y") | "Design principles" section | These are persistent, not one-off |
| We discover a non-obvious bug root cause | "Critical CSS gotchas" or new gotcha section | Prevents repeat debugging |
| New external dependency added (CDN, API, data source) | "External dependencies" section | Recovery + audit |
| New CSS class pattern or naming convention established | "Class system" section | Code consistency across pages |
| A page is added or removed from the site | "Current pages" table + "Pages removed" list | Sitemap accuracy |
| New repo touched (image hosting, data storage, etc.) | "Storage / data repos" section | Future Claude needs full inventory |
| User mentions ecosystem entity by name (project, tool, person) | "Key entities" section | Avoid confusion with similar-named things |
| A device/browser is tested or fails | "Tested environments" section | Honest compatibility tracking |
| Naming collision discovered (e.g. our project vs. similar-named other) | "Naming clarification" section | SEO + comms accuracy |
| User clarifies a workflow constraint (e.g. file naming on download) | "Working conventions" section | Smooth handoffs |

### When to update CHANGES_PENDING.md

| Trigger | Add it to | Why |
|---|---|---|
| User mentions a feature in passing as "we should do this someday" | "Future projects" section | Don't lose ideas |
| We notice tech debt while working on something else | "Low priority / cleanup" section | Track without derailing current work |
| User asks an open question we don't answer immediately | "Open questions" section | Resume the discussion later |
| A bug found but not fixed in current session | "Active / next round" with priority | Visible queue |
| External task identified (e.g. "submit to Search Console") | Top of "Active / next round" if blocking | Visible queue |

### When to update index-log.md

| Trigger | Action | Why |
|---|---|---|
| User-visible change to `index.html` is shipped | Bump rev (e.g. 3.21 → 3.22), prepend new entry | Public changelog accuracy |
| Doc-only changes (this file, etc.) | Do NOT bump rev | Rev tracks site UI state only |
| Other pages get changes | Future: their own `<page>-log.md` (not yet implemented) | Per-page granularity if needed |

### How to know if something is "trackable"

If the answer to ANY of these is yes, it goes in a doc:
- "Will the next Claude need to know this if I forget to mention it?"
- "Is this a decision rather than just an implementation detail?"
- "Could this surprise someone who doesn't know the history?"
- "Did the user use words like 'always', 'never', 'prefer', 'remember', 'note', 'important', 'careful'?"

When in doubt, add it. Over-documentation is fine; lost context is not.

---

## Project basics

- **Owner:** defipatriot — council member of The Alliance DAO (aDAO)
- **Main repo (deployed code only):** `github.com/defipatriot/aDAO-links-site` — HTML, CSS, JS, sitemap, manifest. **Never put docs/notes here** — Vercel only deploys what's in this repo, so keep it focused on what's actually rendered to users.
- **Image hosting repo:** `github.com/defipatriot/aDAO-Image-Files` (favicons, logos, OG images, collection PFPs — referenced via raw.githubusercontent.com URLs)
- **Docs repo:** `github.com/defipatriot/website-adao-core` — `PROJECT_KNOWLEDGE.md`, `CHANGES_PENDING.md`, all changelog `*-log.md` files, plus `sitemap.xml`, `robots.txt`, `site.webmanifest`, etc. Everything lives at the **root** of the repo — no subdirectories. **All project documentation lives here, NOT in the main site repo.** Edits don't trigger Vercel redeploys.
- **Live URL (canonical):** `thealliancedao.com`
- **Live URL (alias, 308 redirects):** `theadao.com`
- **Vercel fallback URL:** `a-dao-links-site.vercel.app` (still works, useful for testing)
- **Deployment:** Vercel (auto-deploys on push to `main`)
- **Stack:** Static HTML + Tailwind CSS (CDN) + vanilla JS. No build step. Each `.html` file is self-contained.

### Repo size facts (rough)
- `index.html` ≈ 12,400 lines / ~700 KB — the main dashboard, by far the largest file
- `tla_tool.html` ≈ 621 KB — TLA admin tool
- `tla-tool_ext.html` ≈ 433 KB — TLA extensions tool
- `tla-stats.html` ≈ 377 KB — public TLA dashboard
- `dao_governance_tool.html` ≈ 268 KB
- All other pages are <150 KB

These large files matter when considering refactors (file size affects parse time + Vercel cold-start cost) and when planning a static-site-generator migration.

---

## ⭐ Design principles (user preferences as rules)

These came from explicit user statements across multiple sessions. Treat them as project-wide rules unless the user overrides for a specific case.

### 1. Honest data over false positives
**The user prefers blank tiles, error indicators, or spinners over showing stale/fallback data that might be wrong.** Quote: *"I would rather they all just be blank so people dont get wrong idea with the data... if it has data in it I may think its working."*

In practice:
- No hardcoded fallback values that mask broken fetches
- No "snapshot N stale" yellow text — replace with spinner
- Treasury totals only shown if ALL assets successfully priced
- Use green pulse dot for live, spinner for loading. No red dots, no em-dashes.
- Exception: LST ratios (`bLUNA || 1.6048` etc.) are kept because they drift slowly and the fallback is less misleading than the alternative. This is tech debt to revisit.

### 2. Clean over busy
User has repeatedly rejected denser UIs in favor of cleaner ones. Quote: *"makes it to busy."*

In practice:
- Footer = condensed link row, not a multi-column site map
- App install info = button + popup, not always-visible block
- Mobile tile titles = uniform short labels, not full descriptive names
- New UI elements should justify their visual weight

### 3. Mobile-first treatment, but desktop must still feel rich
The user tests primarily on iPhone. Desktop should look professional and use the extra space (e.g. ALLY Rewards as a tall hero tile on desktop), but mobile is the constraint that drives layout decisions.

### 4. Progressive enhancement, not progressive degradation
PWA shortcuts, install prompts, default-page selectors — all are bonus features for users who installed the app, but the site fully works without them. Never block core functionality on PWA-mode being detected.

### 5. Minimum required ceremony
- No build step
- No external libraries beyond what's needed
- No frameworks "just in case"
- Inline solutions when they're <50 lines (the markdown changelog parser is a good example — 30 lines, no library)

---

## What aDAO is

10,000-NFT collection on Terra blockchain, backed by LUNA staking rewards.

- **Supply breakdown:** 3,172 public / 5,828 unminted DAO Treasury / 1,000 broken (multisig security via Props 64-69)
- **Yield mechanics:** NFT contract holds an "Ally" token earning ~0.72% LUNA staking rewards via Terra Alliance, +40% boost via Eris Protocol ampLUNA conversion. 10% take rate to DAO Treasury.
- **Holder action:** "Break" an NFT for ~$12 backing (one-time, irreversible). Broken NFTs keep governance VP but stop accruing.
- **"Last NFT Standing" dynamic** as breaks accumulate — fewer claiming NFTs = more rewards per remaining NFT, since LUNA inflation continues regardless.

### Key entities

- **Main DAO** + **Council DAO** (~6 members) on DAODAO
- **Eris** — council member. Builds the entire **Eris Protocol** stack:
  - **Eris Protocol** — hosts TLA, the LST infrastructure
  - LSTs: **ampLUNA**, **arbLUNA**, **ampCAPA**, **ampROAR**
  - **Creda Finance** — money market built by Eris
  - **Votion** — vote aggregator built by Eris (lets users delegate voting)
- **Capa Protocol** — partnership in works (stablecoin issuer, possibly new NFT marketplace). Will eventually need lore integration.
- **TLA = Terra Liquidity Alliance** — ve(3,3) system, weekly epochs ending **Sunday 23:59 UTC**. Reward distribution mechanism.

### Lore framework
10 post-human tribes scattered across the galaxy via "The Lattice". Designed to be flexible for absorbing partner DAOs:
- **Lion DAO** → became "Canyon-Clans of Ozara North" (precedent for how partnerships fit lore)

### Storage / data repos
- `defipatriot/tla_json_storage` — current TLA snapshots
- `defipatriot/tla-ext_json_storage` — extended TLA data
- `defipatriot/adao_json_storage` — aDAO-specific snapshots
- `defipatriot/aDAO-Image-Files` — favicons, logos, OG images, collection PFPs
- `defipatriot/website-adao-core` — project docs + changelogs (this repo)

---

## Naming clarification — important for SEO and comms

There is an unrelated organization called "Alliance" / "Alliance DAO" that ranks first on Google when searching `the alliance dao`:

- **Alliance** at `alliance.xyz` — a Web3/AI startup accelerator (formerly "DeFi Alliance")
- Runs `@alliancedao` on X (Twitter) — has 690+ followers, says "Moved to @Alliance"
- Founded 2020, supported projects like Tensor, Kamino, Pump.fun

**This is NOT us.** We're "The Alliance DAO" — a Terra blockchain NFT project at `thealliancedao.com`.

This affects SEO strategy (we likely can't outrank them for the bare query) and any time we describe the project to outsiders. Use full phrase "The Alliance DAO" or shorter "aDAO" — not "Alliance DAO" alone, which is ambiguous.

---

## Current pages (in active use)

Each page has a target rev number for the changelog system rollout. See "Cross-page consistency" rollout in `CHANGES_PENDING.md` for status. **Note:** the rev numbers below were assigned by the user as starting points based on rough recollection of how much work each page has had — they are estimates, not exact commit counts.

| Display name | File | Starting rev | Notes |
|---|---|---|---|
| (Home) | `index.html` | 3.23 | The main dashboard, ~12.6k lines. Has the changelog system. |
| NFT Explorer | `nft-explorer-index.html` | 4.13 | Top nav tab. ✅ Cross-page chrome added in Rev 3.22. Map view removed in Rev 4.13. |
| aDAO Lore | `adao-lore.html` | 2.9 | Top nav tab. ✅ Renamed from `planet-map.html` in Rev 3.22. ✅ Cross-page chrome added. |
| TLA Stats | `tla-stats.html` | 1.15 | Top nav tab. ✅ Cross-page chrome added in Rev 3.22. |
| DAO | `dao.html` | 1.6 | Top nav tab. ✅ Renamed from `dao_governance.html` in Rev 3.22. ✅ Cross-page chrome added. |
| ALLY Rewards | `ally.html` | 3.2 | Top info-card tile. ⏳ Cross-page chrome pending. |
| Tutorials | `tutorials.html` | 1.3 | Top info-card tile. ⏳ Cross-page chrome pending. |
| Tools | `tools.html` | 1.2 | Top info-card tile (hub for fuel-tool, ampcapa-tool, tla_tool). ⏳ Cross-page chrome pending. |
| Rarity Info | `rarity-explained.html` | 1.1 | Top info-card tile. ⏳ Cross-page chrome pending. |
| NFT Releases | `release-history.html` | 1.2 | Top info-card tile. ⏳ Cross-page chrome pending. |
| Official Links | `links.html` | 1.2 | Top info-card tile. ⏳ Cross-page chrome pending. |
| Alliances | `alliances.html` | 1.2 | Top info-card tile. ⏳ Cross-page chrome pending. |
| DAO TLA Deposits | `dao_tla_deposits.html` | 2.1 | Linked from DAO Links dropdown tile. ⏳ Cross-page chrome pending. |
| DAO Treasury | `dao_treasury.html` | 2.1 | Linked from DAO Links dropdown tile. ⏳ Cross-page chrome pending. |
| Fuel Tool | `fuel-tool.html` | 1.2 | Linked from Tools page. ✅ Renamed from `fuel_tracker.html` in Rev 3.22. ⏳ Cross-page chrome pending. |
| ampCapa Tool | `ampcapa-tool.html` | 1.2 | Linked from Tools page. ✅ Renamed from `capa_lp_converter.html` in Rev 3.22. ⏳ Cross-page chrome pending. |
| TLA Docs | `tla-docs.html` | 1.1 | Linked from TLA Stats. ⏳ Cross-page chrome pending. |

### Admin / dev pages (not in user-facing changelog rollout)
| File | Purpose |
|---|---|
| `tla_tool.html` | TLA admin tool (data collection) |
| `tla-tool_ext.html` | TLA extensions tool |
| `dao_governance_tool.html` | DAO governance tool |

### Pages removed
`graphs.html`, `news.html`, `rampt.html`, `on-ramp.html`, `off-ramp.html`, `alliance-dao-docs.html`, `test-page.html` — all deleted from repo.

### Notable removed-page history
- **`alliance-dao-docs.html`** — the user noted this had on-chain address fetching logic that may be useful future reference. The user has the file saved separately. If we need the address-fetching pattern in the future, ask the user for it.

---

## External dependencies (CDN-loaded, no local fallback)

If any of these CDNs go down, the site breaks. Worth knowing for incident response.

| Resource | Source | Used for |
|---|---|---|
| **Tailwind CSS** | `cdn.tailwindcss.com` | All styling utilities |
| **Font Awesome** | `cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/` | All icons |
| **Inter font** | `fonts.googleapis.com` | Body font |
| **Chart.js** | `cdn.jsdelivr.net/npm/chart.js` | All charts |
| **Chart.js date adapter** | `cdn.jsdelivr.net/npm/chartjs-adapter-date-fns` | Date axis on charts |
| **CoinGecko widget** | `widgets.coingecko.com/gecko-coin-price-marquee-widget.js` | Top price ticker |
| **Vercel Insights** | `/_vercel/insights/script.js` | Analytics |
| **Image assets** | `raw.githubusercontent.com/defipatriot/aDAO-Image-Files/main/` | Favicons, logos, OG images |

### Live data sources
- **Terra LCD** endpoints (chain queries)
- **Backbone Labs (BBL)** marketplace API (NFT listings)
- **Boost** marketplace (NFT listings)
- **CoinGecko** (token prices)
- **Custom JSON snapshots** in the 3 storage repos above

---

## Tested environments — what we know works

**These are the ONLY environments verified working.** Anything else should be considered untested until confirmed. Update this table proactively whenever a new device/browser is tested.

| Environment | Status |
|---|---|
| Desktop Chrome | ✅ Tested, working |
| iPhone 16 (standard size) Chrome browser | ✅ Tested, working |
| iPhone 16 PWA (added to home screen, default browser is Chrome) | ✅ Tested, working |
| iPad portrait (640–768px width) | ⚠️ NOT tested — known awkward gap (sees desktop nav at small viewport) |
| iPad landscape | ⚠️ NOT tested |
| Android phones | ⚠️ NOT tested |
| Safari (iOS or macOS) | ⚠️ NOT tested |
| Firefox | ⚠️ NOT tested |
| Smaller iPhones (SE, mini) | ⚠️ NOT tested |
| Larger iPhones (Pro Max) | ⚠️ NOT tested |

When discussing compatibility, default to "this works in tested environments only" — don't claim cross-device support without actual testing.

---

## PWA update rules

Tracked here so we don't break installed PWAs accidentally.

### ✅ Updates automatically (no user action needed)
- HTML / CSS / JS changes — picked up on next app launch
- Icon image changes (replacing same filename in `aDAO-Image-Files` repo)
- `description`, `theme_color`, `background_color` in `site.webmanifest`
- New shortcuts added/removed in `site.webmanifest`
- Content updates anywhere

### ⚠️ May require user to reinstall to take effect
- Changing `start_url` in `site.webmanifest`
- Changing `scope` in `site.webmanifest`
- Changing `display` mode (e.g., `standalone` → `fullscreen`)
- Changing `name` or `short_name`
- Changing the manifest path or filename

**Rule of thumb:** structural changes to manifest = potential reinstall. Content/icon changes = automatic.

If we ever need to change one of the structural fields, plan to communicate to users they should remove and re-add the home screen icon.

---

## Site architecture conventions

### Class system

- `.stat-card` — base for all data tiles (DAO Treasury, Avg Daily Gain, etc.). Has `display: flex; flex-direction: column; justify-content: space-between`.
- `.stat-card-link` — same as stat-card but for `<a>` tags (clickable). Hover gets cyan glow.
- `.info-card` — also has `.stat-card`. Navigation tiles in the top grid (ALLY Rewards / Tutorials / etc.). Uses `align-items: center` + `text-align: center`.
- `.info-card-tall` — modifier on ALLY Rewards info-card. Spans 2 rows on desktop (`grid-row: span 2`).
- `.info-card-dropdown` — also has `.info-card`. Tiles that open a modal (DAO Links / Contract).
- `.mobile-bottom-nav` — fixed-position bottom nav, only shown on mobile (`≤767px`).
- `.show-only-on-mobile` / `.hide-on-mobile` — display utility classes for responsive content.

### Color palette (cyan/teal theme)
- Primary cyan: `#22d3ee` (text + active states)
- Light cyan: `#67e8f9` (hover + emphasized)
- Teal accent: `#2dd4bf` (icons, secondary)
- LUNA orange: `#ff7e1d` (TLA Locks branding)
- Bg: `#0a0b0f`

### Mobile breakpoints
- `≤640px` — main mobile breakpoint (most overrides live here)
- `≤380px` — small phones (extra tightening)
- `≤767px` — bottom tab bar visible up to this width
- `≥768px` — desktop top nav becomes visible (Tailwind `md:`)

### Critical CSS gotcha — DON'T REPEAT
- `.stat-card h3` mobile rule has `white-space: nowrap; text-overflow: ellipsis` for data tile titles.
- Info-cards inherit this (they're also `.stat-card`), causing text truncation with "..." instead of wrapping.
- **Fix already in place:** the rule is now scoped as `.stat-card:not(.info-card):not(.info-card-dropdown) h3` so info-card titles wrap normally.
- **General lesson:** when a CSS rule isn't applying despite `!important`, check whether a *more specific or later-defined* selector with equal specificity is winning the cascade. Don't pile on more `!important` — fix the selector.

### Modals
- Standard pattern: `<div id="xModal" class="modal fixed inset-0 hidden">` + `<div class="modal-content scale-95">`.
- Open: remove `hidden`, set opacity/scale on next animation frame.
- Close: reverse opacity/scale, then add `hidden` after 200ms.
- Click outside, Escape key, X button all close.

### Markdown changelog system
- Each page footer has Rev number + "Changelog" link.
- Modal fetches the relevant markdown log file from `website-adao-core` repo on open.
- Inline markdown parser handles `# / ## / ### / ####` headings, `-` lists, `**bold**`, `*italic*`, `` `code` `` inline.
- Updates: bump rev in HTML AND prepend a new `## Rev x.y — date` block to the relevant log file.

**Simplified log file scope** — only the **5 top-level navigation destinations** get their own dedicated log files. All other pages display `index-log.md` since most site changes happen on the homepage anyway:

| Page | Log file fetched | Notes |
|---|---|---|
| Home (`index.html`) | `index-log.md` | All-purpose site changelog |
| NFT Explorer | `explorer-log.md` | Tracks NFT Explorer-specific changes |
| aDAO Lore | `lore-log.md` | Tracks Lore page changes |
| TLA Stats | `tla-log.md` | Tracks TLA Stats changes |
| DAO | `dao-log.md` | Tracks DAO landing page changes |
| All other pages (ALLY, Tutorials, Tools, Rarity, NFT Releases, Official Links, Alliances, DAO Treasury, DAO TLA Deposits, Fuel Tracker, Capa Converter, TLA Docs, etc.) | `index-log.md` | Show the site-wide changelog |

This keeps the log surface manageable — 5 log files instead of 17+ — while still letting users see what's been worked on recently across the whole site.

**Naming convention reminder:** log files live at the root of the `website-adao-core` repo as `<short-name>-log.md` — NOT in any subdirectory. Use the short navigation label, not the full page filename (e.g. `explorer-log.md`, not `nft-explorer-index-log.md`).

### Cross-page consistency requirements
**Every user-facing page should look and feel like `index.html`.** This is a hard requirement — pages should be visually indistinguishable from each other except for body content. Specifically:

- **Header** — same logo + tagline + Terra logo as index
- **Top nav tabs (desktop)** — Home + NFT Explorer + aDAO Lore + TLA Stats + DAO (5 tabs total). Currently index has 4 (no Home tab); rolling this out adds the Home tab everywhere.
- **Mobile bottom nav** — Home + NFTs + Lore + TLA + DAO (already 5 tabs)
- **Active page highlighting** — both top nav and bottom nav should highlight the current page in cyan
- **Footer** — same condensed link row + Rev number + Changelog link
- **Color palette** — same cyan/teal theme (`#22d3ee`, `#67e8f9`, `#2dd4bf`)
- **Modal patterns** — DAO Links dropdown, Contract dropdown, App install info, Changelog — all available across pages

**Implementation note:** This is currently per-page duplicated code. A static site generator (Astro/Eleventy) would make this dramatically easier — see "Future projects" in CHANGES_PENDING. Until then, each page needs the shared HTML/CSS/JS copied in. When updating shared elements, every page must be updated.

---

## Data flow patterns

### Staleness handling (important — see Design Principle #1)
- Tiles either show **live data** (green pulse dot) or a **spinner**.
- No more red snapshot dots, em-dashes, or "stale data" text.
- TLA freshness uses `getTlaDataMeta().isStale` (epoch-based, more accurate than date-based).
- Treasury tile only displays a number when ALL assets successfully priced — partial sums mislead.

### Hardcoded fallbacks (LST ratios — known tech debt)
- `bLUNA || 1.6048`, `ampLUNA || 1.9015`, `arbLUNA || 2.6873` still in code in ~10 places.
- These ratios drift slowly (weekly), so the fallback is less misleading than for other data — but still violates Design Principle #1.
- **Open question:** keep or remove? Currently kept.

---

## PWA setup

- `site.webmanifest` defines app name, scope, icons, shortcuts.
- 3 quick-launch shortcuts: NFT Explorer, TLA Stats, DAO Governance.
- PWA mode detection script in `<head>` adds `pwa-mode` class to `<body>`.
- `.pwa-only` CSS class shows/hides UI only in installed-app mode (e.g., default-page selector).
- App install instructions in modal accessed via "App" footer link.
- Default-page preference stored in `localStorage` as `aDAO_default_page`.

### Mobile bottom tab bar
- Currently only on `index.html`.
- Uses fixed-position `<nav class="mobile-bottom-nav">` with 5 tabs: Home / NFTs / Lore / TLA / DAO.
- `env(safe-area-inset-bottom)` for iPhone home indicator clearance.
- Auto-detects active page from filename via inline script.
- **TODO:** add to all other pages so it persists across navigation.

---

## SEO situation — current state and what we know

Searching "the alliance dao" in Google does NOT surface our site (`thealliancedao.com`). Top results are unrelated `alliance.xyz` (the startup accelerator). See "Naming clarification" above.

### What we've already done
- Canonical URL meta tag on `index.html`
- Full Open Graph + Twitter Card meta tags on `index.html`
- `robots: index, follow`
- `theme-color`
- Improved meta description (~150 chars)
- `sitemap.xml` cleaned up — dead pages removed, active pages added with priorities
- 308 redirects from all alt domains to `www.thealliancedao.com`

### What we still need (next SEO project)
- Submit sitemap to Google Search Console — without this, Google won't reliably discover/recrawl
- Verify domain ownership in Search Console
- Apply meta tags to other 18 pages (currently only `index.html` has them)
- Add structured data (`Organization` and/or `WebSite` JSON-LD) so Google understands what we are
- Build backlinks — biggest SEO factor we can't control purely via code
- Wait. Even with everything right, Google can take weeks to months to reindex

### Key constraint
We're competing for "the alliance dao" against `alliance.xyz`, which has 5+ years of history, backlinks, and brand association. Realistic outcome: rank well for "thealliancedao", "aDAO Terra", "alliance dao terra" — niche queries where we're clearly the right answer. Probably can't outrank `alliance.xyz` for the bare query.

---

## Deployment / Vercel

- 308 permanent redirects on all 4 domain variants (`theadao.com`, `www.theadao.com`, `thealliancedao.com` → `www.thealliancedao.com`).
- Auto-deploy on push to `main` — every commit goes live within ~30 seconds.
- **Cron jobs run on Vercel** (defined in `vercel.json` pointing to serverless function endpoints). When building new scheduled tasks (TLA data collection, etc.), default to Vercel cron — don't suggest GitHub Actions unless there's a specific reason.
- One known leftover task: delete duplicate Vercel project `a-dao-links-site` (keep `a-dao-links-site-t6nu`).

---

## Working conventions for future chats

### Always start a coding session with
```
cd /home/claude/aDAO-links-site
git fetch && git reset --hard origin/main
```
Otherwise local copy goes out of sync with whatever was pushed since last chat.

### Push workflow (user-side)
- User downloads the file Claude exports
- User pushes via GitHub web upload (manual)
- Claude doesn't push; user is the final commit author

### File handoff caveat
When the user downloads a file with the same name multiple times in a session, the browser renames them `index (1).html`, `index (2).html`, etc. The user has to know which is the latest and rename appropriately before uploading to GitHub. **When generating multiple iterations in a session, mention which file is the final one to push.**

### Recap files at start of new chat
Fetch these raw URLs to load context:
- `https://raw.githubusercontent.com/defipatriot/website-adao-core/main/PROJECT_KNOWLEDGE.md`
- `https://raw.githubusercontent.com/defipatriot/website-adao-core/main/CHANGES_PENDING.md`
- `https://raw.githubusercontent.com/defipatriot/website-adao-core/main/index-log.md`

---

## Cold-start recovery checklist

If a future Claude session has zero memory and the user is in trouble, here's the minimum to get back up to speed:

1. **Fetch all three doc files** (URLs above)
2. **Clone the repo:** `git clone https://github.com/defipatriot/aDAO-links-site /home/claude/aDAO-links-site`
3. **Read the latest changelog** to see what was last shipped
4. **Read CHANGES_PENDING** to see what's queued
5. **Confirm with user** what they want to work on, then proceed

The docs are the source of truth. If the docs disagree with the code, trust the code and update the docs. If the docs disagree with the user, trust the user and update the docs.

---

## Mental model: what kind of work happens here

This is an **iterative, screenshot-driven UI project**. Pattern:
1. User pushes to GitHub
2. User screenshots what's wrong on mobile/desktop
3. Claude diagnoses + edits
4. User pushes the new version
5. Repeat

Don't over-engineer. Don't add CSS that wasn't asked for. **When user reports something broken, find the actual cause** (often a CSS specificity conflict from layered prior fixes) rather than adding more `!important` overrides on top.

---

## Questions Claude should ask itself before responding

- **Is there context I should have but might be missing?** Search the transcript or past chats first.
- **Is there a design principle that applies here?** Check section above.
- **Will this change affect installed PWAs?** Check PWA update rules.
- **Will this change need to propagate to other pages?** Check Current pages table.
- **Is this something I should add to the docs?** Check Tracking responsibilities at top of file.
- **Does this overlap with a Pending item?** Check `CHANGES_PENDING.md`.
