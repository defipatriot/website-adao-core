# Changes Pending — aDAO website

> Rolling list of identified work for upcoming sessions. See PROJECT_KNOWLEDGE.md "Tracking responsibilities" for what goes here vs. there.
> Older completed items have been pruned — they live in changelog files (`index-log.md`, `catalog-log.md`, etc.) instead.

Last cleared: **2026-06-07** (post NFT inventory Rev B deploy). Rev 0.16 catalog Phase 0 items previously cleared 2026-06-06.

---

## 🛠 Active / next round

### 🔥 P1 — NFT Explorer page migration (Rev 2)
**Identified 2026-06-06 during deving.zone outage investigation. Rev B cron foundation shipped 2026-06-07.** The `nft-inventory` cron Rev B now produces a full chain-of-truth replacement for `deving.zone/nfts/alliance_daos.json` (which has confirmed bugs: 16 missing DAODAO stakers, 54 undercounted, DAODAO contract itself listed as a 384-NFT user, no Atrium awareness, no Boost seller resolution). The explorer page still reads from deving.zone — Rev 2 swaps the data source.

**Affected file:** `nft-explorer-app.js` (237 KB main page logic)

**Changes needed:**
1. **[ ]** Swap `STATUS_DATA_URL` from `deving.zone/nfts/alliance_daos.json` → our cron's `nfts.json` raw URL
2. **[ ]** Adapt `mergeNftData()` to handle Rev B records[] format (schema v2) with new fields: `real_owner`, `listing{...}`, classification flags
3. **[ ]** Replace dead `MEMBERS_CSV_URL` (`adao_json_storage/main/members.csv` — repo dead since 2026-05-17) with our cron's `summary.json` (richer data — per-staker counts + voting_power_pct)
4. **[ ]** Add marketplace badges with prices: "Listed: 2,200 bLUNA ($1,875)" — BBL/Atrium/Boost icons
5. **[ ]** Add backing display tile: collection-wide treasury value ($1.65M today) + per-NFT share (88.20 ampLUNA) + boost-mechanic story ("share grew +12.3% since launch as 1,093 NFTs broke")
6. **[ ]** Add AbortController timeouts on all `.json()` fetches (deving.zone-hang lesson — same fix applied to `index.html` below)
7. **[ ]** Add new badges/filters: "DAO Treasury" (898 broken), "Atrium Listed" (1), distinguish "Enterprise Staked" (403 real) vs "Enterprise DAO Broken" (100 gov)
8. **[ ]** Surface pending claims from `summary.daodao_pending_claim` (cron ships this as of Rev B.3): a global "N NFTs unstaked & pending claim" stat, and a per-wallet "You have N NFTs ready to claim" nudge when a viewed/connected address appears in `claimable[]`. Show `reconciled: false` defensively (render count, treat per-wallet detail as best-effort).

Estimate: 4-6 hrs. Verify cron data has run cleanly for 24+ hours first. Don't ship Rev 2 same-day as Rev B.

---

### 🔥 P1 — Switch adao-positions Render schedule from weekly to daily
**Identified 2026-05-17. Confirmed still pending 2026-06-06.** The cron is currently scheduled `0 1 * * 1` (Mondays only). For the Portfolio Tracker dashboard to accumulate meaningful position history, it needs to run **daily**. The cron code now produces a `data/daily/{YYYY-MM-DD}.json` archive on every run — that file overwrites within a day, so daily cadence gives one snapshot per calendar day.

Two changes required:
1. **[ ] Update Render cron expression**: `0 1 * * 1` → `0 1 * * *` (manual click in Render dashboard)
2. **[x] Update `next_expected_run_at` constant in `adao-positions.js`** — done 2026-05-17, now `25 * 60 * 60 * 1000` (25 hours)

Ship both together. If only the Render click happens, the heartbeat is wrong; if only the code change is deployed, the dashboard flags the cron stale every 25 hours.

Without the Render change, letting things run for weeks produces 0 weeks of Portfolio Tracker history. **Top priority — should ship before any other accumulated-data work.**

---

### 🔥 P1 — Migrate `index.html` off retired admin-tool storage repos
**Identified 2026-06-05 during deving.zone outage investigation.** The page still reads from old admin-tool storage repos (`tla_json_storage`, `tla-ext_json_storage`) that stopped publishing on **2026-05-17**. The page silently falls back to epoch 185 (now 3+ weeks stale) labeled "STALE - N epochs old" in the console but renders without obvious warning to users.

Affected fetches in `index.html`:
- `fetchTlaData()` — `tla_json_storage/main/tla-data-epoch-N-end.json` (last write epoch 185)
- `fetchTlaExtData()` — `tla-ext_json_storage/main/tla-ext-epoch-N-end.json` (last write epoch 185)

The data now lives across multiple `_2026` repos with different schemas. Field mapping documented in catalog-log.md (Rev 0.15 deep-dive).

**Path forward:** Hybrid approach — `fetchTlaExtData` has a clean 1:1 mapping to `network-and-prices-data_2026/data/network-and-prices.json` (do this first, 1-2 hrs). `fetchTlaData` needs multi-source composition (do as separate larger pass, 4-6 hrs).

**Why P1 now:** every passing day the stale data drifts further. Member-facing tiles (TLA Deposits, Locks, treasury balances) become wrong.

---

### 🟢 P2 — Add timeout / AbortController to all `await response.json()` calls in `index.html`
**Identified 2026-06-05** during deving.zone outage. When deving.zone returned 200 OK headers but stalled mid-body, `fetch().catch()` didn't fire (it only handles network errors), `response.json()` hung forever waiting for body end, and the page appeared blank/spinning with no JS error.

**Fix pattern:**
```js
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 10000);
fetch(url, { signal: controller.signal })
  .finally(() => clearTimeout(timeoutId))
const data = await response.json().catch(() => null);
```

Apply to all four primary fetches (`onChainStatsUrl`, `contractUrl`, `ampLunaRateUrl`, `priceUrl`) and any other `.json()` in the page. Without this, any third-party endpoint hiccup blanks the page silently.

---

### 🟢 P2 — Filter LCD 500 responses on "no new proposal" as not-an-error
**Identified 2026-06-05.** The `checkForLiveProposals` loop in `index.html` queries `terra1va3tny5...` for proposals 38, 39, 40, 41, 42 to detect new ones. The contract throws 500 when proposal doesn't exist — which is the **normal case** (no new proposals). The page logs these as errors, generating ~5 console errors per page load even when nothing is wrong.

**Fix:** Treat HTTP 500 with a "proposal does not exist" error body as "no new proposal" (not an error). Stop the loop on first 500 (proposals are sequential).

---

### 🟢 P2 — Migrate `index.html` inline live-data code to `aDAOLive` library
**Identified 2026-05-28 (Rev 3.47).** The shared library `lib/adao-live-data.js` is now the canonical source for live RPC fetching, but `index.html` still has its own inline copies of `fetchLiveTlaDeposits`, `queryChain`, `fetchTlaSharedCatalog`, `fetchLiveTlaDepositsFromChain`, etc. They coexist (both work) but the duplication will drift. Migrate incrementally — when touching one of these code paths for another reason, swap it for the library call.

Bonus: removes ~300 lines from `index.html`, helping cold-start parse time slightly.

### 🟢 P2 — Migrate `dao_treasury.html` inline live-balance code to `aDAOLive.getDaoTreasury()`
**Identified 2026-05-28.** `dao_treasury.html` already pulled live wallet balances correctly before the library existed (it was the first page to use this pattern). Now that `aDAOLive.getDaoTreasury()` does the same thing with consistent caching across pages, migrate. The library was tested to return identical values ($13,912.14 across 9 priced tokens) against the page's own code at deploy time, so this is a safe drop-in.

### 🟢 P2 — Fix TLA Deposits modal inside `index.html` to show live per-pool data
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

### 🟢 P2 — CoinGecko bulk fetch failing in `network-and-prices-data_2026`
**Identified 2026-06-05.** The `network-and-prices` cron heartbeat reports `coingecko_bulk.ok = false` causing overall `status = partial`. Astroport prices are filling the gap so user-facing data still works, but CG bulk is failing systematically (rate-limited 429s observed in the cron log). Investigation needed in the `network-and-prices` cron code.

### 🟢 P2 — TLA Chain Registry catalog: acquisition guide curation pass
**Identified 2026-06-02 audit. Still pending.** Council members (especially the owner) have first-hand verified routes for tokens they actually hold. Several tokens in TLA still have `auto_suggested` or `route_known_unverified` guides where a council member could provide a verified route.

Drafts captured in `catalog-log.md` Rev 0.10 narrative:
- **ATOM** — standard Keplr IBC from Cosmos Hub. Verified by owner 2026-06-02 (deposit test).
- **USDC** — Swapped.com → Keplr Noble → IBC to Terra. Verified by owner 2026-06-02.
- **wBTC.atom** — Skip.go bridge from Ethereum WBTC (Eureka path). Route_known_unverified.
- **PAXG** — same Skip.go pattern from Ethereum. Route_known_unverified.
- **wBTC.creda.a** — Creda Finance minting on Terra (not a bridge). Unverified.
- **USDt** — auto-suggested guide will show "Kava-suspected" from bridge data.
- **EURe** — owner noted "truly don't know how you get it"; auto-derived shows source-chain hint.

Effort: low per token (a JSON entry). High clarity benefit — users deposit into the wrong variant if they pick the wrong bridge.

### 🟢 P2 — TLA Chain Registry catalog: Eris CG-ID outreach
**Identified 2026-06-02 audit. Still pending.** 3 tokens have `coingecko_match = "mismatched"` where Eris's `/prices` claims a CoinGecko ID that doesn't actually match the token on CG's terra-2 platform list. These produce wrong USD prices on any consumer that trusts Eris's claim directly.

User-side: ping Eris team, ask them to correct the mappings in their backend. Catalog-side: already handled (Stage 7c flags as mismatched, downstream consumers can skip).

### 🟢 P2 — TLA Chain Registry catalog: fill council member curation candidates
**Identified 2026-06-05 (Rev 0.15).** 125 TLA member wallets have no curated label and no PFPK profile name. Top 30 by VP templatized in `tla-chain-registry/curated/curation-candidates.json` (drop-in compatible with `wallets.json`).

User action: open the file, fill in `label` fields for addresses you recognize, merge into `wallets.json` under the `wallets` key, push. Next cron run picks them up.

Biggest unnamed wallet: 5.4M VP (`terra13aae4futz6jk...`) — significant council member.

### 🟢 P3 — NFT Inventory cron — Rev C: tier architecture (hot/warm/cold split)
**Identified 2026-06-07 during Rev B design.** Rev B ships with single hourly cadence. Rev C splits into:
- **Hot** (every 15 min): user-held + marketplace + recently-unstaked (~1,200 NFTs, ~115k queries/day)
- **Warm** (daily): staked + claims/unstaking (~2,064 NFTs, ~2k/day)
- **Cold** (weekly Sun 02:00): unminted + DAO-broken full reconcile (10k once weekly)

Achieves 47% query reduction vs current 240k/day, with 4x freshness on hot data. Override file `tla-chain-registry/curated/nft-overrides.json` for manual promotion to hot path when DAO releases NFTs via prop. Weekly cold reconcile is safety net (max 7-day drift before catching missed movements).

**Don't ship before Rev 2 is live** — Rev 2 page should be reading the merged `nfts.json` regardless of which tier path produced each record. Schema is the same.

### 🟢 P3 — NFT Inventory cron — Rev D: daily yield timeline
**Identified 2026-06-07.** Parse `update_rewards_callback` events from chain transaction history → exact daily ampLUNA inflows + 7d/30d/all-time rolling averages + annualized APR calculation. Pattern decoded from txn `70757515D0FEBE07DABC2013CAC9217514C16AE252AA54BF5E395A9885215B18` on 2026-04-25.

Daily inflow ~809 ampLUNA today (90% of 899 ampLUNA produced by Alliance staking → 10% goes to DAO main wallet). Per-NFT daily yield = `daily_inflow / unbroken_count` = ~0.091 ampLUNA today.

Rev B's `data/daily/{YYYY-MM-DD}.json` snapshot already captures the substrate — Rev D adds the txn-history parser and surfaces timeline charts on the explorer page.

### 🟢 P3 — NFT Inventory cron — Rev E: pending-unstake tracking
**Identified 2026-06-07. DAODAO half SHIPPED as Rev B.3 (2026-06-07).** DAODAO pending-claim tracking is live: count is chain-truth (`custody − total_power_at_height`), per-wallet attribution tracked forward via `unstake`/`claim_nfts` tx-search, persisted in `data/v2/pending-claims.json`, self-reconciling. Note the original `claims{address}` idea was superseded — that query is per-address only and misses full-unstakers (zero VP, zero in-wallet), so we use the historical event diff instead. **Remaining:** (a) explorer-side "ready to claim" nudge — see P1 item 8 below; (b) Enterprise NFT staking pending-unstake tracking (if Enterprise exposes a claim queue — not yet investigated; lower value, far fewer unstakes there).

### 🟢 P3 — NFT Inventory cron — Rev F: marketplace offers + bid history
**Identified 2026-06-07.** BBL `collection_offers_by_contract` (collection-wide buy offers), Atrium `offers_by_nft` + `collection_offers_for_collection`, BBL `bid_history_by_auction_id` (per-auction bid timeline). Surface buyer demand + live auction bid feed.

### 🟢 P3 — Generic multi-collection NFT inventory cron (Rev G future direction)
**Identified 2026-06-07.** The nft-inventory architecture (all_tokens enum → per-NFT info → marketplace integration) is collection-agnostic. Rev G could parameterize the cron to track any cw721 on Terra — including the other 7 Terra collections on BBL (Skeleton Punks, pixeLions, Galactic Punks, SoulReapers, Burning Lion Festival, Origin Enigma, Scandalous Birds). One cron writes per-collection JSON files; any future explorer page can read whichever collection it cares about. Same chain-of-truth pattern applied broadly.

Out of scope for current TLA work — note for future direction. Lion DAO collections (pixeLions, Burning Lion Festival) noted as candidates if/when there's user demand for an "ecosystem NFT catalog" page.

### 🟢 P3 — SS API migration
**Identified 2026-05-04.** The SS API migrated to `/api/pools` ~May 4. The cron already handles the new endpoint but `test.html` temporarily hides SS lines (legacy display logic). Cleanup: re-enable SS lines in `test.html` once verified, or remove if no longer needed.

---

## 🆕 New ideas / not yet prioritized

### Phase 1+ direction (post Phase 0 lock-in)

After Rev 0.16 locked in Phase 0, four directions for next phase:

- **A. TLA Stats migration** — evolve existing 7,000-line `tla-stats.html` Rev 3.51 to consume catalog data via `aDAOLive.getTlaCatalog()`. Big effort but biggest user impact.
- **B. Member Stats `dao-tla.html`** — net-new page using catalog as foundation. Per-member VP, positions, voting patterns, P&L. Fresh build, no legacy.
- **C. `index.html` migration** — close the tech debt from the deving.zone investigation (also overlaps with P1 above).
- **D. Portfolio Tracker** — depends on adao-positions daily archive being in place (P1 above), then time-series + P&L.

User to choose direction at next session start.

### Hardening: third-party endpoint resilience
The deving.zone outage exposed how a single third-party JSON endpoint hanging mid-body can blank the entire page. Pattern in P2 above (AbortController on `.json()`) is the immediate fix. Broader hardening could include:
- Cached fallback for `deving.zone/nfts/alliance_daos.json` (we have 157-member CSV)
- Service worker or `<noscript>` fallback page
- Surface "feature degraded" banner instead of blank when key endpoints fail

---

## ✅ Recently shipped (last 30 days, summarized — full detail in changelogs)

- **NFT Inventory Rev B.6 (2026-06-07)**: DAODAO pending-claim tx-search fix. LCD started rejecting the query (`400 "specify tx.height with strict equality"`) because it carried a `tx.height>` range; dropped the height term from the query and moved height filtering client-side in `fetchDaodaoTxs`. Restores forward per-wallet attribution tracking (count was always chain-truth; only the "who" was frozen). Parsers/reducer unchanged; logic re-verified (genesis replay → [1319,3605,6847,7123], incremental no-op, forward claim removal). Live LCD acceptance confirmed on the Render run. Detail in cron README "Rev history" + "Failure modes". This completes the DAODAO half of Rev E; only the page-side nudge (P1 item 8) + optional Enterprise claim-queue tracking remain.
- **NFT Inventory Rev B.5 (2026-06-07)**: USD pricing fix — it had been silently skipping (both sister-cron URLs 404'd, and the parser assumed a schema that didn't match). Corrected URLs (`…/data/network-and-prices.json`, `…/2026/current.json`) and rewrote `fetchPriceData` to the real schema: LUNA from `token_prices.LUNA.final_price_usd`, ampLUNA from `token_prices.ampLUNA.final_price_usd` (fallback `lst_ratios.ampLUNA.ratio × luna`), joining registry catalog (address→symbol+decimals) with `token_prices` (symbol→price). Verified live: LUNA $0.0512, ampLUNA $0.1103 → `treasury_value_usd` ≈ $86.8K, `per_nft_value_usd` ≈ $9.74 (were null). Marketplace listing USD now resolves. Detail in cron README "Rev history".
- **NFT Inventory Rev B.4 (2026-06-07)**: Marketplace pagination hardening. Fixed log-spam (`⚠ NFT #X listed on BOTH BBL and BBL` repeated to the page cap) that surfaced once BBL active listings crossed 30 (now 43) and pagination began re-fetching the same window. All three marketplace fetchers now de-dupe by listing id and break the page loop when a page brings nothing new; merge warning now fires only on genuine cross-marketplace conflicts. Data was always correct (one listing kept per token, classification sums to 10000) — fix removes noise + ~100 wasted queries/run. Marketplace data layer only; classification/pending-claim logic untouched. Detail in cron README "Rev history".

- **NFT Inventory Rev B.3 (2026-06-07)**: DAODAO pending-claim tracking. Surfaces NFTs unstaked from DAODAO but not yet claimed (7-day queue, or forgotten indefinitely). Count is chain-truth (`daodao_staked` custody − `total_power_at_height` active stake = 1,661 − 1,657 = 4); per-wallet attribution tracked forward via `unstake`/`claim_nfts` tx-search, persisted in `data/v2/pending-claims.json`, reconciled every run (heartbeat `daodao_pending_reconciled`). Seeded once with 4 verified legacy forgotten-claims (tokens 1319, 3605, 6847, 7123); self-maintaining thereafter. Verified end-to-end against full chain history before deploy. Inline in `nft-inventory.js` (+~196 lines, additive). New `summary.daodao_pending_claim` block. Explorer "ready to claim" nudge is the page-side follow-up (P1 item 8).

- **NFT Inventory Rev B (2026-06-07)**: Schema v2 + chain-of-truth replacement for deving.zone. Treasury/Enterprise classification fixed (898 treasury + 403 real Enterprise stakes + 100 DAO-controlled Enterprise broken). All 3 marketplaces (BBL 43 + Atrium 1 + Boost 4) with seller resolution. Backing data (ampLUNA balance + per-NFT share). Sister cron price integration for USD conversion. Daily snapshots for future timeline work. ~250 lines new code in `cron-scripts/nft-inventory/nft-inventory.js`, schema v1→v2 with backward-compat aliases preserved so existing dashboard JS keeps working during Rev 2 migration window. Detail in `cron-scripts/nft-inventory/README.md` "Rev history". Page-side migration (Rev 2) deferred to next session — see P1 above.
- **Rev 0.16 (2026-06-06)**: Phase 0 lock-in — 5 polish fixes (Eris not labeled DEX, pair_type normalization, definitional failure detection, SS source synthesis, expanded fingerprint)
- **Rev 0.15 (2026-06-06)**: contract_info via cw2 raw storage (fixes Rev 0.14 error spam), SS indexer correction, avatar capture defensive ungating, curation candidates file
- **Rev 0.14 (2026-06-05)**: Pool architecture surfacing — all 75 pools get architecture object (contract, version, pair_type, dex)
- **Rev 0.13 (2026-06-05)**: Wallet names + avatars — 668/668 wallets have meaningful labels (PFPK names + synthesized DAO-membership labels)
- **Rev 0.12.x (2026-06-05)**: Token logos (3-layer system) + curated URL audit + CDN cache bypass via SHA-pinned URLs
- **Rev 0.11 (2026-06-05)**: amplp classification fix — 65 amplps fully classified with bucket inheritance
- **Rev 0.10 (2026-06-02)**: 10 systemic catalog fixes (self-referential vault detection, Stage 5/6/7 cascade, source coverage transparency)

Phase 0 LOCKED IN as of 2026-06-06 after Rev 0.16 deploy.
