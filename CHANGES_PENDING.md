# Changes Pending — aDAO website

> Rolling list of identified work for upcoming sessions. See PROJECT_KNOWLEDGE.md "Tracking responsibilities" for what goes here vs. there.
> Older completed items have been pruned — they live in changelog files (`index-log.md` etc.) instead.

---

## 🛠 Active / next round

### 🔥 P1 — Switch adao-positions Render schedule from weekly to daily
**Identified 2026-05-17. Confirmed still pending 2026-05-29.** The cron is currently scheduled `0 1 * * 1` (Mondays only). For the Portfolio Tracker dashboard to accumulate meaningful position history, it needs to run **daily**. The cron code now produces a `data/daily/{YYYY-MM-DD}.json` archive on every run (added 2026-05-17) — that file overwrites within a day, so daily cadence gives one snapshot per calendar day.

Two changes required:
1. **[ ] Update Render cron expression**: `0 1 * * 1` → `0 1 * * *` (this is a manual click in the Render dashboard)
2. **[x] Update `next_expected_run_at` constant in `adao-positions.js`** — done 2026-05-17, now `25 * 60 * 60 * 1000` (25 hours)

Ship both together. If only the Render click happens, the heartbeat is wrong; if only the code change is deployed, the dashboard flags the cron stale every 25 hours.

Without the Render change, letting things run for weeks produces 0 weeks of Portfolio Tracker history. **Top priority — should ship before any other accumulated-data work.**

### ✅ Push current `tla-stats.html` to thealliancedao.com — SHIPPED 2026-05-17
**Done.** Rev 2.1 deployed. Member Data overlay feature live + critical bribes resolver bug fixed. Global Epoch Bribes tile climbed from ~$820 to ~$1,300 as expected. See `tla-log.md` for full record.

### 🟢 P2 — Migrate index.html inline live-data code to `aDAOLive` library
**Identified 2026-05-28 (Rev 3.47).** The shared library `lib/adao-live-data.js` is now the canonical source for live RPC fetching, but `index.html` still has its own inline copies of `fetchLiveTlaDeposits`, `queryChain`, `fetchTlaSharedCatalog`, `fetchLiveTlaDepositsFromChain`, etc. They coexist (both work) but the duplication will drift. Migrate incrementally — when touching one of these code paths for another reason, swap it for the library call.

Bonus: removes ~300 lines from index.html, helping cold-start parse time slightly.

### 🟢 P2 — Migrate dao_treasury.html inline live-balance code to `aDAOLive.getDaoTreasury()`
**Identified 2026-05-28.** `dao_treasury.html` already pulled live wallet balances correctly before the library existed (it was the first page to use this pattern). Now that `aDAOLive.getDaoTreasury()` does the same thing with consistent caching across pages, migrate. The library was tested to return identical values ($13,912.14 across 9 priced tokens) against the page's own code at deploy time, so this is a safe drop-in.

### 🟢 P2 — Fix TLA Deposits modal inside index.html to show live per-pool data
**Identified 2026-05-28.** The TLA Deposits modal (drill-down from the tile) still shows snapshot per-pool data. With `aDAOLive.getDaoTlaDeposits()` already returning live per-position data (16 positions including bluechip + single-asset), the modal can show real-time per-pool breakdowns. Estimated ~80 lines to wire up.

### 🟢 P2 — Enterprise Staked chart shows 403→503 jump in history
**Identified 2026-05-24 (Rev 3.43).** The Enterprise Staked tile now correctly shows 403 (excluding 100 DAO-controlled broken NFTs), but historical chart data captured before the filter was applied shows the unfiltered 503 count. Cron-side fix needed in the source data — either backfill the historical archive files with the corrected counts, or have the dashboard apply the same filter when rendering historical points.

### 🟡 P2 — APR outliers for stable pairs (USDC-USDT, USDC-EURe)
**Discovered 2026-05-17 audit.** These two pools show APR ~5× higher than Eris's number for the same pool. Specific to stable pools — non-stable pools are internally consistent. Likely tied to stable-pair price normalization in the `tla-snapshot` cron's APR formula. Needs investigation.

### 🟡 P2 — Null-dex unnamed pool inflates Astroport count by 1
**Discovered 2026-05-17 audit.** `tla-snapshot` cron has one entry with `name: "cw20:terra1hqq6..."` and `dex: null`. The normalizer defaults null dex to "Astroport" (`p.dex || 'Astroport'`), making this pool count toward the Astroport total. Real fix is cron-side: either classify or skip the unnamed pool. Cosmetic but indicates a classification gap.

### 🟡 P2 — IBC denom resolution gap in network-and-prices
**Discovered 2026-05-17 audit.** The LUNA-USDC bribe asset (`ibc/8D8A7F7253615E5F76CB6252A1E1BD921D5EDB7BBAAF8913FB1C77FF125D9995`) is not in the 27-token `network-and-prices` index. Eris prices this bribe at $12.93 but our resolver returns $0. Fix: add explicit IBC-denom → symbol mapping for known TLA-relevant denoms in the `network-and-prices` cron.

Note: Rev 3.48 added native-denom lookups including this IBC hash → ASTRO in the vote-rewards capture path inside `index.html` as a local workaround. Cron-side fix would let other consumers benefit too.

### 🟢 P3 — SS API migration
**Status 2026-05-26.** Skeleton Swap rebuilt their API around May 4 — moved from `dex.warlock.backbonelabs.io` (frozen) to `/api/pools` direct path. `test.html` temporarily hides SS lines. SS cron needs updating to consume the new endpoint. May intersect with the Rev 3.37 chain-direct rebuild already done — confirm whether the new BackBone endpoint or the chain-direct path is the long-term plan.

### 🟢 dao-tla.html — Member Stats page (Pass 2)
Member-level breakdowns deferred from the `tla-stats.html` V6 rebuild. Data already collected in `adao-positions/current.json` members array (46 members, each with summary VP, vote allocations, locks, rewards, bribes claim status).

**Updated direction 2026-05-17**: this should likely be PROMOTED to header-level Portfolio Tracker (separate page accessed from the header), not a tab inside tla-stats.html. The Member Data overlay shipped in 2026-05-17 covers the inline use case; a proper Portfolio Tracker page covers the deeper "is my position growing, am I being exploited" use case that's the main TLA Stats differentiator from Eris. Wait for 2-4 weeks of accumulated daily data before building.

Spec (whenever built):
- Standalone page, linked from header (alongside Member Data dropdown)
- Per-member portfolio panel: locked VP, individual locks, vote allocations, pending rewards, pending bribes
- **Time-series view** using `adao-positions/data/daily/*.json` history (requires P1 schedule change first)
- P&L computation: position value over time, fees earned, "is this position actually growing"
- Optional: leaderboard by VP, recent activity

### Trend chart accumulation (low priority — passive)
The stat-tile mini sparklines on `tla-stats.html` are currently empty because only `epoch-184` weekly archive exists (cron started capturing this week). Will populate naturally as weekly snapshots accumulate — probably 4+ weeks needed for visual signal. No action needed; just wait.

### Token grade scoring formula refinement
Current `computePoolScores()` in `tla-stats.html` is a simplified stub. Real scoring should weight access/performance/support based on the criteria from `tla_config.json`. Refine once enough historical data accumulates to validate output. **Note (2026-05-17)**: the bigger plan is LP Health Scoring (see P3 below) that goes deeper than token grades — score formula refinement may be subsumed by that work.

### Resilience prereqs (per index-log Rev 3.27 / 3.29 follow-ups)
- [ ] Per-fetch try/catch isolation in `index.html`'s `fetchLiveOnChainData` so one failed treasury balance fetch doesn't poison every downstream calculation
- [ ] Fallback LCD endpoint on 5xx — both `terra.publicnode.com` and `terra-lcd.publicnode.com` returned 500s on May 8, 2026 and the cascade null-safety added in 3.27 was the only thing preventing total dashboard failure
- [ ] Apply the same parallel epoch-fallback pattern from `fetchTlaFromGitHub` to `fetchTlaData` (TLA Stats page) — works correctly today but unnecessarily slow on stale fallback
- [ ] Per-fetch timeout in `fetchLiveOnChainData` for the slow Terra LCD calls — 14.67s load times mostly come from there

### NFT Explorer / dashboard tile work
- [ ] DAO Broken/Held NFTs: live-filter the `nfts` array against the 3 multisig wallet addresses. Hardcoded `1000` per Props 64-69 is correct but not future-proof. Need the two liquidity-wallet addresses (`...8ywv`, `...417v`) added to the codebase first.
- [ ] LST hardcoded ratios (`bLUNA || 1.6048` etc., ~10 places) — soft Design Principle #1 violation but ratios drift slowly. Decide: keep or replace with spinner.

---

## 🚀 P3 — Medium-effort work (next 1-2 months)

### Chain-direct Skeleton Swap capture cron
**Need driven by 2026-05-17 audit.** Current `skeletonswap-lp_data` cron pulls from BackBone Labs' aggregator API (`dex.warlock.backbonelabs.io/api/pools/phoenix-1`), which has been returning cached/stale data for ~30 days. Effective coverage is ~50% real / ~50% duplicate-frozen.

Fix: build a new cron that queries Skeleton Swap pool contracts directly from chain (same approach as Astroport). Frees us from the BackBone dependency entirely. Once running and producing fresh data for several weeks, retire the BackBone-based cron OR keep it as a secondary source for cross-check.

This is the same investigation pattern documented in the audit: chain-direct queries to `terra-lcd.publicnode.com/cosmwasm/wasm/v1/contract/{addr}/smart/{query}` work fine for current state. Schedule and cadence TBD; probably hourly to match Astroport.

### Match Eris APR methodology
**Identified 2026-05-17 audit.** Our `tla-snapshot` cron computes pool APR as `annual_emissions_usd / staked_in_tla_usd × 100`. Eris uses `annual_emissions_usd / depth_usd × 100`. Both correct, measuring different things — our denominator is smaller (TLA-only) so our APR reads higher.

User preference (2026-05-17): match Eris exactly to avoid user confusion when cross-referencing pages. Fix: change formula in `tla-snapshot.js` to use `depth_usd` denominator. **One-line change**, all downstream APRs will then match Eris within rounding. Test thoroughly before deploy — some methodology differences may remain for stable pools (see P2 above).

### Chain-direct verification layer for Astroport
**Defense against API manipulation.** Current astroport cron trusts Astroport's API completely — `pools.getAll` is the single source of truth for TVL, volume, fees, reserves. If their API ever serves stale or manipulated data (as happened to BackBone for Skeleton), we'd have no way to detect it.

Add: hourly chain-direct query to each pool's `{pool:{}}` contract endpoint, cross-check returned reserves against the API. If discrepancy > N%, flag and log. ~50 lines added to the cron. Not blocking accumulation — useful as a trust layer once we're building scoring on top of this data.

---

## 🏗 P4 — Major builds (after 2-4 weeks of accumulated daily data)

These are the differentiating products. Wait until we have enough daily history to power them meaningfully.

### Portfolio Tracker page
The big one. Per-member time-series view: position value, fees earned, P&L computation, "is your position actually growing or being harvested." Uses `adao-positions-data_2026/data/daily/*.json` history (depends on P1 schedule fix shipping first).

Eris structurally can't tell users this — they're the protocol. A third-party analytics site can.

### LP Health Scoring
Composite score per pool from sustained-over-N-epochs metrics:
- Depth stability (variance over time = real LPs vs transient capital)
- Volume-to-depth ratio sustained (real trading vs idle TVL)
- Fee generation consistency (real revenue vs promotional emissions)
- Oracle source (chain-native vs centralized vs manipulable — binary flag, weighted heavily)
- Whale concentration (Gini coefficient of LP shares)
- Bribe-to-organic-volume ratio (gaming this is expensive over multiple epochs)

The whole point is **resistance to gaming**. 24h data is gameable; sustained multi-epoch metrics aren't. Each scoring factor needs a "you can't game this without spending more than you'd extract" justification.

### Pools + TLA Liquidity tabs rebuild
Once 4+ epochs of clean daily data exists, rebuild these tabs with proper historical context. Multi-epoch depth/volume trends, "this LP had $X depth on date Y" verifiable at block height, etc.

---

## 🚀 Future projects — separate threads

### ✅ TLA data collection automation — COMPLETED 2026-05-12 → 2026-05-14
**Done.** 9 production crons live on Render (votion, skeletonswap, astroport, bribes-history, network-and-prices, tla-snapshot, adao-positions, nft-inventory, marketplace-stats). All writing to their respective `*-data_2026` GitHub repos. `tla-stats.html` rebuilt (V6) to consume from the new continuous data sources instead of per-epoch manual snapshots. See PROJECT_KNOWLEDGE.md "TLA cron infrastructure" section for the as-built architecture. `tla_tool.html` and `tla-tool_ext.html` retained as manual fallback but no longer the primary capture path.

### Slim manual capture by prefilling chain-derivable fields
Pair this with the cron work or do standalone. The Rev 3.31 `fetchDaoTlaVp` work proved the live-query pattern. Now mostly moot since the new crons cover most of what was manual.

### Capa Protocol integration prep
Once partnership solidifies — likely new pages/sections + lore integration (per the framework Lion DAO established → Canyon-Clans of Ozara North).

### Static site generator migration
Big refactor, not urgent. Would dramatically simplify the cross-page chrome rollout (currently per-page duplicated code) and meta-tag application. Astro or Eleventy. Logged for awareness; only consider when it actively blocks something.

---

## 🧹 Cleanup — low priority, safe to defer

- [ ] Remove dead Logos modal HTML in `index.html` (line 1745+) and the `'logo-modal-trigger': 'logoModal'` mapping in JS (~line 5396)
- [ ] Delete `unclaimed-stale-banner` HTML element from `index.html` (Rev 3.30 hides it; replaced by modal)
- [ ] Deduplicate `fmt` helper from Rev 3.26 vs `safeLocale` from Rev 3.27 — both do the same thing
- [ ] Remove `fetchTlaFromGitHub` + `_adaoSnapshotCache` if anything is still left after Rev 3.31's deletion (verify)
- [ ] Old `dao_governance.html` is renamed to `dao.html` (Rev 3.22) — check Vercel for stale 404s on old URL

---

## 📝 Open questions / decisions needed

- [ ] **APR methodology — match Eris or document our own?** Cron's APR uses TLA-staked denominator; Eris uses depth_usd. Both correct, measure different things. Current direction (2026-05-17): match Eris. One-line cron change, but want to test thoroughly. See P3.
- [ ] **Skeleton Swap data going forward** — keep capturing best-effort with "unverified" label (current decision), or stop capturing entirely until chain-direct cron is built? Current call: keep, the cost of leaving it running is low.
- [ ] LST ratios: keep the hardcoded fallbacks or remove? (See Design Principle #1; current call is keep.)
- [ ] Astroport chain-direct verification — build it or trust the API? See P3. Adds complexity; useful as trust layer for scoring.
- [x] ~~Should the cron run daily or hourly?~~ — **Resolved.** Hourly for tla-snapshot + network-prices (data freshness matters), daily for DEX/bribes captures (less time-sensitive). **Reopened 2026-05-17 for adao-positions**: was weekly, should be daily for Portfolio Tracker history. See P1.
- [x] ~~Where does the new cron write?~~ — **Resolved.** One `*-data_2026` GitHub repo per cron. Independent systems principle.
- [x] ~~How does the cron handle the multi-week capture gap?~~ — **Resolved.** Crons started fresh in May 2026 with no historical backfill. Historical data before this lives in legacy `tla-ext_json_storage` files (used for trend charts only).
- [x] ~~Epoch numbering off-by-one fix — rename archives or accept gap?~~ — **Resolved 2026-05-15.** Accepted the gap. All crons now use `epochIndex + 1`. Verified live 2026-05-17: crons correctly report epoch 185. Epoch-184 archives exist but no epoch-185 archive files until each cron's next nightly run.
- [x] ~~Should we backfill 4 months of historical chain data for the new tabs?~~ — **Resolved 2026-05-17.** Not feasible without paid archive node access ($50-500/mo). Public LCDs prune state after ~100 blocks (~10 min). Tendermint RPC disabled on `terra.publicnode.com`. Astroport API has no historical endpoint. Decision: forward-only chain-based capture. Accept 4-month wait. In 4 months we'll have 4 months of trustworthy data for both DEXes.
