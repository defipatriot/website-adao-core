# TLA-CORE — STATUS & HANDOFF (audit as of 2026-06-25)

The single grounded reference for the tla-core cron pipeline. Written as a
handoff so a fresh chat can pick up without re-deriving anything. Pairs with the
canonical specs in `website-adao-core` (MISSION, PRICING-DOCTRINE,
TLA-CORE-STORAGE-DESIGN, SYSTEM-AUDIT-AND-OPS) — this doc does not replace those,
it tracks the *tla-core migration* specifically.

---

## 1. WHERE WE ARE RIGHT NOW (the live baseline)

Four things are live in `tla-core` and healthy:

| Module | Status | Notes |
|---|---|---|
| `fuel/` | ✅ live, correct | the reference SNAPSHOT module (`fuel/snapshots/…` + year rollups) |
| `catalog/` | ✅ live, working | address-catalog (WHO) — 389 addresses, self-contained (queries chain) |
| `contracts/` | ✅ live, working | contract-token-catalog (WHAT) — reads tla-snapshot (not self-contained) |
| `prices/` | ✅ live, **clean** | token prices only (27 tokens); LP/ampLP correctly removed |
| `docs/` | ✅ | epoch schedule (`epoch_1-300_date.json`) + `Staking APR.csv` moved in |

**This baseline is stable and correct.** Nothing here is broken. The work that
remains is consolidation + correctness-of-shape, not firefighting.

---

## 2. THE SETTLED DOCTRINES (do NOT re-derive — decided, in website-adao-core)

### Storage layout — `module / product / files` (TLA-CORE-STORAGE-DESIGN.md)
Every module is `tla-core/{module}/{product}/` with, always:
- `heartbeat.json` (schema v1: `cron, capturedAt, capturedAtUnix, runId, runMode, status, stats, next_expected_run_at`)
- `index.json` (manifest: what exists + latest pointer)
- SNAPSHOT product: `current.json`, `daily/`, `hourly/`, `{YYYY}/epoch-avg|monthly-avg/{YYYY}.csv`
- EVENT product: `cursor.json`, `{YYYY}/{MM}/{DD}.jsonl`
- `fuel/` = reference snapshot module; `flows/` = reference event module.
- Year rollover = a new folder, never a new repo.

### Pricing (PRICING-DOCTRINE.md) — already correct in network-and-prices
- Tier-1 (LUNA, wBTC, USDC, ATOM, ETH) → CoinGecko direct.
- Tier-2 (all LSTs + small caps: ampLUNA, arbLUNA, bLUNA, ampCAPA, ampROAR, xASTRO)
  → **base price × on-chain ratio** (`calculated-eris`). Thin pools FLAG, never override.
- This is proven (arbLUNA derived $0.1548 ≈ CoinGecko $0.1523). **Preserve it; don't rebuild.**

### LP / ampLP valuation — share-based, NOT per-unit price
- Positions value by **`staked_shares / total_shares × pool_usd`** (the `adao-positions`
  method that matches Eris and produces the treasury's $7,593.66).
- A per-unit ampLP "price" is the WRONG model — `tla-snapshot.amp_lp.shares` is
  inconsistent across pools and cannot be a divisor. (Proven this session against
  the TLA UI screenshot: per-unit pricing was 1.7×–3.9× low, varying per pool.)
- → LP/ampLP valuation belongs in the **positions/aDAO-Data module**, per wallet.

### Migration philosophy (TLA-CORE-STORAGE-DESIGN §5)
Repoint a proven cron's output to `tla-core/{module}/{product}/`, **run parallel**
with the old repo, verify (incl. system-health heartbeat path), repoint site
consumers, freeze the old `*-data_2026` repo read-only, delete after cooling.

---

## 3. THE ONE REAL DEFECT IN WHAT WE BUILT

The three new crons write **module/files**, missing the **product level** and
`index.json`:

```
WRONG (current):                  RIGHT (per the design, like fuel):
  catalog/current.json              catalog/snapshots/current.json
  catalog/heartbeat.json            catalog/snapshots/heartbeat.json
                                    catalog/snapshots/index.json
```

Same for `contracts/` and `prices/`. **Fix = realign all three to
`{module}/{product}/` + add `index.json` + the full heartbeat schema.** Low-risk,
mechanical. `fuel/` is the copy-from reference. (system-health `MONITORED` paths
must update to the new heartbeat locations when this changes.)

---

## 4. THE INTENDED ARCHITECTURE (the consolidation target)

Collapse ~18 crons + their `*-data_2026` repos into a small set of
**self-contained domain crons** (each queries the chain directly; no reading old
output; old repos get deleted). Names settled with Camron:

| Domain cron | Absorbs (lift code from) | Role |
|---|---|---|
| **`address-catalog`** ✅ exists | tla-registry's address side: known_contracts, wallets, protocols, directory | WHO |
| **`token-catalog`** (rename of price-cron) | network-and-prices (pricing+ratios) + tla-registry (token identity: logos, decimals, categories) | WHAT each token is + worth |
| **`DEX-Data`** (new) | tla-snapshot (lp_health/amp_lp/buckets) + astroport + skeletonswap + LP/ampLP denom work | pools, reserves, slippage, position valuation |

Then **deleted:** network-and-prices, tla-snapshot, tla-registry, astroport,
skeletonswap, the interim contract-token-catalog, and their data repos.

**Important nuance discovered this session:** `tla-chain-registry` (cron
`tla-registry`, output `defipatriot/tla-chain-registry/2026/current.json`) is a
LIVE, comprehensive token/contract/pool registry — 173 tokens with logos +
decimals + categories, plus pools/contracts/directory + curated overrides. It
**overlaps** the interim contract-token-catalog. Its token identity + logos are
what `token-catalog` should absorb (logos: 1/token, 2/pair via token_a/token_b;
source priority curated → cosmos-chain-registry → skeletonswap; 36/173 have logos,
rest letter-circle). Do NOT repoint it (that keeps it alive) — **lift its code.**

> ⚠️ Constraint: the sandbox cannot reach Terra RPC/LCD. So building these means
> lifting the PROVEN functions mostly intact + rewiring output to tla-core; Camron
> runs on Render to verify. Build ONE at a time, run parallel with the old, prove,
> then unplug. This is why parallel-run matters.

---

## 5. tla-flows — BUILT, LOCALLY-VERIFIED, NOT DEPLOYED TO tla-core

`cron-scripts/tla-flows/tla-flows.js` is the EVENT-pattern reference cron
(`tx_search` capture of TLA LP deposit/withdraw/claim/zap). It watches the
compounder, 4 staking buckets, and the zapper, writing
`events/{heartbeat,index,cursor}.json` + `events/{YYYY}/{MM}/{DD}.jsonl`.

- It was **verified locally against 42+6 real on-chain txs** (incl. the test
  wallet `terra1n28qcuxlm0t94dlky2zny0g7w8vrrklgef7229`'s wBTC-LUNA zap →
  21,479 amplp).
- BUT **no `flows/` data exists in tla-core main** (all candidate paths 404).
  The Render deploy hasn't landed data.

**→ This blocks the "verify 24hr position change from the test transactions"
request.** There's no captured event data in tla-core yet to compare against.
**Next-chat action:** confirm the tla-flows Render service exists, is pointed at
`tla-core` with the flows output path, runs, and commits — then the test txs land
and 24hr position deltas become checkable.

---

## 6. PRIORITIZED NEXT STEPS (the grind-out plan)

Do these in order; each is self-contained and parallel-safe.

1. **Realign the 3 live crons to `module/product/` + `index.json`** (§3). Mechanical,
   low-risk, makes the foundation spec-correct. Update system-health paths.
2. **Deploy tla-flows to tla-core** (§5) — get the event module actually live so
   the test txs land. Then verify the 24hr position change.
3. **Build `token-catalog`** (rename price-cron; lift network-and-prices pricing +
   tla-registry token identity/logos). Run parallel with network-and-prices;
   confirm identical token prices; then retire network-and-prices repo.
4. **Build `DEX-Data`** (lift tla-snapshot + astroport + skeletonswap; pools,
   reserves, the share-based LP/ampLP position valuation, slippage data for the
   future simulator). Run parallel; prove; retire.
5. **Fold tla-registry's address side into address-catalog**; retire tla-registry +
   the interim contract-token-catalog.
6. **Delete old repos** once each consumer is repointed (freeze → cool → delete).

After 1–2 the foundation is spec-correct + flows is live. 3–5 are the real
consolidation, one cron per sitting.

---

## 7. OPEN QUESTIONS FOR CAMRON (decisions the next chat needs)

- **tla-flows Render status:** does the service exist + point at tla-core? (blocks §5/step 2)
- **Real Render schedules** for the existing crons (SYSTEM-AUDIT marks these
  [NEEDS CAMRON]) — needed so dependents don't run before their sources refresh.
- **Consolidate vs repoint per cron:** the aggressive lift-and-merge (§4) vs the
  simpler 1:1 repoint. Recommendation: lift-and-merge for the 3 foundation domains
  (token-catalog, DEX-Data, address-catalog) since they're high-value and overlap;
  1:1 repoint the rest opportunistically.

---

## 8. REFERENCE (carry-over facts)

- **Token rollover:** SOLVED — Camron made one no-expiry token, plugged into all
  tla-core crons. No Nov-2026 rollover needed for tla-core.
- **Sandbox network:** github + raw + api.github + npm/pypi only. No Terra RPC/LCD.
  Read committed files via codeload tarball (bypasses CDN cache).
- **Workflow:** full-file replacements only; Camron commits via GitHub web UI;
  Claude produces files. Verify heartbeat paths against production before trusting.
- **Compounder:** `terra1zly98gvcec54m3caxlqexce7rus6rzgplz7eketsdz7nh750h2rqvu8uzx`
  (mints `factory/<compounder>/<N>/<bucket>/amplp`; amplp = amplified positions;
  raw cw20 LP = non-amplified). Staking buckets: stable `terra1v399…`, project
  `terra1awq6…`, bluechip `terra14mmv…`, single `terra1qdz5…`. Zapper `terra1qdjs…`.
- **Treasury TLA deposit ground truth (TLA UI, this session):** $7,593.66 total;
  per-pool deposit/amplify values captured in the screenshot for valuation checks.
- **Mission (north star):** (1) aDAO mandate — Terra-inflation-funded, build
  impactful tools; (2) Allies — tools + name-resolution for partner communities;
  (3) LP grading oracle — accuracy is the moat. Onboarding = config, not code.
- **On the horizon (specs in website-adao-core):** Portfolio Tracker (assembler
  built, $7,508 verified), AI Assistant (needs serverless key-proxy), NFT
  Onboarding Blueprint, LP Grading (8 dimensions), epoch-boundary capture, bribe
  intelligence, history backfill, slippage simulator (DEX-Data consumer).

---

*End of handoff. The baseline is solid; the path is: make the 3 crons
spec-correct → land flows → build token-catalog → build DEX-Data → retire old.
One at a time, parallel-run each, prove before unplugging.*
