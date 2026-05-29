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
| Admin-tool bug found or fixed (data-capture, snapshot tools) | "Critical data-capture gotcha" section | Prevent regression; admin tools don't have a changelog footer so this doc is the only record |

### When to update CHANGES_PENDING.md

| Trigger | Add it to | Why |
|---|---|---|
| User mentions a feature in passing as "we should do this someday" | "Future projects" section | Don't lose ideas |
| We notice tech debt while working on something else | "Low priority / cleanup" section | Track without derailing current work |
| User asks an open question we don't answer immediately | "Open questions" section | Resume the discussion later |
| A bug found but not fixed in current session | "Active / next round" with priority | Visible queue |
| External task identified (e.g. "submit to Search Console") | Top of "Active / next round" if blocking | Visible queue |

### Rev numbers — single source of truth

Rev numbers live in 3 places and they MUST stay in sync:

1. **The "Current pages" table in this file** — canonical at-a-glance reference. Future Claude reads this on startup. **This is the source of truth.**
2. **Page footer** (`Rev X.Y · Changelog` in the HTML) — what users see
3. **`<page>-log.md` in `website-adao-core`** — full revision history

Whenever Claude bumps a rev on any page, all three update in the same push:

| Trigger | Action |
|---|---|
| User-visible change to any page is shipped | Bump rev in HTML footer, prepend new entry to that page's log file, AND update this table |
| Doc-only changes (this file, CHANGES_PENDING.md) | Do NOT bump any rev |
| New page added to the site | Add row to the table with starting rev (usually 1.0) and create its log file |

If a page doesn't have a rev/changelog footer yet (the cross-page rollout is still in progress), it shows `—` in the rev column. Bump it the first time we touch that page after adding the chrome.

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
- **TLA pool positions are dual-typed.** Each pool position consists of a **non-amplified deposit** (white in TLA UI) AND an **amplified deposit** (orange). The DAO holds both side-by-side at roughly 50/50 because:
  - Non-amplified earns rewards in claimable form, but is slowly eaten by the 10% TLA take-rate
  - Amplified (ampLP token) compounds in place via Eris Amp Compounder — outpaces the take-rate
  - Together: claimable rewards stream from non-amp side + principal preservation from amp side
- **TLA success metric: ampLP token count growth per epoch.** Per the DAO's strategy, the goal is each epoch to gain ampLP tokens not lose them. USD value can fluctuate freely with market conditions; what matters is the underlying ampLP count is increasing. This is the right thing to surface on dashboards rather than a USD figure.

### Lore framework
10 post-human tribes scattered across the galaxy via "The Lattice". Designed to be flexible for absorbing partner DAOs:
- **Lion DAO** → became "Canyon-Clans of Ozara North" (precedent for how partnerships fit lore)

### Storage / data repos

#### Legacy / manual-snapshot repos (still in use)
- `defipatriot/tla_json_storage` — Holds `tla_config.json` (site scoring weights) and `epoch_1-300_date.json` (canonical epoch schedule — **the 1-indexed source of truth** for epoch numbers). Older per-epoch v3 JSONs from manual `tla_tool.html` captures also live here but are no longer actively updated.
- `defipatriot/tla-ext_json_storage` — Historical TLA data from `tla-tool_ext.html`. Contains `tla_ext_historical_2025.json` and `tla_ext_historical_2026.json` (per-LP epoch averages for liquidity + volume, used by trend charts), plus `tla_pd_bribes.json` (master PD bribes file).
- `defipatriot/adao_json_storage` — Legacy aDAO snapshots from manual captures. Members CSV fallback lives here.
- `defipatriot/aDAO-Image-Files` — favicons, logos, OG images, collection PFPs.
- `defipatriot/website-adao-core` — project docs + changelogs (this repo).

#### Cron-produced data repos (active 2026 — written automatically by Render crons)
**9 production crons live and writing on schedule.** Cron source code lives in `defipatriot/cron-scripts` (one folder per cron). Each writes to its own `*-data_2026` data repo. Status verified 2026-05-17:

| Data repo | Source cron | Schedule | Main output file |
|---|---|---|---|
| `defipatriot/tla-snapshot-data_2026` | `tla-snapshot` | Hourly :40 (+ daily archive at 23:xx) | `data/tla-snapshot.json` + `data/daily/{YYYY-MM-DD}.json` archive — the unified per-epoch snapshot consumed by `tla-stats.html` |
| `defipatriot/network-and-prices-data_2026` | `network-and-prices` | Hourly :40 | `data/network-and-prices.json` — LUNA + 27 token prices, LST ratios. Known gap: some IBC denoms (e.g. LUNA-USDC bribe `ibc/8D8A7F...`) not indexed. |
| `defipatriot/adao-positions-data_2026` | `adao-positions` | **Should be daily 01:00** (currently weekly Mon — needs Render schedule update for Portfolio Tracker history; see CHANGES_PENDING) | `data/current.json` + `data/daily/{YYYY-MM-DD}.json` (added 2026-05-17) + `data/weekly/epoch-{N}.json` archive |
| `defipatriot/astroport-pool-data_2026` | `astroport` | Daily 23:50 | `astroport/astroport-epoch-{N}.json` + `data/daily/{YYYY-MM-DD}.csv` (20 columns as of 2026-05-17, includes fees/reserves/LP-supply/staked-liquidity/assets_json for LP health scoring) |
| `defipatriot/ss-pool-data_2026` | `skeletonswap-lp_data` | Daily 23:45 ⚠ | `data/{month}_backup/{YYYY-MM-DD}.csv` + weekly avg. **Upstream source unreliable** — BackBone aggregator returning cached data for ~30 days as of 2026-05-17. Don't use for scoring; see cron-scripts/skeletonswap-lp_data/README.md "Data quality warning" for full audit. |
| `defipatriot/bribes-data_2026` | `bribes-history` | Daily 23:35 | `data/current-state.json` + `data/by-epoch/epoch-{N}.json` + `data/pd-bribes-history.json` |
| `defipatriot/votion-data_2026` | `votion` | Weekly Sun 23:55 | `votion/votion-epoch-{N}.json` (next-epoch optimization) |
| `defipatriot/nft-inventory-data_2026` | `nft-inventory` | Hourly :30 | `nfts.json` (full 10K inventory) + `summary.json` (minted/unminted/broken counts) |
| `defipatriot/marketplace-data_2026` | `marketplace-stats` | Hourly :15 | BBL + Boost listings, floor prices, sales history (per-year files), activity feed |

#### Notes on data repo numbering quirks
- **Votion uses NEXT-epoch convention** — `votion-epoch-185.json` is captured during epoch 184, contains optimization data FOR upcoming epoch 185. This matches Eris's Votion UI convention.
- **All other epoch-named cron outputs use CURRENT-epoch convention** — `astroport-epoch-185.json` was captured during epoch 185.
- **Epoch off-by-one fix shipped** (resolved 2026-05-15). All crons now compute `epochIndex + 1` to match the canonical 1-indexed `epoch_1-300_date.json`. Verified live: today (2026-05-17) the crons correctly report epoch 185. Note: epoch-184 archive files exist but no epoch-185 archive files until the next nightly run — one-epoch gap accepted, not back-filled.

### Key on-chain contract addresses (Terra phoenix-1)
Originally discovered May 10 2026 via HAR capture of the Eris liquidity-hub UI for the on-chain Vote tab fetcher. Now used by the live `tla-snapshot`, `adao-positions`, and `bribes-history` crons.

| Role | Address |
|---|---|
| TLA Gauge Controller | `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj` |
| TLA Incentive Manager (bribes) | `terra1tuuwm8yrj54qeg0c8xu00aha9ryatyhtczq8qq2q8tntuw0auzas9037wh` |
| vAMP Minter (total VP) | `terra1uqhj8agyeaz8fu6mdggfuwr3lp32jlrx5hqag4jxexde92rzkamq3l62zg` |
| LUNA-FUEL Astroport pool | `terra10yfnsqn20rzlnlzkeva5255q27zp6ws9te9uuql9e0lacfcze7zsffjct5` (used by `fetchFuelPriceFromChain`) |
| Votion (arbLUNA Max) | `terra13aae4futz6jk7hmdv0gwm2xs6p4nxv4xwz5tc0c2vt4960u4j6jqpqmye9` (only one captured so far; others unknown) |
| aDAO Core (treasury) | `terra1sffd4efk2jpdt894r04qwmtjqrrjfc52tmj6vkzjxqhd8qqu2drs3m5vzm` |
| ADAO Voting Contract | `terra1c57ur376szdv8rtes6sa9nst4k536dynunksu8tx5zu4z5u3am6qmvqx47` |
| TLA Asset Compounder | `terra1zly98gvcec54m3caxlqexce7rus6rzgplz7eketsdz7nh750h2rqvu8uzx` |
| Global Config | `terra1hwxg6s732eparz3ys7sa4t5f64ngpd2w8syrca6z7ckv3fs9uqnsvrpcqa` |
| PD DAO multisig (briber) | `terra1k8ug6dkzntczfzn76wsh24tdjmx944yj6mk063wum7n20cwd7lxq4lppjg` |
| zLUNA Hub | `terra1u72y7gppxrsncctvgfyqduv3md6pgq77pqhz9rxgwl3dqgye00cq7vmf8u` |

---

## TLA cron infrastructure — built 2026-05-12 through 2026-05-14, audited 2026-05-17

Replaces the manual `tla_tool.html` + `tla-tool_ext.html` capture flow. **9 production crons run on Render**, write to GitHub data repos automatically. The (now-deleted) `DESIGN_tla_full_cron_automation.md` design doc described the original plan; this is the as-built record.

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│ 5 TLA PRODUCER CRONS — fetch raw data, write per-cron _2026 repos│
├──────────────────────────────────────────────────────────────────┤
│ network-and-prices    hourly :40   token prices + LST ratios     │
│ astroport             daily 23:50  pool TVL/volume/fees/reserves │
│                                    (20-col CSV as of 2026-05-17) │
│ skeletonswap-lp_data  daily 23:45  SS pool data ⚠ source frozen  │
│ bribes-history        daily 23:35  current bribes + epoch archives│
│ votion                weekly Sun   next-epoch optimization data  │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 1 AGGREGATOR CRON — consumes all 5 producers + chain queries     │
├──────────────────────────────────────────────────────────────────┤
│ tla-snapshot          hourly :40   unified snapshot for website  │
│                       (+ daily archive at 23:xx)                 │
│   - Reads from all 5 producer _2026 repos                        │
│   - Adds rewards math (Alliance weights, APR per pool)           │
│   - Outputs data/tla-snapshot.json (170 KB, 67 pools)            │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 1 INDEPENDENT CRON — DAO member portfolios                       │
├──────────────────────────────────────────────────────────────────┤
│ adao-positions   currently weekly Mon ⚠ should be daily          │
│                  → daily archive at data/daily/{YYYY-MM-DD}.json │
│                  → 46 member + treasury portfolios               │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ 2 NFT-SIDE CRONS — separate from TLA, support the NFT pages      │
├──────────────────────────────────────────────────────────────────┤
│ nft-inventory     hourly :30   10K NFTs, mint/broken/DAODAO state│
│ marketplace-stats hourly :15   BBL+Boost listings, sales, floor  │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ WEBSITE consumes via raw.githubusercontent.com (no API server)   │
├──────────────────────────────────────────────────────────────────┤
│ tla-stats.html ← tla-snapshot.json + network-and-prices.json +   │
│                  adao-positions/current.json + historical files  │
│ dao-tla.html   ← adao-positions/current.json members array        │
│                  (page not yet built — Pass 2 → likely promoted  │
│                  to header-level Portfolio Tracker per 2026-05-17│
│                  strategic discussion)                            │
└──────────────────────────────────────────────────────────────────┘
```

### Design principles for this infrastructure
- **Independent systems** — one cron failing doesn't break the dashboard. Each writes to its own repo so partial-data renders are possible.
- **Same filenames on each run** — `current.json`, `tla-snapshot.json`, etc. always overwrite. Epoch archives accumulate alongside.
- **Render free tier** — cron jobs run as Node.js services, push to GitHub via PAT.
- **No API server in front** — website fetches data via `raw.githubusercontent.com` direct URLs.
- **Adao-positions treasury = "aDAO" at TLA-wide level.** Individual member breakdowns live in a separate page (`dao-tla.html`, not yet built). The treasury wallet (`terra1sffd4ef...`) is the single canonical aDAO voting entity. Each user's VP allocates once per bucket (4 buckets), so pool-summed VP inflates 4× per user — use max-bucket VP (~24M) as canonical "Total TLA VP" to match Eris.

### Pool uniqueness — critical for any code reading TLA data
- `pool_name` alone collides across DEXes (LUNA-USDC on both Astroport and Skeleton)
- `name + dex` can also collide within Astroport (two LUNA-WBTC pools in BLUECHIP with different `gauge_pool_id`)
- **`gauge_pool_id` is the truly unique key** (e.g. `cw20:terra1wdz...`)
- Snapshot exposes it as `gauge_pool_id`; member-vote data exposes it as `pool_gauge_id` — same values, different field names
- Any code keying on pool name alone will eventually hit collisions. Always key on `gauge_pool_id`.

### Key value reconciliation (verified 2026-05-14)
| Metric | Value | Cross-check |
|---|---|---|
| Total TLA VP | 24.11M | matches Eris UI |
| Votion VP | 6.90M | matches votion.money lockup data exactly |
| aDAO VP (treasury) | 757K | matches Eris UI |
| Treasury LP USD | $6,669 | matches Eris ±$103 |
| Treasury pending rewards | $453.38 | matches Eris ±$11 |
| Treasury pending bribes | $443.13 | matches Eris ±$23 |

### Audit findings — 2026-05-17

Full per-cron audit performed. Most data is reliable. Two real issues found and addressed:

| Finding | Severity | Resolution |
|---|---|---|
| Bribes resolver bug in `tla-stats.html` — looked up cw20 prices at `entry.address`; actual schema nests at `entry.prices.{source}.address`. All CAPA, ROAR bribes priced as $0. | 🔴 Critical | **Fixed** in dashboard. Global Epoch Bribes tile: $820 → ~$1,300 (more accurate). Member bribes tile correctly captures CAPA bribes. |
| `adao-positions` had no daily snapshot persistence (only weekly per-epoch). | 🔴 Critical | **Fixed** in cron code (`data/daily/{YYYY-MM-DD}.json` archive added). Render schedule must also change from weekly to daily for Portfolio Tracker history to accumulate. |
| `astroport` cron missing fee/reserves/LP-supply fields. | 🟡 Important | **Fixed** in cron. CSV is now 20 columns including `fees_24h_usd`, `fee_apr`, `lp_total_supply`, `astro_staked_usd`, `assets_json`. |
| Skeleton Swap upstream API frozen ~30 days (BackBone aggregator returning cached data). 50% effective coverage, not 75%. | 🔴 Critical | **Documented** (skeletonswap-lp_data/README.md "Data quality warning"). Keep capturing best-effort. **Don't use for scoring.** Label "unverified" wherever surfaced. |
| APR for stable pairs (USDC-USDT, USDC-EURe) 5× too high vs Eris. | 🟡 Real bug | Undiagnosed. Specific to stable pools. Tracked in CHANGES_PENDING. |
| Unnamed pool with `dex: null` inflates Astroport count by 1. | 🟢 Cosmetic | Tracked in CHANGES_PENDING. |
| LUNA-arbLUNA appears twice in snapshot with different `gauge_pool_id`. | 🟢 Cosmetic | Dashboard now keys on `gauge_pool_id` correctly. |

**Methodology vs bug:** the cron's pool APR uses `annual_emissions_usd / staked_in_tla_usd` while Eris uses `annual_emissions_usd / depth_usd`. Our denominator is smaller → our APR reads higher. Both correct, measuring different things. To match Eris exactly, change the formula in `tla-snapshot` cron — one-line change. Tracked in CHANGES_PENDING.

### Cron deployment
- All code in `defipatriot/cron-scripts` GitHub repo
- Each cron has its own subdirectory deployed as a separate Render cron job
- Render builds via `npm install` in the cron's folder, runs via `npm run snapshot` (or equivalent)
- Required env vars per cron: `GITHUB_TOKEN`, `GITHUB_REPO`, `GITHUB_BRANCH`
- Tokens have write scope to that cron's data repo only

### Per-cron documentation
Each cron has its own `README.md` in its folder. Recent changes are tracked in a "Recent changes" section at the end of each cron's README. The top-level `cron-scripts/README.md` has a "Project status & roadmap" section covering cross-cutting context (strategic direction, data trust, prioritized roadmap). The dashboard (`tla-stats.html`) has its own changelog in `website-adao-core/tla-log.md` (following the per-page log convention: `index-log.md`, `tla-log.md`, `lore-log.md`, `explorer-log.md`, `dao-log.md`).

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

| Display name | File | Current rev | Notes |
|---|---|---|---|
| (Home) | `index.html` | 3.48 | The main dashboard, ~12.6k lines. Has the changelog system. Live RPC tile architecture matured 3.38–3.48. |
| NFT Explorer | `nft-explorer-index.html` | 4.13 | Top nav tab. ✅ Cross-page chrome added in Rev 3.22. Map view removed in Rev 4.13. |
| aDAO Lore | `adao-lore.html` | 2.9 | Top nav tab. ✅ Renamed from `planet-map.html` in Rev 3.22. ✅ Cross-page chrome added. |
| TLA Stats | `tla-stats.html` | 2.1 | Top nav tab. ✅ Rebuilt 2026-05-14 to consume continuous cron data sources. Member Data overlay + bribes resolver fix in Rev 2.1 (2026-05-17). |
| DAO | `dao.html` | 1.6 | Top nav tab. ✅ Renamed from `dao_governance.html` in Rev 3.22. ✅ Cross-page chrome added. |
| ALLY Rewards | `ally.html` | 3.4 | Top info-card tile. ✅ Cross-page chrome added in Rev 3.24. Duplicate header cleaned in Rev 3.25. |
| Tutorials | `tutorials.html` | 1.5 | Top info-card tile. ✅ Cross-page chrome added in Rev 3.24. Duplicate header cleaned in Rev 3.25. |
| Tools | `tools.html` | 1.4 | Top info-card tile (hub for fuel-tool, ampcapa-tool, tla_tool). ✅ Cross-page chrome added in Rev 3.24. Old in-page nav removed in Rev 3.25. |
| Rarity Info | `rarity-explained.html` | 1.3 | Top info-card tile. ✅ Cross-page chrome added in Rev 3.24. Duplicate header cleaned in Rev 3.25. |
| NFT Releases | `release-history.html` | 1.4 | Top info-card tile. ✅ Cross-page chrome added in Rev 3.24. Duplicate header cleaned in Rev 3.25. |
| Official Links | `links.html` | 1.4 | Top info-card tile. ✅ Cross-page chrome added in Rev 3.24. Duplicate header cleaned in Rev 3.25. |
| Alliances | `alliances.html` | 1.4 | Top info-card tile. ✅ Cross-page chrome added in Rev 3.24. Duplicate header cleaned in Rev 3.25. |
| DAO TLA Deposits | `dao_tla_deposits.html` | 1.2 | Linked from DAO Links dropdown tile. ✅ Cross-page chrome added in Rev 3.24. Rev 1.1 added live overlay via `aDAOLive` library (2026-05-28). Rev 1.2 fixed compare anchor + chart extends to live. |
| DAO Treasury | `dao_treasury.html` | 3.1 | Linked from DAO Links dropdown tile. ✅ Cross-page chrome added in Rev 3.24. Rev 3.0 (2026-05-28) — full unified-view rebuild (removed 4-tab structure, added token pills + Amount/USD toggle). Rev 3.1 (2026-05-29) — 3-bucket "What Changed" framework (Market / Organic / DAO Actions). |
| Fuel Tool | `fuel-tool.html` | 1.3 | Linked from Tools page. ✅ Renamed from `fuel_tracker.html` in Rev 3.22. ✅ Cross-page chrome added in Rev 3.24. |
| ampCapa Tool | `ampcapa-tool.html` | 1.3 | Linked from Tools page. ✅ Renamed from `capa_lp_converter.html` in Rev 3.22. ✅ Cross-page chrome added in Rev 3.24. |
| TLA Docs | `tla-docs.html` | 1.3 | Linked from TLA Stats. ✅ Cross-page chrome added in Rev 3.24. Title block cleaned in Rev 3.25. |
| _Admin: TLA Tool_ | `tla_tool.html` | — | Internal admin (manual TLA snapshots). Favicon + analytics only — chrome intentionally skipped (admin context). |
| _Admin: TLA Tool Ext_ | `tla-tool_ext.html` | — | Internal admin extension. Favicon + analytics only — chrome intentionally skipped. |
| _Admin: DAO Gov Tool_ | `dao_governance_tool.html` | — | Internal governance audit tool. Favicon + analytics only — chrome intentionally skipped. |
| _Planned: Member Stats_ | `dao-tla.html` | — | **NOT YET BUILT.** Pass 2 of the TLA infrastructure rebuild. Per-member portfolio panels (locks, votes, rewards, bribes claim status) for the 46 named DAO members. Data already collected in `adao-positions/current.json`. Link from `tla-stats.html` tab strip already in place. |

### Admin / dev pages (not in user-facing changelog rollout)
| File | Purpose |
|---|---|
| `tla_tool.html` | TLA admin tool (data collection) |
| `tla-tool_ext.html` | TLA extensions tool |
| `dao_governance_tool.html` | DAO governance tool |

**Rev-bump convention for admin tools:** these intentionally have NO `Rev / Changelog` footer chrome (the public 5-tab nav would be misleading context). When fixing bugs in admin tools, **don't bump a rev** — there's nothing to bump. Document the change in `CHANGES_PENDING.md` (under a "Recent admin-tool fixes" section if needed) and in this doc's "Critical data-capture gotcha" section if the bug class is recurrent. The `index-log.md` is for user-visible changes only.

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

### Critical data-capture gotcha — Vote tab parser hardcoded DEXes/tokens (May 9 2026)
- `parseVoteTab()` (the Vote-tab paste parser that builds `store.lpRegistry`) had hardcoded DEX names `{Astroport, Skeleton Swap, WhiteWhale}` and a hardcoded list of single-sided tokens `['ampCAPA', 'xASTRO']`. When TLA added a new DEX (Creda, hosting `wBTC.creda.a` as a single-sided pool in May 2026), every Creda entry was silently dropped from the registry — and all the downstream consequences (no ampLP balance lookup, no asset metadata entry, missing from snapshot exports) flowed from that single deletion.
- **Fix shipped May 9 2026:** rewrote `parseVoteTab` to detect entries by SHAPE rather than by name lookup. A pool entry is anything that's:
  1. A line that passes `looksLikePoolName()` (has letters, isn't a reserved UI label, isn't a numeric/value line)
  2. Followed within 6 lines by a `VP <amount>  <pct>%` line
  3. With a DEX name (any capitalized word that isn't reserved) somewhere between them
- Verified against the user's epoch-184 paste: captures `wBTC.creda.a|Creda` correctly. Adapts automatically to new DEXes and new single-sided tokens going forward — no code change required.
- **General principle:** when a parser fails because the input has new entries it doesn't know about, the right fix is to make the parser describe *what entries look like* (structurally) rather than enumerate *which entries are allowed* (by name). The enumerated approach guarantees future regressions.

### Critical data-capture gotcha — Astroport D90 dateRange deprecated (May 9 2026)
**Root cause finally identified.** Astroport's TRPC charts endpoint silently dropped `"D90"` as a valid `dateRange` enum value sometime before May 2026. It now returns a Zod validation error: `{"received": "D90", "code": "invalid_enum_value"}`. The endpoint itself works fine — `D7` and `D30` still return real time-series data for active pool addresses.

**Why it manifested as "all-zero exports":**
1. `runFullAstroWorkflow()` fetches all three ranges in sequence: D7 → D30 → D90, saving each into `astroMultiRangeData.{D7,D30,D90}`.
2. D7 and D30 capture real data (~42-45 points per pool for active pools).
3. D90 fetches all error out — but the per-pool entries still get written to `astroMultiRangeData.D90` with `epochs: {}` and `latestLiquidity: 0`.
4. The export builder used `astroMultiRangeData.D90 || astroAutoData.epochData` — `||` doesn't fall through because D90 is a populated (zero-filled) object.
5. Every export read D90 → all zeros, even though D7+D30 had everything.

**Fix shipped May 9 2026 (tla-tool_ext.html):** Added `selectBestAstroRange()` helper that walks D90 → D30 → D7 and returns the first range with at least one entry having non-zero `latestLiquidity` or non-empty `epochs`. Five call sites updated. Astroport error truncation increased from 80 to 200 chars so the schema-validation message is readable next time something else changes upstream.

**Pool-not-found errors are EXPECTED** for deprecated duplicate pool addresses (e.g. `terra16a45v4...` for LUNA-USDC). The dedup logic correctly flags these. They count as "failed" in the new error reporting, but functionally aren't a problem — the active pool address gets the data. Could improve the categorization to suppress these from the failures dialog later.

### Critical data-capture gotcha — Votion API HTTP 500 is INTERMITTENT, not broken
- Eris's Votion backend (`backend.erisprotocol.com/votion/.../optimization`) returns HTTP 500 from origin on a meaningful fraction of calls — but NOT all of them. The page itself succeeds because it retries automatically.
- **Confirmed via HAR capture (May 10 2026):** the same `/optimization` URL that failed multiple times during our captures returned HTTP 200 with full data when the actual app loaded. Same URL, same backend, same data shape we already know how to parse. Not a new architecture, not a different host.
- The `phoenix-rpc.erisprotocol.com` requests we saw in the page's network log alongside `/optimization` are *additional* contract reads for the live `staked` amount shown at the top — not a replacement for the optimization endpoint.
- **Fix shipped May 10 2026:** Added retry-with-backoff (3 attempts, 0/2s/4s waits), per-run failed-proxy memory (don't re-test 4xx-failing proxies between lockups in the same run), 500ms stagger between lockups (Eris's backend handles serial calls better than rapid burst). Inline progress shows which lockup is being fetched + retry status. 5xx errors trigger retries; 4xx errors blacklist the proxy for the run (since it's the proxy at fault, not the origin).
- **Smart proxy ordering:** the cached `workingProxy` (last successful one) is tried first, then untested proxies, then last-known-failed proxies as a last-resort fallback. So in a 6-lockup run, after the first lockup figures out which proxy works, the other 5 use that proxy directly without re-testing the broken ones.

### Critical data-capture gotcha — `epoch: "unknown"` in workflow exports
- The May 9 2026 Astroport and Skeleton workflow exports both have `meta.epoch: "unknown"` and per-pool `currentEpoch.epoch: "unknown"` despite `live-epoch-info` resolving correctly at page load. Skeleton even picked old hardcoded epochs (168, 169) for its 4-epoch averaging instead of the actual current epoch (~184).
- **Where to look first:** the export-builder code that writes the JSON should be reading `store.liveEpochInfo?.currentEpoch` like the rest of the tool does. If it's reading a stale or non-populated field, the export gets `"unknown"`.
- **Until fixed:** rename downloaded files manually before push (`astroport-epoch-184-2026-05-09.json` not `epoch-unknown`), and the Skeleton 4-epoch-avg numbers should be considered suspect for any epoch where `historicalEpochsLoaded` doesn't include the actual current epoch.

### Critical data-capture gotcha — Silent coercion hides cron query failures (May 26 2026)

**Root cause of "adao-positions cron dropped 6 of DAO's 16 LP positions for weeks before anyone noticed."**

The cron's `queryContract` function returned `null` on transient LCD failures (rate-limiting from publicnode under 15-way concurrent load = ~255 in-flight queries per batch). Downstream code did:

```js
.then(r => ({ bucket, entries: Array.isArray(r) ? r : [] }))
```

`Array.isArray(null) === false`, so `null` got silently coerced to `[]`. No error recorded, no `_errors` push, nothing logged. Output looked legitimate — just had fewer positions than reality. For the DAO this meant the entire bluechip bucket (3 amp positions) plus 3 of 6 project amp positions vanished from cron output every single run.

Detection took ~3 weeks because:
- The cron output was still being written successfully (just incomplete)
- The dashboard tile read cron data and showed "$6,340" — a plausible value
- No one noticed until cross-referencing against Eris UI's $9,751 for the same wallet

**Pattern to recognize**: any expression of the form `Array.isArray(r) ? r : []` or `r || []` where `r` came from a network call. These collapse "query failed" into "no data" without distinguishing the two.

**Fix (adao-positions v1.3.0, shipped Rev 3.45):**
1. **Retry with backoff** in `queryContract` — 2× primary attempts with 200-500 ms jittered backoff, then fallback endpoint, then `null`
2. **Distinguish null from empty** — `entries: null` (failed) vs `entries: []` (genuinely empty), propagate as different states
3. **Surface to `_errors`** — failures push to `portfolio._errors` so they appear in output
4. **Lower concurrency** — `BATCH_CONCURRENCY` from 15 → 5. 5-way × 17 queries/member = ~85 concurrent peak, well within publicnode tolerance

**General principle for crons:** any silent coercion of a network result is a latent bug. Always preserve the distinction between "we asked and got nothing back" and "we asked and the answer was empty." Push failures to a structured error list in the output so partial-data states are detectable downstream.

### TLA pool valuation paths (matches cron + library logic)

When valuing a DAO position against a pool from `tla-snapshot.json`, the resolver path depends on the position's asset shape:

- **Cw20 LP tokens** → `poolByLpAddr[cw20.toLowerCase()]`
- **Native / factory tokens** → `poolByGaugeId["native:" + denom]`
- **Single-asset pools** (ampCAPA, xASTRO, ampROAR-ROAR) — have `lp_address: null`:
  1. Try `tokenPrices[poolName].final_price_usd` from `network-and-prices.json` (most accurate)
  2. Fallback to `lp_health.asset_0.price_usd` from snapshot
  3. Last resort: compounder share fallback `staked_in_tla_usd × (userLp / totalLp)`

Schema gotchas to remember:
- `asset_configs[].gauge` IS the bucket name (`'single'`, `'stable'`, `'project'`, `'bluechip'`) — not a contract address
- `user_claimable` (bribe manager) returns `{ start, end, buckets }` — NOT an array. Iterating directly throws.
- `treasury.lp_positions[].estimated_position_usd` is the correct USD field (NOT `usd_value` at root)
- `pool.gauge_pool_id` and member-vote `pool_gauge_id` carry the same values — different field names
- `treasury.wallet_balances[].symbol` is matched on `/zluna/i` to find unredeemed reward tokens (any zluna-suffixed factory denom counts)

### Treasury "What Changed" 3-bucket framework

Documented in `dao_treasury.html` Rev 3.1 (2026-05-29). When showing how the treasury value changed over a window:

```
Net Change = Market Movement + Organic Growth + DAO Actions
```

- **Market Movement** — price impact on starting holdings (sum of `oldAmount × (newPrice − oldPrice)`)
- **Organic Growth** — yield, rewards, slippage (`positionChange − propActionsUsd`)
- **DAO Actions** — sum of executed props' actual treasury impact (`Σ calcPropImpactUsd(prop.netByToken)` for props in the window)

Without splitting DAO Actions out, a "Deposit into TLA" prop deploying LUNA looks identical to losing money. The 3-bucket attribution makes intent legible. `isReceiptToken` filters zLUNA / amp-LP / LP shares from prop impact valuation so a deposit shows only the LUNA outflow, not the offsetting amp-LP token inflow.

Implementation in `dao_treasury.html`: `getActualTreasuryImpact(prop)` returns per-token net flows; `calcPropImpactUsd(netByToken)` values them in current USD. Default compare window = most recent snapshot → Live.

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

### Shared live-data library (`lib/adao-live-data.js`)

Introduced in Rev 3.47 (2026-05-28). Single source of truth for live TLA / treasury / unclaimed-rewards fetching across the dashboard suite. Exposes `window.aDAOLive` with composite getters that follow **live RPC primary → cron fallback → snapshot last-resort** for each metric.

**Why this exists:** the same fetch logic was duplicated across `index.html`, `dao_tla_deposits.html`, and `dao_treasury.html` — including the cron-vs-live-vs-snapshot resolution, the LP valuation paths (cw20 vs native vs single-asset), and the bank balance enumeration. Three copies that drifted out of sync. Library consolidates them.

**Public API:**
- `aDAOLive.getDaoTlaDeposits()` → composite LP positions + rewards + bribes + zLUNA
- `aDAOLive.getDaoTreasury()` → wallet balances priced via CoinGecko
- `aDAOLive.getDaoUnclaimedRewards()` → deposit + rebase + vote rewards
- `aDAOLive.getTlaCatalog()` → pools, amp configs, token prices
- `aDAOLive.queryChain(addr, msg)`, `aDAOLive.bankBalances(wallet)`, `aDAOLive.cw20Balance(...)` — primitives for one-off needs
- `aDAOLive.getCron(name)` — raw cron data passthrough when needed
- `aDAOLive.clearCache(pattern?)` — selective cache invalidation
- Constants: `DAO_MAIN_WALLET`, `TLA_STAKING_BY_BUCKET`, `TLA_ASSET_COMPOUNDER`, `ZLUNA_CONNECTORS`, `denomMap`

**Caching:** layered in-memory `Map` + `sessionStorage` (keyed `adao_live:`). Survives navigation between pages within a tab. TTL 5 min for live LP capture + catalog, 1 min for unclaimed rewards. Second page load is effectively instant.

**Architecture rule:** dashboard tiles should be LIVE feeds (RPC primary), cron is fallback + historical capture only. Never tie tile freshness to cron's hourly cadence. New tile work goes through `aDAOLive`. The cron is correctly positioned as historical capture + resilience, not the data source for current state.

**Migration status:** `index.html` and `dao_treasury.html` still have inline live-data code that should migrate to `aDAOLive` incrementally. New work uses the library; legacy paths get migrated when touched. Tracked in `CHANGES_PENDING.md`.

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
- `https://raw.githubusercontent.com/defipatriot/website-adao-core/main/tla-log.md` (if working on TLA stats)

If working on cron-side code, also pull:
- The relevant cron's source from `defipatriot/cron-scripts` (e.g. `tla-snapshot/tla-snapshot.js`)
- The latest output JSON for that cron to verify schema (e.g. `tla-snapshot-data_2026/main/data/tla-snapshot.json`)

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
