# Design — Full TLA snapshot cron automation

> **Status:** Inventory ready. Phase planning ready. Implementation begins next session.
> **Created:** 2026-05-10
> **Goal:** retire `tla_tool.html` + `tla-tool_ext.html` as manual capture tools. Replace with a Vercel cron job that produces equivalent JSON snapshots daily, with the existing tools kept as a fallback for rare cases.

---

## TL;DR

The snapshot file (`tla-ext-epoch-{N}-end.json`) has **9 top-level sections** with ~50 distinct fields. Most are derivable from on-chain queries; a few from external APIs that already work (CoinGecko, the Skeleton CSV repo); a few currently come from manual paste and need either a chain replacement OR a documented "still needs human" status.

**This is a multi-session project.** Don't try to ship it all at once. Phase order matters because some fields depend on others (e.g. `dex_performance` is computed from `lp_registry` + `astroport_data` + `skeleton_swap_data`).

---

## Field-by-field automation inventory

Source classification:
- 🟢 **Already automated** — works today, port to cron straightforward
- 🟡 **Chain query needed** — investigated, contract+query known, just needs implementation
- 🟠 **API exists** — third-party API works but flaky; needs retry/fallback
- 🔴 **Manual today, chain replacement TBD** — need investigation
- ⚫ **Unavoidably manual** — accept human input or find a creative substitute

### `meta` section
| Field | Status | Source |
|---|---|---|
| `generated_at` | 🟢 | `new Date().toISOString()` at cron run time |
| `epoch` | 🟢 | Already computed from `tla_json_storage/epoch_1-300_date.json` schedule + current date |
| `phase` | 🟢 | "end" (cron always captures end-of-epoch state) |
| `source` | 🟢 | "cron" (literal string, distinguishes from manual tool exports) |

### `votion` section
| Field | Status | Source |
|---|---|---|
| `ratios.arbLUNA` / `ratios.ampLUNA` | 🟢 | Already auto-fetched in `tla-tool_ext.html` via Eris hub contracts (`fetchLstRatios`) |
| `total_vp` | 🟡 | Query `terra1uqhj8agyeaz8fu6mdggfuwr3lp32jlrx5hqag4jxexde92rzkamq3l62zg` (vAMP minter) → `{total_vamp:{}}` returns `{vp: "..."}` |
| `pools[]` | 🟡 | Built from `gauge_infos` queries (see lp_registry below) |
| `lockups[]` | 🟠 | Eris's `backend.erisprotocol.com/votion/.../optimization` returns the optimization data we need. Intermittent 500s; new retry logic shipped May 10 helps. **For cron: also retry with exponential backoff over minutes if needed — cron has time budget the UI doesn't.** Manual paste is current fallback. |

### `lst_ratios` section
| Field | Status | Source |
|---|---|---|
| `snapshot_date` | 🟢 | timestamp at run time |
| `chain_staking_apr` + date | 🟢 | Already auto-fetched from `tla-ext_json_storage/Staking APR.csv` |
| `tokens.ampLUNA` ratio | 🟢 | Already auto-fetched from `terra10788fkzah89xrdm27zkj5yvhj9x3494lxawzm5qq3vvxcqz2yzaqyd3enk` (Eris hub) |
| `tokens.arbLUNA` ratio | 🟢 | Already auto-fetched (Eris arb hub) |
| `tokens.bLUNA` ratio | 🟢 | Already auto-fetched (Backbone Labs hub) |
| `tokens.ampCAPA` ratio | 🟢 | Already auto-fetched (Capa Eris hub) |
| `tokens.ampROAR` ratio | 🟢 | Already auto-fetched (Lion DAO Eris hub) |
| `tokens.xASTRO` ratio | 🔴 | **Currently manual paste.** Need to find Astroport's xASTRO/ASTRO ratio query. Probably one chain query against a single contract. |
| `tokens.{any}.apy` | 🟠/🔴 | Some are computed from chain rewards (chain_staking_apr × ratio), others are manual paste from Eris/Astroport UIs. **Audit needed:** which APYs are chain-derivable vs which require Eris's calculated number? |

### `token_prices` section (23 tokens)
Existing classifications by source:
| Source | Tokens | Status |
|---|---|---|
| `coingecko` | LUNA, USDC, USDT, ATOM, ASTRO, CAPA, EURe, INJ, ROAR, PAXG, SOLID, stATOM, stLUNA | 🟢 Already automated |
| `bridged` | wBNB.axl, wBTC.atom | 🟢 Already automated (mapped to base CoinGecko entries) |
| `calculated` | ampLUNA, arbLUNA, bLUNA, boneLUNA, ampCAPA, ampROAR, xASTRO | 🟢 Already automated (LUNA price × LST ratio) |
| `lcd-pool-derived` | FUEL | 🟢 Shipped May 9 2026 (LUNA-FUEL pool reserves on chain) |

**Future additions to watch:** Creda-DEX tokens like `wBTC.creda.a`. Will need a Creda-specific price source — could be `lcd-pool-derived` from the Creda pool contract once we know its address. Currently registers in `lp_registry` but `token_prices` won't have it.

### `lp_registry` section (43 entries) — THE CENTERPIECE
Currently built from manual paste of the Eris Vote tab.

**Replacement (HAR capture confirmed):**
- Query gauge controller `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj` four times: `{gauge_infos: {gauge: "stable" | "project" | "bluechip" | "single", time: "next"}}`
- Each returns `[[poolId, {voting_power, fixed_amount, slope}], ...]`
- Pool IDs come prefixed: `cw20:terra1xxx...` or `native:factory/terra1xxx.../uLP`
- Per-entry fields needed: `lp` (pair name), `dex`, `bucket`, `vote_pct`, `vp_amount`, `depth`, `active`, `single`

**Pool-id-to-name resolver** is the hard part. Three resolver strategies in priority:

1. Try `{pair_info: {}}` query against the LP token contract (works for Astroport CW20 pools, gives both token addresses, derive name from those)
2. If that fails, try `{token_info: {}}` (works for single-sided tokens like ampCAPA, xASTRO)
3. If that fails, log warning and skip this entry — manual fallback for rare new pools

**Cache aggressively.** Pool addresses change rarely. Persist resolver cache to `localStorage` (browser) or to a `pool_id_resolver.json` file in the storage repo (cron). 30-day TTL.

**Depth field:** the manual paste captures `Depth $ X` as displayed in the UI. On chain, depth = pool TVL = sum of token amounts × prices. We can compute this ourselves once we have the pool's `pool` query result + token prices. **Note:** for the live web tool we already have `astroport_data[*].spotLiquidity` which IS the same number — Astroport's `pools.byAddress` endpoint returns it. So `depth` in `lp_registry` can either be computed locally OR pulled from Astroport's API.

### `pd_bribes` + `pd_bribes_master` sections
Currently manual paste of Phoenix Directive proposal posts.

**Replacement (HAR capture confirmed):**
- Query incentive manager `terra1tuuwm8yrj54qeg0c8xu00aha9ryatyhtczq8qq2q8tntuw0auzas9037wh` → `{bribes: {period: "next"}}`
- Returns `{buckets: [{gauge, asset: {cw20|native: ...}, assets: [{info, amount}]}, ...]}`
- The `asset` field is the pool being bribed; `assets` is what's offered as the bribe (often LUNA or other tokens).

**Critical caveat:** the chain returns the **current state of bribes** — what's actually been deposited. The manual paste comes from the **proposal text** which describes intent. These can differ:
- A proposal might say "$100/epoch for 4 epochs" but only 3 epochs have been funded so far
- Mid-epoch top-ups (the launch incentives case) are visible on-chain but the proposal might be older

**For cron:** capture the chain state, but ALSO keep the existing `pd_bribes_master` as a sanity-check overlay. The new PD bribes editing UI we shipped May 9 already supports merge — that becomes the audit trail.

`pd_bribes` (current epoch only) and `pd_bribes_master` (full history) — chain only directly answers "current epoch". For historical, we either:
- Query `{bribes: {period: <past_period_number>}}` for each epoch (expensive)
- Keep the historical record in `pd_bribes_master` as we already do (cheaper, additive)
- Recommend: keep historical as-is, replace `pd_bribes` (current) with chain query

### `dex_performance` section (43 entries)
Computed field. Combines `lp_registry` + Astroport historical + Skeleton historical to produce per-pool epoch averages.

**No new source needed** — it's a derivation from other fields. Once `lp_registry` and the DEX data sources are reliable, this rebuilds correctly. Logic to port from `tla-tool_ext.html`: `buildDexPerformanceForExport()`.

### `astroport_data` section (41 entries)
Currently uses Astroport's TRPC `charts.{liquidity,volume}` endpoints + `pools.byAddress` for spot liquidity.

**Status mixed:**
- 🟢 `pools.byAddress` (spot liquidity) — works fine
- 🟠 `charts.liquidity` and `charts.volume` (historical D7/D30/D90) — `D90` enum was deprecated May 2026; `D7`/`D30` still work. The May 9 `selectBestAstroRange` fix handles this.

**For cron:** keep using Astroport's TRPC. It's external but it's their official API. Same retry+backoff pattern as Votion. If Astroport ever takes the TRPC down entirely, fallback options are:
- The unused `astroport-pool-data_2026` repo we never wired up
- Computing time-series from chain by polling pool state at intervals (very expensive — only viable for cron, not tool)

### `skeleton_swap_data` section (40 entries)
Currently fetches CSVs from `defipatriot/ss-pool-data_2026/data/weekly-avg/2026-epoch-{N}.csv`.

**Status:** 🟢 Already automated — this is a GitHub-hosted CSV that someone (you or an upstream cron) maintains. Just port the fetch to the new cron.

**Caveat from May 9 capture:** the tool was loading old epochs (168, 169) instead of current. The "epoch unknown" bug. Logged in CHANGES_PENDING. Cron must compute the correct current epoch from the schedule file before fetching.

---

## Summary by status

| Status | Count of fields | Notes |
|---|---|---|
| 🟢 Already automated | ~30 | Lift-and-shift to cron |
| 🟡 Chain query known, needs implementation | ~5 | Vote tab fetcher (4 queries), total_vamp (1) |
| 🟠 External API works but flaky | 2 | Votion `/optimization`, Astroport TRPC |
| 🔴 Manual today, replacement TBD | 3 | xASTRO ratio, some LST APYs, PD bribes (chain query exists but proposal-vs-actual reconciliation) |
| ⚫ Unavoidably manual | 0 | — none identified, every field is at minimum chain-queryable |

**Real conclusion: full automation is feasible.** No field requires unavoidable human input. The "🔴 manual today" items just need investigation and are 1-2 contract queries each once we find them.

---

## Phased implementation plan

### Phase 1 — Vote tab + LP registry (Session 1, ~3-4 hrs)
**Pure-JS module** `voteTabFetcher.js` (or inline in tool first, extract later). Same code runs in browser tool AND in Node cron.
- 4 `gauge_infos` queries against gauge controller
- Pool resolver with Astroport `pair_info` (Phase 1.A) and CW20 `token_info` fallback (Phase 1.B)
- Build `lp_registry` entries identical to today's shape
- Wire into tool as a "Fetch from Chain" button next to existing "Parse Vote Tab"
- Manual paste fallback unchanged

**Acceptance:** for the same epoch, chain-fetched `lp_registry` matches manual-paste `lp_registry` for ≥95% of entries (deltas should be only depth-rounding, not pool count).

### Phase 2 — Resolver cache + multi-DEX (Session 2, ~3-4 hrs)
- localStorage cache for resolver
- Investigate Skeleton Swap pool schema (does `pair_info` work? alternate query?)
- Investigate Creda pool schema (e.g. wBTC.creda.a)
- Total VP query (small win, drops in trivially)
- "Clear pool resolver cache" admin button

### Phase 3 — PD bribes from chain (Session 3, ~2-3 hrs)
- `bribes: {period: "next"}` query
- Map asset addresses to pool names using Phase 2's resolver
- Replace `pd_bribes` (current epoch); leave `pd_bribes_master` alone
- Reconcile differences between proposal-paste and chain-state (the merge UI we shipped May 9 helps here)

### Phase 4 — xASTRO + LST APYs (Session 4, ~2-3 hrs)
- Investigate xASTRO/ASTRO ratio source on Astroport
- Audit which LST APYs are chain-derivable vs require Eris's calculated value
- Replace any remaining manual entries with chain queries

### Phase 5 — Cron migration (Session 5, ~3-4 hrs)
This is where browser-only assumptions get factored out.
- Extract data-fetching from `tla-tool_ext.html` into a pure Node module
- Vercel serverless function that runs daily, writes `tla-daily-{date}.json` to `tla_daily_storage` repo (or directly to the existing `tla-ext_json_storage`)
- The browser tool stays as a fallback / debugging path (manual capture still possible)
- Dashboard tiles point at the daily file (eventually); the existing snapshot system can run in parallel for verification

### Phase 6 — Verification + cutover (Session 6, ~2-3 hrs + 2-3 weeks soak)
- Run cron alongside manual captures for 3 weeks
- Diff each cron output against the manual capture
- Fix discrepancies
- After 3 clean weeks, cutover: dashboard reads cron output, manual tool deprecated

---

## Key contract addresses (Terra phoenix-1)

| Role | Address |
|---|---|
| TLA Gauge Controller | `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj` |
| TLA Incentive Manager | `terra1tuuwm8yrj54qeg0c8xu00aha9ryatyhtczq8qq2q8tntuw0auzas9037wh` |
| vAMP Minter | `terra1uqhj8agyeaz8fu6mdggfuwr3lp32jlrx5hqag4jxexde92rzkamq3l62zg` |
| Eris ampLUNA hub | `terra10788fkzah89xrdm27zkj5yvhj9x3494lxawzm5qq3vvxcqz2yzaqyd3enk` |
| LUNA-FUEL Astroport pool | `terra10yfnsqn20rzlnlzkeva5255q27zp6ws9te9uuql9e0lacfcze7zsffjct5` |
| Votion arbLUNA Max contract | `terra13aae4futz6jk7hmdv0gwm2xs6p4nxv4xwz5tc0c2vt4960u4j6jqpqmye9` |

Full list lives in PROJECT_KNOWLEDGE.md "Key on-chain contract addresses" — append to that as we discover more.

---

## Architectural principle

**The tool and the cron should share data-fetching code.** Don't build two implementations. Approach: write each fetcher as a pure async function that takes minimal config and returns its slice of the snapshot, with no DOM or browser dependencies. The browser tool wraps it in UI; the cron wraps it in a serverless handler.

Practical: put fetchers in `<script type="module">` blocks or refactor into a shared module. Pure JS, no Tailwind/DOM in the data path.

---

## Open questions for next session

1. **Cron host:** Vercel serverless function (already used elsewhere) is the obvious choice. Confirm.
2. **Output destination:** new `tla_daily_storage` repo for the daily file, or write directly into the existing `tla-ext_json_storage`? Keeping them separate during the soak period is safer.
3. **Failure mode:** if any one section's fetch fails, does the cron still write a partial snapshot, or does it abort and the previous day's file remains as fallback? **Default:** write partial with explicit `{section: { error: "..." }}` markers, so the dashboard can degrade gracefully.
4. **First implementation target:** Phase 1 = Vote tab fetcher in browser tool only. Does NOT touch cron yet. Cron migration is Phase 5. Keep early phases low-risk.
5. **Verification:** how do we compare cron output vs manual for parity? Suggest a tiny diff-script that runs both and reports field-level mismatches; saves hours of manual eyeballing.

---

## What this design doc replaces

The earlier scoping called this "on-chain Vote tab fetcher" — that's still Phase 1, but the goal is the full snapshot. Don't re-scope down to just Vote tab in implementation; treat each section as a milestone toward the same destination.
