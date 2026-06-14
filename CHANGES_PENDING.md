# Changes Pending — aDAO website

> Rolling list of identified work for upcoming sessions. See PROJECT_KNOWLEDGE.md "Tracking responsibilities" for what goes here vs. there.
> Older completed items have been pruned — they live in changelog files (`index-log.md`, `catalog-log.md`, etc.) instead.

Last cleared: **2026-06-07** (post NFT inventory Rev B deploy). Rev 0.16 catalog Phase 0 items previously cleared 2026-06-06.

---

## 🛡 Systemwide reliability audit (2026-06-09)

Triggered by finding that `nft-inventory.js` had been *silently* dropping DAODAO unstakes for months (a publicnode pagination quirk: `pagination.offset` is ignored, only `page` is honored). That one bug exposed a recurring **failure-class** pattern. Every cron was walked through the checklist below. The common root across all findings: **code that couldn't distinguish "query failed" (null) from "no data" ([]/end-of-list)**, which silently produces incomplete data that can reach permanent archives.

### Failure-class checklist (run this against any new cron)
- **F1 — Pagination truncation.** `pagination.offset` (ignored by publicnode → use `page`), `page`-cap, or a `start_after` loop that stops early.
- **F2 — Silent null-coercion.** `r || []` / `Array.isArray(r) ? r : []` right after a query that returns `null` on rate-limit → empty masquerades as "no data."
- **F3 — Overwrite-with-partial.** A snapshot clobbers last-good with fewer/empty records on a bad run (worst when it reaches a permanent archive).
- **F4 — Corrupt-vs-absent input.** A `try/catch` that treats a *corrupt* file like a *missing* one → silently drops a whole source.
- **F5 — Staleness / schema drift.** Static reference data going stale (oracle), or an upstream field rename silently zeroing a parser.
- **F6 — Required-vs-optional misclassification.** A source that should be fatal treated as optional → partial publishes marked `ok`.
- **F7 — Heartbeat honesty.** Does `status` actually flip to `partial`/`error` on failure, or always say `ok`? If it lies, the health widget never alerts.
- **F8 — Epoch/time boundary.** Off-by-one epoch, UTC flip, missed end-of-epoch window → irreversible wrong-epoch capture.

### Fixes shipped this pass
- **[x] `cron-scripts/nft-inventory/nft-inventory.js`** — F1: `buildTxSearchUrl`/`fetchDaodaoTxs` now page-based `ORDER_BY_DESC` (was ignored `pagination.offset`). Captures all unstakes; `reconciled` flag will read true.
- **[x] `cron-scripts/tla-snapshot/tla-snapshot.js`** — **F2+F3 (critical):** added a completeness gate after the 9 core chain queries (`gauge_infos × 4`, `total_staked_balances × 4`, `distributions`). A `null` (failure) now aborts the run (exit 2, no publish) instead of `|| []`-coercing a whole bucket to empty and freezing it into the permanent daily archive.
- **[x] `cron-scripts/chain/tla-registry/tla-registry.js`** — **F2 (high):** `list_stakers` + `all_tokens` enumeration loops now distinguish `null` (failure) from `[]` (genuine end); a mid-walk failure records to a module `ENUMERATION_FAILURES` registry → status `partial` (+ surfaced in snapshot). No more silently-truncated catalog.
- **[x] `cron-scripts/tla-vp-holders/tla-vp-holders.js`** — F2: same `all_tokens` truncation fix → `ENUM_INCOMPLETE` → status `partial`.
- **[x] `cron-scripts/bribes-history/bribes-history.js`** — F2+F7: proposal-walk truncation fix; **added a `partial` status it never had** (`PROPOSALS_INCOMPLETE`).
- **[x] `cron-scripts/adao-positions/adao-positions.js`** — F7: run status now escalates to `partial` when any member portfolio has `_errors` (was only treasury/council), + `members_with_errors` in heartbeat stats.
- **[x] `nft-inventory-data_2026/{nft-provenance,bbl-sales,atrium-sales}-backfill.js`** — **F3 (critical):** never-shrink publish guard. History is append-only; a sweep producing fewer records than committed = incomplete → abort (exit 1), don't overwrite.
- **[x] `nft-inventory-data_2026/nft-analytics-builder.js`** — **F1 fix #5 (F4):** boost/atrium/bluna inputs now distinguish corrupt (throw) from absent (skip). **F5:** extends LUNA + bLUNA oracles to "now" via live `network-and-prices` prices, so post-oracle sales price live instead of stale last-known (best-effort; falls back to static oracle).

### Clean bill
- **`network-and-prices`** — the model cron (per-source `.ok` flags, `stuck/partial/ok` escalation, fingerprint staleness detector). Propagate its fingerprint approach to others over time.
- **`astroport`, `votion`, `skeletonswap`** — single-fetch / concurrency-worker patterns, no enumeration loop to truncate. (`skeletonswap`'s `while(true)` is a parallel-map worker, not pagination.)

### Remaining (flagged-not-silent — polish, not landmines)
1. **[ ] F5 follow-up:** the static `luna-usd-daily.json` / `bluna-usd-daily.json` only get *live-extended* at build time now; consider a tiny daily appender so the on-disk oracle itself grows (the in-memory extension covers correctness today).
2. **[ ] `network-and-prices` carry-forward:** on dual-oracle failure for a token it writes `final_price_usd: null` (overwriting last-good). Already flagged `partial` + dashboard caches, so visible. Fix: add `fetchPreviousSnapshot()` and carry forward last-good with a `stale: true` flag. *Touches the linchpin — test carefully.*
3. **[ ] `astroport` / `votion` partial status:** both are throw-based all-or-nothing; `astroport` can partially succeed (liq ok, vol fail via `fetchOk`) but status stays `ok`. Minor F7 — add a `partial` branch.
4. **[ ] `marketplace-stats` (Pixel-Lions, parked):** `fetchBblActivityPages` catch does `warn + break` (silent truncation of the activity feed). Tier 3, daily-refresh, no permanent archive. Fix with the same flag-to-`errors` pattern when Pixel-Lions work resumes.

---

## 🎯 TLA Stats expansion — capture layer ✅ BUILT (2026-06-13), page UI next

**The entire data-capture layer is built and live.** All four pillars now have their feeds. Full detail in `cron-scripts/README.md` and `PROJECT_KNOWLEDGE.md`.

### ✅ DONE — capture layer (2026-06-13)
- **[x] adao-positions → daily** (`0 1 * * *`) — P&L history accumulating.
- **[x] `lib/capture-engine.js` extracted** from adao-positions (verified identical output) + **v1.1 enrichments**: first-participation tenure (chain-native `user_first_participation`), lock end_period/auto-max/weeks-to-unlock, inactive-take exposure, VP spread. `lib/ally-capture.js` for ally discovery.
- **[x] adao-positions widened** to all 156 members (named + unknown; current=all, archives=registered-only). Surfaced ~510K VP (21%) previously invisible.
- **[x] `tla-participants`** — lock holders ∪ bribe providers, ~203 participants, 26.8M VP (full electorate). Live-only.
- **[x] `adao-allies`** — Pixel Lions + Lion DAO bundled in ONE cron, per-ally isolation, registered-only. (Lion DAO is **cw20-staked** → `daoVotingCw20Staked`, not token — gotcha.)
- **[x] `tla-locks`** — system + per-holder lock intelligence. Stale-VP gap **3.49M VP unclaimed**, unlock cliffs, decay projection, per-asset VP, auto-max split.

**Gotchas captured (cost real time):** (1) `GITHUB_REPO` env var needs `defipatriot/` prefix or 404 on publish — bit us 3×. (2) DAODAO formula depends on voting-module type (cw721/cw20/token) — confirm via `{info:{}}`. (3) tla-locks stale-VP math keys LST ratio off symbol — raw-address symbols undercount (269K→3.49M); hardcoded symbol map. (4) GitHub web-UI partial commits — edit in place, verify via codeload not raw CDN.

### 🔥 P1 — Vote/lock history backfill (spec'd 2026-06-13, NOT built) — `SPEC-tla-history-backfill.md`
The NFT-provenance-pipeline move applied to TLA: EVENTS (votes, lock create/relock/merge) ARE backfillable via `tx_search` to genesis (they're permanent transactions); VALUATIONS (position USD over time) are NOT (pruned state, forward-only). New repo `tla-history-data_2026`. Feeds Vote Intelligence (vote churn, votes-on-dead-LPs). Own cron, mirror the NFT backfill pattern (DESC paging, seed-once, never-shrink, F2 guards). Probe the exact `wasm.action` values first.

### 🔲 P2 — TLA Stats page (`tla-stats.html`) — the four pillars UI
All raw data feeds now EXIST. **Portfolio Tracker**, **LP Performance & Health Scoring**, **Bribes Tracking**, **Vote Intelligence**. Bribes/Vote-Intelligence buildable soonest; Portfolio Tracker P&L needs the daily-archive runway (clock started 2026-06-13). `tla-stats.html` is ~7,000 lines of polished rendering — **data-layer changes only, never restructure the render code.**

### 🟢 P3 — capture-layer polish (deferred, non-blocking)
- **[ ] Label contract-holders** in tla-participants (the 63-char top holders, possibly Votion) via a known-contract map.
- **[ ] Lion DAO ROAR stake USD** — currently `stake_raw` only (ROAR ≈ $0.000000229, 6 decimals, balances in billions); value via network-and-prices when wanted.
- **[ ] Tenure fallback** to lock `start_period` for holders who never voted (first_participation null).

---

## 🛠 Active / next round

### ✅ DONE — Dashboard data-source migration (`index.html` `fetchTlaData`) — shipped Rev 3.51–3.54 (2026-06-11/12)
**Resolved via the dao-dashboard cron** (a cleaner solution than the per-source adapter originally specced below). The new `dao-dashboard` cron assembles the DAO aggregates server-side into a legacy-compatible `{meta, dashboard}` shape, so `fetchTlaData` simply reads that one file (live-primary, 26h fresh-gated) with the legacy epoch walk-back as fallback. The Unclaimed Rewards / TLA Deposits / Lion tiles are now hourly-fresh instead of frozen at epoch 185. Deep-dive pages (`dao_treasury.html`, `dao_tla_deposits.html`) migrated the same way. Also shipped in this arc: cron-first instant paint (~9s→3-5s load), deving.zone fully eliminated from index, chart history revived past 185, heartbeat false-stale fix. Full detail in `index-log.md` Revs 3.51–3.54 and `cron-scripts/dao-dashboard/README.md`. The original per-source adapter plan is retained below for reference but is superseded.

<details><summary>(superseded) original per-source adapter plan</summary>

**Identified 2026-06-09.** The DAO Unclaimed Rewards + DAO TLA Deposits tiles are stuck (`--` / spinner). Root cause: `fetchTlaData()` (index.html ~line 9836) still reads the **dead** monolithic `tla_json_storage/main/tla-data-epoch-{N}-end.json` (404 for epochs ≥186). That old file bundled pools + DAO treasury + locks + balances + ratios in one blob; the **new architecture split it across 4 crons**, so this is a *routing* migration, not a URL swap.

**Old `tlaData.*` field → new source mapping (confirmed against live data 2026-06-09):**
| Old field(s) | New source | New path |
|---|---|---|
| `tlaData.pools`, `tlaData.vote.pools` | tla-snapshot cron | `tla-snapshot-data_2026/data/tla-snapshot.json` → `pools[]` / `buckets{}` / `totals{}` / `epoch{}` |
| `tlaData.dao`, `tlaData.locks(.individual_locks)`, `tlaData.totalDeposit`, `tlaData.tokenBalances` | adao-positions cron | `adao-positions-data_2026/data/current.json` → `treasury.{locks, lp_positions, wallet_balances, summary}` |
| `tlaData.vote` (rewards) | adao-positions + tla-snapshot | `treasury.{pending_rewards, pending_rebase, pending_bribes}` and/or `tla-snapshot totals.rewards` / `buckets[].rewards` |
| `tlaData.lstRatios`, `tlaData.ampRatios`, `tlaData.tokenPricesAtSnapshot` | network-and-prices cron | `network-and-prices-data_2026/data/network-and-prices.json` → `lst_ratios{}` / `token_prices{}` |
| `tlaData.snapshotDate` | any | new `capturedAt` |
| `tlaData.meta` (staleness) | rebuild | from heartbeat `dataFreshness` / `capturedAt` — **date-based now, not epoch-based** |
| `tlaData.dashboard(.alliances)` | TBD | needs an archived `tla-data-epoch-N-end.json` to confirm exact semantics, or reverse-engineer from consumers |

**Recommended approach (lowest risk):** rewrite `fetchTlaData()` as an **adapter** that fetches the 3 new sources and assembles an object matching the old `tlaData` contract, so the **12 consumer call sites stay unchanged** (lines 7159, 8272, 8546, 9330, 9836, 10065, 10307, 10483, 11134, 11229, 11297, 11385). Field mapping lives in one place.

**Hard requirements (per Camron, 2026-06-09):**
- **Remove the old fallback entirely** — delete the `tla_json_storage` epoch walk, the `tla-ext_json_storage` reads (`fetchTlaExtData`, ~line 9883), and the `epoch_1-300_date.json` ref (~line 9807).
- **Work-as-intended-or-error:** if a new source is unavailable, the tile shows an **error state** — never a stale snapshot or a silent default.
- Also retire the v3-format fallback block (~line 5020-5024).

**Caveats:** untestable from the sandbox (browser code in a 914 KB / 15k-line file). Test in-browser after. Obtain one archived `tla-data-epoch-N-end.json` if possible to nail `dashboard`/`dao` field semantics exactly. Per project rule, `index.html` data-layer changes only — don't touch render logic.

</details>

### ✅ DONE — NFT Explorer repoint (`nft-explorer-app.js`) — shipped 2026-06-09/10
deving.zone fully removed; explorer reads `data/v2/nfts.json` + canonical rarity files only, with hard-fail integrity gates (10,000-record check, owner-resolution check, no fallbacks). Details in the Rev 2 section below and `explorer-log.md`.



### 🔥 P1 — NFT Explorer page migration (Rev 2)
**Identified 2026-06-06. Rev B cron foundation shipped 2026-06-07. Explorer migration SHIPPED 2026-06-09/10 (items 1–7); item 8 (pending-claims surfacing) remains — per-record flag now carried through the merge, UI not yet built.** The `nft-inventory` cron Rev B now produces a full chain-of-truth replacement for `deving.zone/nfts/alliance_daos.json` (which has confirmed bugs: 16 missing DAODAO stakers, 54 undercounted, DAODAO contract itself listed as a 384-NFT user, no Atrium awareness, no Boost seller resolution). The explorer page still reads from deving.zone — Rev 2 swaps the data source.

**Affected file:** `nft-explorer-app.js` (237 KB main page logic)

**Changes needed:**
1. **[x]** Swap `STATUS_DATA_URL` from `deving.zone/nfts/alliance_daos.json` → our cron's `nfts.json` raw URL
2. **[x]** Adapt `mergeNftData()` to handle Rev B records[] format (schema v2) with new fields: `real_owner`, `listing{...}`, classification flags
3. **[x]** Replace dead `MEMBERS_CSV_URL` (`adao_json_storage/main/members.csv` — repo dead since 2026-05-17) with our cron's `summary.json` (richer data — per-staker counts + voting_power_pct)
4. **[x]** Add marketplace badges with prices: "Listed: 2,200 bLUNA ($1,875)" — BBL/Atrium/Boost icons
5. **[x]** Add backing display tile: collection-wide treasury value ($1.65M today) + per-NFT share (88.20 ampLUNA) + boost-mechanic story ("share grew +12.3% since launch as 1,093 NFTs broke")
6. **[x]** Add AbortController timeouts on all `.json()` fetches (deving.zone-hang lesson — same fix applied to `index.html` below)
7. **[x]** Add new badges/filters: "DAO Treasury" (898 broken), "Atrium Listed" (1), distinguish "Enterprise Staked" (403 real) vs "Enterprise DAO Broken" (100 gov)
8. **[ ]** Surface pending claims from `summary.daodao_pending_claim` (cron ships this as of Rev B.3): a global "N NFTs unstaked & pending claim" stat, and a per-wallet "You have N NFTs ready to claim" nudge when a viewed/connected address appears in `claimable[]`. Show `reconciled: false` defensively (render count, treat per-wallet detail as best-effort).

Estimate: 4-6 hrs. Verify cron data has run cleanly for 24+ hours first. Don't ship Rev 2 same-day as Rev B.

### 🟢 P2 — NFT Explorer Analytics tab: investor-grade expansion (spec'd 2026-06-10)
**Context.** Goal: stats an investor in a stock/token would expect, applied to the collection. The wishlist originally lived only in chat — this section is now canonical.

**Shipped 2026-06-10 (client-side, live data only):**
- **Supply screener** — the collection read like a token: Max 10,000 · Circulating (minted) 4,172 · Staked/DAO-controlled 3,049 (1,632 DAODAO + 14 pending + 403 Enterprise + 1,000 DAO broken) · Free float 1,054 + 48 listed; stacked supply bar.
- **Governance concentration** — Nakamoto coefficient (currently **4** wallets > 50% of staked VP), top-1/5/10 VP shares (19.9% / 57.9% / 68.1%), 157 stakers. VP = DAODAO-staked NFTs; broken keep VP.
- **Floor by tier** — Broken / Unbroken (base) / Phoenix rows: listed count, **listing floor** vs **sales floor** (median of recent tier sales, USD-at-sale) and the **spread** between them. Backing reference shown. Caveat noted in-UI: sales classified by *current* broken state. NOTE: the panel surfaced an apparent −84% base spread on day one, which turned out to be a cron-side ghost listing (see brief item 5) — real base floor ≈ $101 / spread ≈ −6%. Panel self-corrects when the resolver fix lands; the panel catching this is the point.
- **Floor-history chart (complete)** — sales bars (low->high + median) x 12W/12M x Broken/Base/Phoenix with ‹older/newer› paging through full Dec-2023->now history; historical cheapest-listing **USD-range band + mid line** from `listing-history.json` + daily oracles (stablecoin-denominated asks render flat — correctly); **LUNA price overlay** (own scale, toggleable); legend; tier classification **exact via `broken-at.json`** (109 previously mislabeled sales reclassified). Defensive liveness filter on open listing segments (ghost 14765).
- **Mark price & Market cap** — per-tier mark = mid(sales floor, listing floor); Mark column in Floor-by-tier; hero = Market cap (Σ tier mark × circulating) + FDV + Mark(base) + volume + highest sale ("Value today" removed).
- **Click-to-explain** — Market cap, Mark, Volume, Backing/NFT, Total backing, Supply, Nakamoto open methodology modals with live numbers substituted into the formulas. Nakamoto shows a 1–20+ zone scale with label.
- **Buyers/sellers** — 12-month ownership-trajectory line per trader (holdings reconstructed from marketplace trades), desktop-only center column; behaviour blurbs retained.
- Cache-busted asset URLs (`?v=5.x`, bump per release). Listing-floor overlay (incl. USD drift of standing listings) is the cron-side follow-up (listing backfill, brief item 0).
- Earlier this pass: matching-traits tooltip; analytics thumbnails moved to CDN-primary + IPFS-fallback (ipfs.io rate-limiting fix).

**Floor methodology — SETTLED:** sales floor = median of recent sales within the tier (Phoenix segmented out so trait skew can't pollute base), displayed *against* the listing floor as a spread rather than picking one "true" number. This unblocks per-wallet cost-basis P&L.

**Remaining (explorer-side, data already live):**
1. **[ ]** Pending-claims surfacing (migration item 8): global "N unstaked & pending claim" stat + per-wallet "ready to claim" nudge (`pending-claims.json` + per-record flag shipped; flag now carried through merge).
2. **[ ]** Per-wallet cost-basis P&L in Wallet tab: paid (from `sales-enriched` buys) vs backing vs tier sales-floor; "no basis" for non-marketplace acquisitions. Unblocked by floor methodology above.
3. **[ ]** Per-NFT provenance drill-down on card/modal (`nft-provenance.json` is 13 MB — fetch per-token on demand, never wholesale).
4. **[ ]** Backing growth story on the backing tile ("per-NFT share grew +X% since launch as 1,093 NFTs broke").

**Cron-side: ✅ ALL DONE 2026-06-11** — floor-history.json (daily per-tier listing+sales floors, DOM, bids), listing-first-seen.json, broken-at.json (1,093/1,093 break timestamps), listing-history.json (3,264 listings w/ outcomes 1,252 sold / 1,958 delisted / 54 active), BBL resolver fixes, and forward-fill in the incremental. Explorer can now: classify Broken-tier sales by `sale_ts` vs `broken_at` (remove the interim warning), chart real floor history, show DOM, upgrade `sales_tiering`.


### 🔥 P1 — Rarity system overhaul: explorer wiring + DAO proposal (spec'd 2026-06-10)
**Context.** Full investigation done 2026-06-10 in the NFT *Inventory* chat (this section is the handoff to the explorer chat). Rarity worked out from `all_nfts_metadata.json` + the HashLips design and reconciled against BBL's live marketplace API (HAR capture). Page + data files are shipped; what remains is the explorer wiring, an explorer bug fix, and the proposal.

**Two ranking systems — both canonical files live in `defipatriot/nft-metadata`:**
| | Intended (design) | BBL (marketplace) |
|---|---|---|
| File | ✅ `adao-rarity-intended.json` (committed) | ✅ `adao-rarity-bbl.json` (committed; built 2026-06-10 19:02 UTC, 1.28 MB) |
| Method | **Object trait alone defines the grade** (40 objects ↔ 40 grades, 1:1; metadata `Rarity` attribute = that grade). Grade order follows HashLips **planned weights**, not realized counts: Phoenix Rising planned 12 → Grade 40 apex (ranks #1–25) even though 25 minted vs Saber's 6. Grades laid end-to-end apex-first → 1–10,000 `intended_rank`; **within a grade tokens are equal-rarity by design**, ordered by token id as a lucky-draw tiebreaker (lower mint id = lower number; the grade is the rarity, not the within-grade slot). Planet/Inhabitant are flat by design (~500/value), Light/Weather are scene — **none are rarity factors** | Generic inverse-frequency sum over **every** attribute — including the derived `Rarity` tier (Object effectively double-counted) and the `broken`/`rewards` status. Result: realized-count order (Saber #1–6 above Phoenix #48–66) plus atmosphere leakage — the six `Weather: Lightning strike` tokens (common weapons) rank BBL #7–12 but intended #3,022–7,476 |
| Broken NFTs | All 10,000 ranked (grade is broken-agnostic) | Most broken NFTs come back unranked (`bbl_rank:null`); a small number still slip through with a rank — not fully consistent. Our cron faithfully records what BBL serves |
| Spot-check | #9068 (Phoenix) = grade 40, rank **24** | #9068 = rank **68** |

**BBL Action — DONE.** Weekly GitHub Action (`bbl-rarity.yml`) in `nft-metadata`, `bbl-rarity.js` 218 lines committed. Verified against the user's BBL text dump: **38/38 sample ranks match exactly** (incl. #242=1, #9068=68, #3937=443, #3021=444). File is 8,931 ranked + 1,069 unranked = 10,000. Hard-floor 8,500 captured + auto-fill BBL-unreturned tokens as `bbl_rank:null` (handles BBL's null-block pagination instability — confirmed: 411 broken correctly unranked + 658 unreturned-filled + 24 broken-but-ranked-by-BBL = 1,093 broken NFTs total, matches `nft-inventory` truth). 5 structural self-checks before any write (id universe, rank uniqueness, sums). Commit-on-change only → quiet weeks = no commit → file `built` date = "last time BBL ranks moved." Deliberately isolated: if BBL dies, delete the workflow + js + drop the toggle; zero blast radius.

**Rarity page — DONE.** `rarity-explained.html` shipped 2026-06-10 (replaces site-root page). Covers HashLips + planned weights, layer roles, designed grade ladder w/ planned-vs-actual, intended-rank construction including the lucky-draw within-grade framing, BBL divergences (apex inversion + weather example + broken-handling differs) with no hard-coded counts that'll age, full per-trait scoreboards (Object/Weather/Light/Planet/Inhabitant with planned where applicable), and a "Trait matches — the home system" section (P+I: 967, P+I+O: 80 — these are the *correct* numbers; see explorer bug below). References block links HashLips repo, the Terra-money Notion rarity doc, and both canonical JSONs.

**🐛 Explorer bug found while reviewing the rarity wiring (must fix before/with the wiring):** `nft-explorer-app.js` lines 166 and 180 (`PLANET_INHABITANT_MAP` and `PLANET_OBJECTS_MAP`) use the key `'Pampa'`. The metadata's Planet base name is `'Pampas'` (Pampas North / Pampas South). The strip-North/South regex returns `'Pampas'`, the lookup misses, and ~1,000 Pampas-planet NFTs are silently excluded from every matching-trait check. Symptom: explorer shows P+I=**864** instead of **967** and P+I+O=**74** instead of **80**. Two-character fix in two places — change both `'Pampa'` keys to `'Pampas'`. Verified by re-running both maps against the metadata; this is the *only* discrepancy — every other planet base, all 10 inhabitant species, and all object spellings reconcile exactly.

**Explorer changes — ✅ SHIPPED 2026-06-10** (staged on `nft-explorer-test.html`, promoted same day; verified live: Pampa fix bumped P+I 864→967 / P+I+O 74→80; #9068 = Rank 24 Intended / 68 BBL; 1,069 BBL-unranked render "Unranked"):
1. **[x]** Load both rarity files (`raw.githubusercontent.com/defipatriot/nft-metadata/main/adao-rarity-{intended,bbl}.json`); join to records by `token_id`.
2. **[x]** **Rank-system toggle** `BBL Rank / Intended Rank`, switching every rank shown. Display style: `Rarity 40, Rank 24` for Intended; `Rarity 40, Rank 68` for BBL — the grade stays visible in both; the *rank* is what switches. BBL + broken (`bbl_rank:null`) → render "Unranked," not 0 / blank.
3. **[x]** Small disclaimer line near the toggle when BBL is active: *"BBL ranks mirrored from BackBone Labs · last changed {file `built` date} · BBL leaves most broken NFTs unranked."*
4. **[x]** Display-option toggles — defaults ON: **Rank, Planet, Inhabitant, Object**; defaults OFF/hidden: **Weather, Light, Rarity** (the old `40/1`-style line is retired from the default card view).
5. **[x]** Filter dropdowns: still 4; **replace the Rarity dropdown with Rank** — filters by the 1–40 grade under the hood (a 10,000-option exact-rank dropdown is impractical; user-confirmed intent).
6. **[x]** **Sort By**: `Ranking, Rarity, ID`, default **Ranking High→Low** (best rank first, honoring the active toggle).
7. **[x]** Footer: **remove "Sorting Explained" and "Snapshot Tool" entirely.** Remaining: **Rarity Explained** (now a link to `rarity-explained.html`, not a modal) + **Badges Explained**.
8. **[x]** The explorer's internal sub-rank computation (rarityClass + Weather/Light tie-break, source of the old `40/1` display) is superseded — ranks come only from the canonical JSONs.
9. **[x]** Fix the `Pampa`→`Pampas` typo above (lines 166 + 180).

**DAO proposal (separate task, evidence-ready):** adopt Intended as the collection's official grading; ask Atrium to grade by it (file is open + verifiable at `raw.githubusercontent.com/defipatriot/nft-metadata/main/adao-rarity-intended.json`). Evidence ready: HashLips design intent (Object-only, planned weights, Phoenix apex), three named BBL divergences (apex inversion, weather leakage, inconsistent broken handling), the explorer toggle as the both-worlds bridge, full per-trait scoreboards already on the rarity page.

---

### 🔥 P1 — Vote/lock history backfill → now spec'd as `SPEC-tla-history-backfill.md` (2026-06-13)
Superseded/expanded: the lock-NFT backfill is now part of a broader **TLA history backfill** spec covering both lock events AND vote events (the differentiated Vote-Intelligence data). Same playbook as the NFT-provenance backfill. Lock contract `terra1uqhj8…` (create/merge/unlock + Boost activity) + gauge controller (vote changes). Key finding: EVENTS are backfillable via tx_search (permanent txs); VALUATIONS are not (pruned, forward-only). New repo `tla-history-data_2026`. First step: probe exact `wasm.action` names. See the spec doc for the full build plan. Start in a FRESH chat.

### ✅ DONE — adao-positions switched to daily (2026-06-13)
Render schedule `0 1 * * 1` → `0 1 * * *`. Daily P&L history now accumulating (`data/daily/{YYYY-MM-DD}.json`). (Also widened to all 156 members same day — see TLA Stats expansion section above.)

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
**Identified 2026-06-07 during Rev B design. Stage 1 SHIPPED as Rev C.1 (2026-06-07).** Rev B ran a single hourly full scan. Rev C splits into:
- **Hot** (every 15 min): user-held + marketplace + recently-unstaked (~1,100 NFTs)
- **Warm** (daily): hot ∪ staked (DAODAO + Enterprise, ~3,200 NFTs)
- **Cold/full** (weekly Mon 02:00): all 10k, full reconcile, rebuilds `hot-set.json`

**Stage 1 (Rev C.1) — DONE:** mode infrastructure via `RUN_MODE` env, `hot-set.json` membership file, scoped per-NFT fetch + `mergeRecords` onto the last full base, full-scan fallback. One script / three Render jobs. Default `full` = unchanged behavior. Deploy steps in cron README "Deployment (tiered modes)". Mode only changes per-NFT scope; Phases 3-7 identical, so output is always a complete 10k picture.

**Stage 2 — TODO (activity deltas):** each hot run diffs against the previous hot snapshot and appends what moved (transfers, list/delist, stake/unstake/claim, sales) to the day's activity log. This is what powers traffic/volume charts (net first-vs-last endpoint diffs would miss intra-day churn — must accumulate per-run deltas).

**Stage 3 — TODO (rollups):** finalize each daily file as opening snapshot + closing snapshot + accumulated activity; then aggregate daily→weekly (`weekly/<YYYY-Www>.json`)→monthly (`monthly/<YYYY-MM>.json`)→yearly (`yearly/<YYYY>.json`). Higher periods are rollups of the dailies, NOT fresh endpoint diffs. Forward-only (no backfill — LCDs prune).

Achieves ~50% query reduction vs the old full-every-hour, 4× freshness on hot data. Manual promotion to hot via `tla-chain-registry/curated/nft-overrides.json` when the DAO releases NFTs via prop; weekly cold reconcile is the safety net (max 7-day drift). **Page (Rev 2) reads the merged `nfts.json` regardless of which tier produced each record — schema is identical.**

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

- **NFT events backfill + forward-fill (2026-06-11)**: One-time sweep (`nft-events-backfill.js` + Action in the data repo) reconstructed `data/v2/broken-at.json` — **1,093/1,093 break timestamps, zero missing** (breaks are executed on the NFT contract, so capture is frontend-agnostic: Atrium-UI and Boost-UI breaks verified) — and `data/v2/listing-history.json` — **3,264 listings** back to Dec 2023 (BBL 3,121 / Boost 122 / Atrium 21) with derived outcomes **1,252 sold / 1,958 delisted / 54 active / 0 unknown** (sold = sales-enriched match, token-strict; delisted = provenance exit timestamp; no cancel-event needed). Forward-fill folded into the 6-hour incremental Action with per-stream watermarks (new breaks append; new creates append; active listings auto-close to sold/delisted). Same parsers serve both callers — one implementation, no drift.
- **NFT Inventory floor-history + first-seen + bids (2026-06-11)**: `data/v2/floor-history.json` — daily per-tier (broken/base/phoenix) row: listed count, listing floor, sales floor (median of last 5/10/3 enriched sales by `notional_usd`, n recorded), avg days-on-market, per-NFT backing USD, active bids. Same-date upsert, prior dates immutable, never-shrink guard. `data/v2/listing-first-seen.json` — DOM accrual from 2026-06-11 (Atrium `created_at` heights preserved for future precision upgrade). Full/warm runs only.
- **BBL listing-resolver fixes (2026-06-10/11)**: (1) Phantom listing excluded — chain-live-but-not-buyable auction 14765 set a fake $17.59 floor; warlock (BBL's own API) is now the liveness oracle: chain-only auctions excluded + warned, warlock-down ⇒ unfiltered + warned, never blanked. (2) Completeness — the contract's `auction_by_contract` cursor skips entries (mid-range holes, root cause unknown); 6 live listings recovered directly from warlock (`source:'warlock_recovered'`, denom/price byte-identical to chain). Verified live: 35/35 listings, base floor self-corrected to ~$102. Heartbeat canary: `listing_resolver_warnings`.
- **Data & pipeline registry + hardcode audit (2026-06-11)**: new section in `cron-scripts/README.md` — every producer → outputs → consumers with status labels, cleanup actions (stale `nft-inventory.js` orphan in the data repo; frozen pre-v2 `data/nfts.json`), and a classified hardcode inventory (IMMUTABLE / CONFIG / ASSUMPTION / STALE-PRONE with canaries). Rule it encodes: **one fact, one producer** — e.g. wallet names are owned solely by `adao-positions` (`members.json` via pfpk). Read it before building any new capture.

- **Rarity foundation (2026-06-10)**: Canonical rarity data + page shipped (explorer wiring still pending — see P1 above). `nft-metadata` repo now holds `adao-rarity-intended.json` (all 10k tokens: object/grade/planned+actual counts/intended_rank/percentile; #9068 = grade 40 rank 24) plus `bbl-rarity.js` + weekly GitHub Action producing `adao-rarity-bbl.json` (mirrors BBL's published ranks; commit-on-change only; broken NFTs faithfully `null`). `rarity-explained.html` Rev 2.0 rewritten around design intent (HashLips planned weights; Object-only grading; Phoenix apex; BBL weather-leakage example with real tokens). Key findings preserved in the P1 spec above.
- **NFT Inventory — staked-NFT staker resolution (2026-06-09/10)**: DAODAO + Enterprise stakes now resolve `real_owner` to the actual staker per token (phantom-whale fix — staking contracts no longer appear as top holders). DAODAO via `staked_nfts{address}` per staker (157, sums to 1,632 = exactly the DAODAO UI); Enterprise via `user_stake{user,limit}` per member with `total_user_stake` completeness check. `daodao_pending_claim` per-record flag (29 = chain truth; 15 untracked inferred pending, custody = active + pending definitionally); 81 Enterprise legacy stakes unattributable (abandoned contract, no reverse lookup) → flagged `enterprise_unattributed`, label "Enterprise (legacy, unattributed)". `dao_members_count` corrected 746 → **157** (DAODAO governance only; Enterprise ≠ DAO membership) with new `non_custody_holders_count` (746) for "anyone holding." Hard errors (query failure/truncation) flip status `partial`; known-incomplete-upstream stragglers are warnings, status stays `ok`. Verified live: error_count 0, classification sums to 10,000. Detail in cron README.
- **NFT Inventory Rev C.1 (2026-06-07)**: Tiered run modes, stage 1. `RUN_MODE` env (`full` default / `warm` / `hot`) scopes the per-NFT fetch only; Phases 3-7 run identically so output is always a complete 10k `nfts.json`. Full (weekly) rebuilds `hot-set.json`; warm (daily) re-fetches hot ∪ staked; hot (15 min) re-fetches the hot set — both merge fresh records onto the last full base, with full-scan fallback if base/hot-set unreadable. One script, three Render jobs (deploy steps in cron README). Merge/derive unit-tested; live cadence verified on Render. Stages 2 (activity deltas) + 3 (daily→weekly→monthly→yearly rollups) still to come. Gets 15-min fresh active data + ~50% query reduction.
- **NFT Inventory Rev B.7 (2026-06-07)**: Atrium listings schema-drift fix. `listings_by_collection` started 500'ing (`unknown field collection` — contract renamed the field). `fetchAtriumListings` now self-resolves the collection field name by probing common CosmWasm conventions (`collection_addr`, `nft_contract`, etc.), memoizes the winner, and logs the contract's full valid-field list if none match. No regression (Atrium NFTs already classified by ownership; this restores price/seller detail). Confirm via the `ℹ Atrium collection field resolved to '…'` log line on the Render run. This was the last known cron-side error — all three marketplaces + pricing + pending-claims now clean.
- **NFT Inventory Rev B.6 (2026-06-07)**: DAODAO pending-claim tx-search fix. LCD started rejecting the query (`400 "specify tx.height with strict equality"`) because it carried a `tx.height>` range; dropped the height term from the query and moved height filtering client-side in `fetchDaodaoTxs`. Restores forward per-wallet attribution tracking (count was always chain-truth; only the "who" was frozen). Parsers/reducer unchanged; logic re-verified (genesis replay → [1319,3605,6847,7123], incremental no-op, forward claim removal). Confirmed live: `lastScannedHeight` advanced 21353559 → 21355202. Detail in cron README.
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
