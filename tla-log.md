# TLA Stats Changelog

This is the change history for `tla-stats.html` (the Terra Liquidity Alliance public dashboard).
Newest revisions on top. Times are UTC.

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
- Epoch number labeled as 184 instead of 185 — known off-by-one bug in cron output, dates correct. Fix planned across all crons.

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
