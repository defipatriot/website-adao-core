# TLA Stats Changelog

This is the change history for `tla-stats.html` (the Terra Liquidity Alliance public dashboard).
Newest revisions on top. Times are UTC.

---

## Rev 2.2 — 2026-05-29

Pool health, capital-flow, and a full member mode built on top of the Rev 2.1 member overlay. All additions live in the rendering layer; the cron data layer is untouched except for two new history rollups that ride along with the existing `tla-snapshot` cron (see `cron-scripts/` and the new data files below).

### What changed

#### Pool Health & Capital Flow panel (new)
A watchlist of TLA's largest exposures, ranked by staked capital. Each pool shows a 4-epoch sparkline of its TLA stake, dollar flow this epoch, a health dot, and in/out/net summary cards. Comparisons are made within each pool's own `pool_address` series (never by name) so old/new pool migrations don't create phantom drops. Alarms are market-normalized — a pool is only flagged when it's draining materially faster than its bucket's median, so a broad market dip doesn't trip false alarms. Tiers: combined exit signal (depth + reserves + price all down hard), draining faster than peers, and sustained bleed (down every epoch and at least 15% cumulative). Reserve skew is shown only when a pair is meaningfully off 50/50.

#### Member mode (new) — the panels become personal when you pick a wallet
Selecting a member in the header now transforms two panels into a personal view, and reverts cleanly when deselected:

- **Pool Health becomes "Your positions & flow"**: your LP positions ranked by your own capital. Each row splits two honest signals side by side:
  - **Stake** — the change in your *real* LP units (vault/ampLP shares, falling back to LP tokens or underlying), which is the true deposit/withdraw signal. Auto-compounding and deposits grow it; only a genuine withdrawal shrinks it.
  - **Value** — the USD change, which also moves with token price. Price-driven moves are tagged `(price)` so a falling token can never look like you pulling capital.
  - Summary cards: **Value change** (USD, incl. price), **Stake added** (deposits + compounding), and **Stake reduced** (actual withdrawals only).
- **Threshold Watch becomes "Your at-risk pools"**: driven by your actual positions (`status` + `distance_from_threshold_pp`), not your votes — your pools that are near the 1% line, dropped this epoch, or already inactive.

Members holding the same pool under multiple stake configs are aggregated into one row. Member flow is epoch-over-epoch against the prior weekly archive; when that archive is missing, positions still list and the flow baseline is marked unavailable.

#### Threshold Watch rework
Rebuilt to be history-driven (keyed `name|bucket`): active danger-band pools (1–2% of bucket) most-at-risk first with an epoch-over-epoch trend, pools dropped this epoch, and an expandable list of drops over the last 4 epochs.

#### Leaderboards & APR history
Leaderboards now use true 4-epoch rolling averages with rank-movement badges and percentage deltas. New APR history (per-epoch rollup feeding a page consumer) adds an APR movement badge versus the last completed epoch.

#### Fixes
- **Single-asset pools** (ampCAPA, xASTRO) were mislabeled "Skeleton" in the Vote Breakdown waterfall — the dex sub-label was a binary `Astro` / `Skeleton`, so anything not Astroport fell through to Skeleton. They now correctly read "Single." (ampROAR-ROAR is a genuine Skeleton Swap pair and is unchanged.)
- Removed the orphaned "snapshot missed" popup.
- Removed the Skeleton Swap amber data-limitation banner.
- Fixed a false STALE outline caused by the bribes-history sporadic-data flag.

#### New data (cron side)
Two history files now accumulate, written once/day by rollups folded into the existing `tla-snapshot` cron (no new Render service):
- `apr-history.json` — per-epoch APR + staked averages per pool
- `pool-status-history.json` — per-epoch VP, bucket %, status, depth, staked, and reserves per pool (keyed `pool_address|bucket`)

### Verified working
- Pool Health watchlist flags the genuinely draining pools (LUNA-arbLUNA sustained bleed, LUNA-ATOM faster-than-peers) and stays quiet otherwise; net flow reconciles
- Single-asset pools read "Single"; Astroport reads "Astro"; Skeleton Swap reads "Skeleton"
- Member mode verified against live `adao-positions` data: the stake-vs-value split correctly separates real withdrawals from price — the only genuine withdrawal DAO-wide this epoch is one member trimming USDC-SOLID ~19%; every other apparent outflow was price movement
- Member at-risk view honestly shows "none near threshold" when all of a member's pools sit comfortably above the 1% line
- Both history rollups produce output byte-identical to hand computation against real daily archives

### Known limitations (acceptable)
- Member flow is epoch-over-epoch (positions update at the adao-positions cron cadence), and depends on the prior epoch's weekly archive existing
- For compounder vaults, manual deposits and auto-compounding both grow your unit count and can't be fully separated — hence the "Stake added: deposits + compounding" label

---

## Rev 2.1 — 2026-05-17

Member Data overlay feature + critical bribes resolver bug fix. Surgical additions to the rendering layer; cron data layer untouched (separate cron-side updates ship in the same session — see `cron-scripts/` repo for those).

### What changed

#### Member Data overlay (new feature)
Header dropdown selector — pick any aDAO member, the Overview tab visuals update with their data overlaid in amber. Pools / TLA Liquidity / aDAO tabs unchanged (member overlay is Overview-only by design).

When a member is selected:
- **VP Breakdown pie**: carves a member-colored slice out of "Other" — total VP unchanged
- **Vote Breakdown waterfall**: adds an amber member layer to each pool the member voted in. Bucket totals row gains a member chip; per-pool tooltip gains a member row
- **Threshold Watch**: filters to pools the member voted in. Header gains a "Filtered: {member}" badge. Empty states are member-aware ("None of {name}'s pools are at risk")
- **Member Stats Row**: 6 amber tiles below the global stat tiles — Astroport LPs, Skeleton LPs, Epoch Rewards, Epoch Bribes, Avg APR Non-Amp, Avg APR Amplified. Hidden by default; appears only when a member is selected
- Dropdown styling: dark color-scheme to fix invisible-text issue on some browsers; sorted by VP descending

#### Critical bug fix: bribes resolver
`resolveTokenPriceFromInfo()` was looking up cw20 token prices at `entry.address`. The actual `network-and-prices` schema nests the address at `entry.prices.{source}.address` (or under `prices.{source}.all_chains.{chain}.address` for multi-chain tokens). Any bribe paid in a cw20 token (CAPA, ROAR, etc.) silently priced as $0.

**Impact before fix**: Global Epoch Bribes tile showed ~$820. After fix: ~$1,300 (about 58% more accurate, more aligned with Eris). Member bribes tile correctly captures CAPA bribes (was 100% understated for members voting in LUNA-CAPA, ampCAPA).

Same resolver is used in `buildBribesIndex()` so this fix also corrects the per-pool bribe attribution used by waterfalls and ranking displays.

#### Pool lookup keying
All member-overlay lookups now use `gauge_pool_id` (truly unique, e.g. `cw20:terra1wdz...`) instead of `name+dex` (which can collide e.g. two `LUNA-WBTC|Astroport|BLUECHIP` entries with different gauge IDs). Required adding `gauge_pool_id` passthrough to both pool normalizers in the rendering layer (`votePools` normalizer ~line 3213, `normalizePoolData` ~line 2882).

Member-vote field is `pool_gauge_id`; snapshot field is `gauge_pool_id` — same values, different field names. Both are now handled.

#### Color scheme
- Member overlay color: amber (`#f59e0b`)
- "Other" VP: slate gray (`#64748b`) — was previously amber in waterfall, now consistent with pie chart slate
- Updated all 3 legends (waterfall totals, waterfall bottom, member tile row) for consistency

### Verified working
- Member dropdown populates from `adao-positions/current.json` members array
- Picking any member updates pie, waterfall, threshold watch, and member tile row in sync
- Switching to "All members" cleanly restores the global view
- Global Epoch Bribes tile climbs to ~$1,300 (verified against cron data)
- Member bribes correctly capture CAPA — tested against members voting in LUNA-CAPA pool
- All existing tabs and features continue to work (no regression in the ~7,000 lines of preserved rendering code)

### Known minor issues (acceptable for now)
- Skeleton Swap data labeled in Member Stats row but upstream source is frozen (see audit findings in `PROJECT_KNOWLEDGE.md`)
- Avg APR tiles still use TLA-staked-USD weighting (different from Eris); methodology fix tracked in `CHANGES_PENDING.md`

---

## Rev 2.0 — 2026-05-14

Major rebuild of the data layer to consume from the new TLA cron infrastructure (7 production crons writing to per-cron `*-data_2026` GitHub repos). Rendering code (~7,000 lines of charts, tables, modals, tabs) preserved intact — surgical surgery on data flow only.

### What changed
- **Removed** epoch/phase selector dropdown and snapshot date badge from the header. Data is now continuous (hourly updates) rather than per-epoch manual captures, so picking an epoch makes no sense. Live epoch + countdown remain.
- **Removed** all references to old per-epoch file paths (`tla-data-epoch-{N}-end.json`, `adao-snapshot_{N}_end.json`) which are no longer being written.
- **Added** new data fetch pipeline in `loadEpochData()`: parallel fetches from `tla-snapshot-data_2026`, `network-and-prices-data_2026`, `adao-positions-data_2026`, `bribes-data_2026`, `tla_ext_historical_2026.json`, and `tla_pd_bribes.json`. Falls through gracefully when individual sources unavailable.
- **Added** `buildLegacyDataShape()` transform function that maps the new continuous-data schema to the v3 store shape the existing renderers expect. Preserves all rendering code untouched.
- **Added** "Member Stats" tab link to the tab strip. Points to `dao-tla.html` (page not yet built — Pass 2 of the rebuild).
- **Fixed** aDAO tab now sources from treasury wallet data (`adao-positions/current.json` treasury field). At the TLA-wide level "aDAO" = treasury entity (single voter, 757K VP). Individual members live on the separate Member Stats page.
- **Fixed** TLA Total VP donut chart now shows mathematically truthful breakdown: 24.11M total (max bucket VP = Eris convention) split into Votion VP 6.90M (28.6%), aDAO/treasury VP 757K (3.14%), Other VP 16.46M (68.3%). Reconciled exactly against Votion's actual lockup data shown on votion.money.
- **Fixed** Liquidity DEX vs TLA Staked bar chart now populates correctly (uppercase bucket names matching renderer's expectations).
- **Fixed** Vote Breakdown Waterfall chart now renders all 4 bucket views (STABLE / PROJECT / BLUECHIP / SINGLE).
- **Fixed** Top by APR rankings excluded dust pools (TLA-staked < $20K) and capped at 200% to prevent illiquid pools with huge emissions/TVL ratios from dominating. Top entries now show realistic 70-80% APRs (LUNA-INJ, LUNA-FUEL, LUNA-CAPA, etc.) matching Eris.
- **Fixed** Avg APR weighted by TLA-staked-USD rather than depth-USD. ~40% Non-Amp / ~42% Amplified.

### Verified working
- All 6 tabs render
- Header tiles (Active pools 22 Astroport + 8 Skeleton, Epoch Rewards 339K LUNA / $22.7K, Epoch Bribes $841, Avg APR 40%)
- TLA Total VP donut with truthful breakdown
- Liquidity DEX vs TLA Staked bar chart (all 4 buckets)
- Vote Breakdown Waterfall (all 4 bucket views work)
- aDAO tab matches Eris UI within ±1%: Locked VP 757K, LP $6,669, rewards $453, bribes $443
- Top by APR rankings with realistic values

### Known minor issues (acceptable for now)
- Trend mini-charts on stat tiles will be empty until 2+ weekly snapshots accumulate (~4 weeks)
- Token grade scoring is a simplified stub — needs proper formula refinement
- Avg APR shows ~40% but Eris shows ~55% (different weighting methods, order of magnitude correct)
- Epoch number labeled as 184 instead of 185 — known off-by-one bug in cron output, dates correct. Fix planned across all crons. **[RESOLVED 2026-05-15 — see Rev 2.1 notes and cron README changelogs]**

---

## Rev 1.15 — 2026-05-08

Cleanup pass after first user review of the unified chrome rollout.

### What changed
- Cleaned up the page-specific header: removed the small aDAO logo, the "← Dashboard" backlink under it, and the "by The Alliance DAO •" subtitle. The "Terra Liquidity Alliance Tracker" title and the Eris TLA link remain. Epoch / phase selector and live epoch info on the right side are unchanged
- Cleaned up the page-specific footer: removed the "Updated: 4/26/2026" line (the changelog timestamp is the source of truth now), the "Built by: DeFi Patriot · DM for edits or errors" credit, and the "© 2025 Alliance DAO Community Project. Not affiliated with Terraform Labs..." copyright notice. The disclaimer block (Not Financial Advice / Data Accuracy / Third-Party Links / Use at Your Own Risk) and the Terra Liquidity Ecosystem links row remain
- Made the `last-updated` JS update null-safe since the element it targets was removed
- Fixed changelog modal — was fetching from `/main/logs/tla-log.md` (404), now fetches from `/main/tla-log.md`

---

## Rev 1.14 — 2026-05-08

Initial entry — page brought into the unified site chrome system.

### What changed
- Added unified site header (logo + 5-tab top nav + Terra logo)
- Added mobile bottom tab bar with TLA tab highlighted as active
- Added unified footer with Rev number + Changelog link (this changelog) — appended after the existing page footer (mission statement + ecosystem links preserved)
- Original page-specific controls preserved (epoch selector, phase selector, all charts and data tables)

### Earlier history (untracked)
TLA Stats has been the primary public face for Terra Liquidity Alliance data — voting share charts, lock data, epoch tracking, ve(3,3) analysis. The data pipeline depends on weekly Sunday 23:59 UTC snapshots captured manually via the TLA admin tool (automation is on the roadmap — see CHANGES_PENDING.md). Starting point of formal changelog tracking is rev 1.14.

Going forward, each meaningful change to this page will get its own entry here.
