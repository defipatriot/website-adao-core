# TLA Data / Cron Brief — Fixes & Additions

**Purpose.** The `tla-stats.html` dashboard is now defended against the data problems we found, but those are *page-side band-aids*. The durable fixes live in the crons. Captured data can't be recreated after the fact, so anything we want to show historically must start being captured **correctly and completely now**. This brief is the full punch list, ordered by impact, with the *why* so the cron work can be done with the same care.

The guiding principle that emerged this session: **a number is only as trustworthy as the rule it must obey.** Where a value violates a structural invariant (a constant-product pool must be ~50/50 by value; Votion must be zero-sum within a bucket; a gauge lives in exactly one bucket), that violation is the signal something upstream is wrong. The crons should enforce/record these invariants, not just emit raw values.

---

## PART 1 — Correctness fixes (things currently wrong or fragile)

### 1.1 [HIGH] Key everything by `gauge_pool_id`, not pool name — kills the whole variant-collision class
**Problem.** A pool pair can have two gauges that share the same `name + dex + bucket`: the real active one and a dead `voted_but_below_threshold` leftover. Separately, the *same* `gauge_pool_id` has appeared split across two buckets. Because `apr-history`, `pool-status-history`, and the Votion/aDAO attachment are keyed by **name** (or name+dex), data lands on the wrong variant. This produced, on the page: +41,884% VP deltas, phantom Votion bars larger than the pool, double-counted bucket totals, inflated inactive groups, and wrong DEX/depth values. We added two page-side safeguards (demote duplicate name+dex+bucket variants; collapse a gauge that appears in >1 bucket) — but they're defensive.
**Fix.** Make `gauge_pool_id` the canonical join key everywhere a cron writes per-pool history or attaches Votion/aDAO/bribe data. `gauge_pool_id` is truly unique; `name`/`name+dex` is not.
**Capture going forward.** Ensure every per-pool record in every repo carries `gauge_pool_id`. Where a pool legitimately has two gauges (active + below-threshold), keep them as distinct rows keyed by `gauge_pool_id`, never merged by name.
**Invariant to enforce in the cron:** within a single snapshot, a given `gauge_pool_id` appears in exactly one bucket, and a given `name+dex+bucket` has at most one `active` gauge. Log a warning if violated (the data has changed shape) rather than silently emitting it.

### 1.2 [HIGH] LST price source — use redemption rate, confirm it's right, and label it
**Status: the prices are actually CORRECT (within ~1% timing).** We initially suspected ampLUNA/arbLUNA/bLUNA prices were wrong because their pools show a ~62/38 value split. They are NOT wrong — these are *accruing* LSTs that redeem for >1 LUNA (arbLUNA ≈ 2.92 LUNA, ampLUNA ≈ 2.14 LUNA, bLUNA ≈ 1.75 LUNA), and their pools use a weighted/concentrated curve, so an uneven value split is correct by design. Verified against on-chain UIs.
**What to do.** Confirm the cron derives LST prices from the **redemption/exchange rate** (or the pool's own ratio), not from a generic market-price lookup that assumes 1:1. Stamp each LST price with its source and the redemption ratio used, so future audits can tell "correct uneven split" from "broken price" instantly.
**Capture going forward.** For every LST, store: `redemption_ratio` (LST→underlying), `underlying_sym`, and the `price_source`. The page can then label LST pool splits as "uneven by design" with confidence rather than heuristic guessing.

### 1.3 [MED] Reconcile `depth_usd` vs `staked_in_tla_usd`
**Problem.** For LUNA-INJ ($71.3K staked vs $66.6K depth) and wstETH-WETH.axl ($1.65K vs $0.84K), TLA-staked reads *higher* than total DEX depth — impossible literally, so it's a measurement/timing/price-source mismatch between the two fields (~7% on LUNA-INJ). Not catastrophic, but if the leaderboard shows "DEX liquidity" lower than "TLA staked," it looks wrong to a user.
**Fix.** Pin both to the same price source and the same capture instant. Document which is authoritative.
**Invariant:** `staked_in_tla_usd <= depth_usd` (allowing a small tolerance). Log when violated.

### 1.4 [MED] Skeleton Swap weekly-avg — epoch LABEL off-by-one (data IS fresh)
**Updated finding.** The SS cron is NOT broken — its heartbeat is healthy (runs daily, ~34 pools, ~$676K TVL, fresh fingerprint) and it IS writing weekly-avg files. The problem is the **epoch label**: the SS cron names its weekly file by the CURRENT (in-progress) epoch, while astroport and the page label by the COMPLETED epoch. So for the week of 2026-05-18..05-24, astroport writes `epoch-186` but SS writes `2026-epoch-187.csv`. The page asked for `epoch-{lastCompleteEpoch=186}.csv` → 404 → fell back to 185 → 404 → no SS data, even though `epoch-187.csv` (the right week's data) was sitting right there. Verified the CSV is fresh and its pool_names match the page's SS pool names.
**Fix (durable, cron-side).** Align the SS cron's epoch label with the rest of the system: name the weekly file by the COMPLETED epoch it covers (so the 05-18..05-24 week is `epoch-186`, matching astroport). This is the same epoch off-by-one class fixed in the other crons — apply the `currentEpoch = epochIndex + 1` vs internal-index discipline so the *file label* matches the canonical completed-epoch number.
**Page side (done as a stopgap):** the weekly-avg fetch now tries `lastCompleteEpoch`, then `+1` (SS convention), then `-1`, so it finds the data regardless. SS Epoch Avg Liq/Vol now populate. Remove the page +1 fallback once the cron label is aligned.

### 1.5 [LOW] Votion cron is stale (6 days as of audit)
**Problem.** `votion-data_2026` last wrote 2026-05-24; per-epoch `votion-epoch-187.json` 404s. **No user impact** — the waterfall reads Votion from `tla-snapshot.json`'s embedded `votion_current_vp/votion_optimized_vp` (refreshed hourly by the snapshot cron), and the per-epoch fetch fails gracefully to null. But the standalone Votion repo is stale for any other consumer.
**Fix.** Restart the Votion cron. Low priority for this dashboard specifically.

### 1.6 [LOW] fuel-data has no heartbeat file
**Problem.** `fuel-data_2026` produces valid `index.json` price data but has no `heartbeat.json`, so the footer cron-health check can't show its freshness.
**Fix.** Emit a heartbeat alongside the fuel snapshot, matching the schema the other crons use (`schemaVersion, cron, capturedAt, capturedAtUnix, runId, runMode`).

---

## PART 2 — New capture (data we need going forward and can't recreate later)

These unlock dashboard features we *want* but currently can't build honestly because the data isn't being captured. Start capturing now so we have history when we build them.

### 2.1 [HIGH] Per-pool aDAO + member VP at the epoch boundary (in the daily snapshot)
**Why.** The daily boundary snapshot (`tla-snapshot-data_2026/data/daily/{Sunday}.json`) currently captures Votion split + pool totals, but NOT how much of each pool's locked-in VP came from the **aDAO treasury** vs **individual members**. Without it, the Vote Breakdown waterfall can only project the *Votion* portion forward; it can't attribute the lock→now total change to users vs aDAO vs members. That's the single biggest "we can only show half the story" gap.
**Capture.** In the boundary daily file, per pool (keyed by `gauge_pool_id`): `total_vp`, `votion_vp`, `adao_vp`, `member_vp` (and ideally the active/inactive status *at the boundary*). This makes the locked-in view fully attributable.

### 2.2 [HIGH] VP-holder / concentration capture (enables the "Control of TLA" panel)
**Why.** We want to show how concentrated voting power is (who controls TLA), but we don't capture per-holder VP. This is the biggest new feature and needs its own cron.
**Capture.** A new `tla-vp-holders-data_2026` repo. Feasible on-chain via the voting-escrow CW721: paginate `all_tokens` → `owner_of` each → `user_info` per wallet → aggregate VP by owner. Store per-epoch: `{ owner, vp, lock_end, n_positions }` plus derived concentration metrics (top-1/5/10 share, Gini/HHI).
**Honest limitation to record:** multi-wallet clustering (one entity, many wallets) **cannot** be proven on-chain. Capture raw per-wallet only and label the panel accordingly — never imply wallet count = distinct people.

### 2.3 [MED] Bribe history (per-pool, per-token, per-epoch)
**Why.** Bribes are a fixed *token amount* whose USD value moves with price. Today we only have the current snapshot, so we can't show whether a bribe went up/down, or separate "more tokens added" from "token price moved." The page already labels bribes as "token amount × current price" to be honest about this, but a history unlocks the real story.
**Capture.** Per epoch, per pool (`gauge_pool_id`), per bribe token: `token`, `amount` (token units), `usd_at_capture`, `price_used`. Then the page can show bribe direction in tokens vs in USD, and attribute changes to additions vs price.

### 2.4 [MED] Per-pool emissions / APR estimate for divergent pools
**Why.** For several pools the cron's `approx_apr_pct` diverges from Eris's shown vAPR (e.g. USDC-EURe we showed ~16% vs Eris ~5%, also USDC-USDT, LUNA-ampLUNA). The page caps at 200% and zeroes <$20K staked, but the underlying estimate is off for these.
**Capture/fix.** Break APR into its components per pool (`fee_apr`, `emissions_apr`, `bribe_apr`) and validate the sum against Eris's vAPR where available. Single-asset pools (ampCAPA shows $0 depth) need a depth/value source so their APR isn't computed off a zero.

### 2.5 [LOW] Single-asset pool depth
**Why.** ampCAPA (single-asset) reports `depth_usd = 0` despite ~$206K staked, so any depth-based calc falls back. Capture a real depth/TVL figure for single-asset pools so they're not special-cased everywhere.

### 2.6 [MED] Propagate `dex_subtype` into `pool-status-history` (and everywhere per-pool)
**Why.** `tla-snapshot.json` carries `dex_subtype` (concentrated / xyk / stable / single), but `pool-status-history` does NOT — only `name, dex, bucket, pool_address` at pool level. This matters a lot for correctness of interpretation: 22 of the pools are **concentrated-liquidity (Astroport PCL)**, where the two sides hold deliberately *unequal* USD value (e.g. LUNA-arbLUNA at 62/38, LUNA-USDC, LUNA-CAPA, LUNA-FUEL). Without the subtype on the history record, the page had to reconstruct it via a fragile `name+bucket` lookup against the live snapshot. A standard 50/50 (xyk) pool at 62/38 *would* indicate a problem; a concentrated pool at 62/38 is perfectly healthy — you can't tell which without the subtype.
**Capture.** Add `dex_subtype` to every per-pool record in `pool-status-history` and the daily boundary files (keyed by `gauge_pool_id`). This also lets the page apply the correct IL model (the simple xy=k formula is wrong for concentrated pools — IL is amplified within the range) and the correct value-split expectation per pool type.
**Invariant tie-in:** the "standard pool must be ~50/50 by value" check in PART 3.3 should apply ONLY to `xyk`/`stable` subtypes — concentrated pools are expected to be uneven. Capturing the subtype is what makes that invariant check valid instead of a false alarm.

---

### 2.7 [MED] Pool migrations leave dead same-name pools — type is part of identity
**Why.** Pairs get migrated to more efficient curve types (e.g. Astroport xyk -> pcl/concentrated, or stable -> pcl). The OLD pool stays on-chain as a near-zero, inactive husk with the SAME name+dex but a different dex_subtype and a different gauge_pool_id. Confirmed in the Eris liquidity dump: LUNA-arbLUNA exists as Astroport-pcl (active, real) AND Astroport-stable (inactive corpse); LUNA-USDC as Astroport-pcl (active) AND Astroport-xyk (dead); USDC-USDT with multiple dead SkeletonSwap-xyk entries. This is a major source of the name-collision pain.
**Implication.** Reinforces 1.1 (gauge_pool_id is the only safe key) AND 2.6 (carry dex_subtype). The cron should treat (gauge_pool_id) as identity and may additionally tag pools that are the *migrated-away* version so the page can hide corpses explicitly rather than relying on a VP-size heuristic. Ideally capture a per-pool flag like is_migrated_legacy or current_canonical_gauge_for_pair so the active pool for each pair is unambiguous, not inferred from whichever has the most VP.

---

### 2.8 [MED] Resolve pool pair-names for all gauges (some show as raw gauge IDs)
**Why.** A handful of pools arrive with `name` = the raw gauge id (e.g. `cw20:terra1hqq6pnx...`, `native:factory/terra1zushw...`) and `dex`/`dex_subtype` = null, because the snapshot never resolved them to a human pair name. Most are tiny below-threshold husks, but at least one is a genuinely ACTIVE pool (`cw20:terra1hqq6pnx...`, ~430K VP, ~$7.5K staked, single bucket) that shows in the table as a gauge id. Two consequences on the page: it displays an ugly raw address instead of a pair name, AND it can't be matched to the astroport epoch-avg liquidity/volume data (which is keyed by pair name), so its Epoch Avg Liq/Vol show blank.
**Fix.** Resolve every gauge's pair name + dex + dex_subtype at capture time (look up the pool contract → its two assets → format "TOKENA-TOKENB"). Key by gauge_pool_id (per 1.1). Then these pools get a proper name and can join the epoch-avg data.
**Page side (done):** the table now shows an explanatory marker instead of a bare blank — "data pending" for Skeleton Swap, "n/a · single-asset" for ampCAPA/xASTRO, and a dash with a tooltip for unresolved-name pools.

### 2.9 [note] Epoch-avg liq/vol blanks — three legitimate causes (mostly not bugs)
For the Pools-tab "Epoch Avg Liq / Vol" columns, blanks come from: (a) **Skeleton Swap** pools — the SS upstream API is stale (see 1.4); resolved when the SS cron moves to /api/pools; (b) **single-asset** pools (ampCAPA, xASTRO) — no LP-pair liquidity/volume exists by nature, so blank is correct and permanent; (c) **unresolved-name** pools — see 2.8. Only (a) and (c) are fixable upstream; (b) is expected. The page now labels each case so a blank never reads as a malfunction.

---

### 2.10 [HIGH] Compute real net APR from the staking contracts (first-principles, no Eris-guessing)
**Decision.** We can't reverse-engineer Eris's *displayed* APR (it has per-pool factors that don't reconcile from displayed data — e.g. LUNA-ampLUNA shows 19% where any depth/staked formula gives 3.5%). Instead, compute the TRUE net APR ourselves from the bucket staking contracts. This is more trustworthy than matching a screen-scraped number.

**What the chain confirmed (validated this session):**
- Our reward DISTRIBUTION is exact. The bucket staking contract's `reward_distribution` query gives per-pool shares; LUNA-USDC = 0.819836966433597106 of the stable bucket, and our allocation gives it 82% of $427,872 = $350,785. Matches to the decimal.
- Our STAKED value is exact. On-chain staked LP for LUNA-USDC = 1,131,043,285,926 (6dec) → ~$597,490 at the LP price; our snapshot had $597,702.
- There is a **10% take rate**. Every asset in `total_staked_balances` / `whitelisted_asset_details` carries `yearly_take_rate: 0.1`. Eris's displayed APR is net of this; our current `approx_apr_pct` is GROSS (that's most of why ours reads ~15-20% higher).

**The bucket staking contracts** (ve3-asset-staking, version 1.8.2), each queryable:
- stable:  `terra1v399cx9drllm70wxfsgvfe694tdsd9x96p9ha36w7muffe4znlusqswspq`
- project: `terra1awq6t7jfakg9wfjn40fk3wzwmd57mvrqtt3a39z9rmet7wdjj3ysgw3lpa`
- bluechip:`terra14mmvqn0kthw6sre75vku263lafn5655mkjdejqjedjga4cw0qx2qlf4arv`
- single:  `terra1qdz5qgafx88kp5mf6m2tah8742g4u5g2cek0m3jrgssexexk7g4qw6e23k`

**Queries that matter** (per contract):
- `total_staked_balances` → per-asset: `amount` (staked LP, 6dec), `shares`/`total_shares`, `taken`, `harvested`, `last_taken_s`, `yearly_take_rate`, `stake_config` (astroport incentives contract + `reward_infos` = reward token).
- `reward_distribution` → per-asset: `distribution` (share of bucket pot), `total_vp`.
- `whitelisted_asset_details` → per-asset config + `whitelisted` flag (use to drop non-whitelisted husks).

**Two ways to compute net APR — recommend B, cross-check with A:**
- **A (rate-based):** from the astroport-incentives contract in `stake_config`, read the reward rate (tokens/sec) for each staked LP → annualize × reward-token price → / staked_usd → × (1 − take_rate). Forward-looking, matches Eris's "assuming nothing changes" framing.
- **B (realized, model-free):** the contract's `harvested`/`taken` grow as rewards accrue. Across two hourly reads, Δharvested / Δt / staked_usd, annualized, is the REALIZED net rate — immune to formula guesswork. Most trustworthy; needs two snapshots to start.

**Store** per pool keyed by gauge_pool_id: `rewards.net_apr_pct` (and keep `approx_apr_pct` as gross for reference). Page switches the APR column to `net_apr_pct`, drops the suppression floor (net APR is well-behaved), and labels it plainly.

**Validation targets** (Eris displayed apr_white, epoch ~187): LUNA-FUEL 61.3, LUNA-USDT 56.15, LUNA-CAPA 54.16, LUNA-USDC 48.23, LUNA-ASTRO 47.79, LUNA-EURe 41.01, LUNA-ATOM 39.92, LUNA-INJ 38.05, LUNA-SOLID 36.96, ampCAPA 26.63, LUNA-PAXG 23.37, LUNA-ROAR 21.69, LUNA-arbLUNA 19.63, LUNA-ampLUNA 19.03, USDC-EURe 5.80, USDC-USDT 0.59. Our gross for LUNA-USDC is 58.7%; ×0.9 take = 52.8%; Eris shows 48.23% — the residual is the per-pool piece approach A/B resolves. Match within ~1-2% and we're done.

**Page side now:** APR stays hidden under $20K (Rev 3.46) until net_apr_pct lands. Don't ship gross APR as if it were the real yield.

### PRINCIPLE: capture from source, store history, reference later
Guiding philosophy for the contract-reading rebuild (Camron): the Eris ve3 contracts hold the complete truth — pools, VP, rewards, bribes, locks. The goal is to (1) read from these authoritative sources rather than reconstruct, (2) SNAPSHOT the raw query outputs each epoch into history repos so we build trend data and always have a record, and (3) have a documented reference (this brief) so when a problem comes up, we know which contract/query holds the answer. Capturing comprehensively now — even data we don't display yet — means we can build features later without re-deriving, and gives us the means to diagnose any discrepancy by going back to source. Store the raw, derive the views; never throw away the raw.

### 2.11 [HIGH] Read the Eris ve3 contracts directly (authoritative source for pools, VP, bribes, rewards)
**Why.** Our crons currently derive pool lists, VP, bribes, and reward data through indirect paths — none of the Eris infrastructure contract addresses appear in our stored output, meaning we reconstruct rather than read the source of truth. Reading these contracts directly would collapse several existing brief items into "just query the contract." Confirmed live this session against the staking + gauge contracts.

**The Eris ve3 contract set** (from the liquidity-hub footer + verified on chainscope):
- `ve3-asset-gauge`     `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj` (v1.7.0) — gauges [stable, project, bluechip, single], min_gauge_pct 0.01, rebase_asset ampLUNA. Likely the master pool/gauge registry + votes. **Could fix 2.8 (name resolution), 1.1 (canonical gauge_pool_id), 2.1 (per-pool VP).**
- `global-config`       `terra1hwxg6s732eparz3ys7sa4t5f64ngpd2w8syrca6z7ckv3fs9uqnsvrpcqa` — likely bucket weights (the 0.4 stable), take rate (0.1), epoch timing, contract pointers. **Could replace hardcoded constants.**
- `voting-escrow`       `terra1uqhj8agyeaz8fu6mdggfuwr3lp32jlrx5hqag4jxexde92rzkamq3l62zg` — VP + locks. **Could be the authoritative Votion VP / lock_contributions source.**
- `bribe-manager`       `terra1tuuwm8yrj54qeg0c8xu00aha9ryatyhtczq8qq2q8tntuw0auzas9037wh` — bribes per gauge/epoch. **Could replace the PD-file + on-chain-active merge.**
- `asset-compounder`    `terra1zly98gvcec54m3caxlqexce7rus6rzgplz7eketsdz7nh750h2rqvu8uzx`
- bucket staking (ve3-asset-staking v1.8.2): stable `…qswspq`, project `…gw3lpa`, bluechip `…lf4arv`, single `…exexk7g4qw6e23k` — staked balances, reward_distribution, take rate (see 2.10).
- astroport incentives `terra1eywh4av8sln6r45pxq45ltj798htfy0cfcf7fy3pxc2gcv6uc07se4ch9x` — per-LP reward RATE lives here (the APR numerator, see 2.10).

**Reward flow (confirmed from on-chain "Distribute Take Rate" txs):** astroport-incentives streams reward tokens (ASTRO + LP tokens) into each bucket-staking contract → staking takes its 10% rate → forwards bribes to bribe-manager → rebase distributed via the connector. So per-pool reward rate = the incentives stream to that gauge; this is the clean APR numerator.

**Next step (data needed):** dump each contract's query list + key query outputs (priority: asset-gauge pool/vote query, then global-config). Then map each query to cron-capture vs live-tile and rewrite the affected brief items to read the contract directly instead of reconstructing.

### 2.12 [HIGH] ve3-asset-gauge `distributions` = authoritative pool registry (collapses several items)
**Confirmed via live query.** The gauge contract `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj` exposes queries that are the SOURCE OF TRUTH for data we currently reconstruct:

- **`distributions` / `last_distributions`** → per gauge (bucket): `total_gauge_vp`; per pool: the gauge-asset address, `distribution` (reward share %), `total_vp`. Period-stamped. This is the canonical pool registry + VP + reward split. Verified it matches our snapshot (stable LUNA-USDC: dist 0.819836966, the exact figure we reconstruct).
- **`last_distribution_period`** → `{period: 186}`. The CANONICAL epoch number — ground truth for the SS/astro epoch-label off-by-one (item 1.4). Confirms lastCompleteEpoch=186 is correct.
- **`config`** → gauge list + `global_config_addr` + `rebase_asset` (cw20 terra1ecgazyd…, = ampLUNA).
- **`user_infos`** → ALL holders' VP `{voting_power, fixed_amount, slope}`. Source for VP-holder/concentration (item 2.2).
- **`user_info{user}` / `user_shares{user}` / `user_first_participation{user}` / `user_pending_rebase{user}`** → per-wallet, address-arg queries (they error with empty `{}` — need `{"user_info":{"user":"terra1…"}}`).

**This single contract collapses:**
- **1.1** (canonical gauge_pool_id) — `distributions` keys every pool by its real asset address.
- **2.1** (per-pool VP at epoch boundary) — `total_vp` per pool, period-stamped, from source.
- **2.8** (bucket membership / name resolution) — the gauge says which bucket each asset is in; the "unresolved" pool terra1hqq6pnx… is confirmed a SINGLE-bucket asset (dist 1.97%, vp 470,056), no more guessing.
- **2.10** (APR numerator) — per-pool reward = `distribution` % × that bucket's reward pot.
- **1.4** (epoch label) — `last_distribution_period` is the canonical number.
- **2.2** (VP-holder concentration) — `user_infos` is the full holder list.

**Recommendation:** add a cron that reads `last_distributions` + `config` + `user_infos` from this contract each epoch, keyed by gauge-asset address, and treat it as the master registry the other crons reconcile against (rather than each reconstructing independently). This is the highest-leverage single change for pipeline trustworthiness.

### 2.13 [HIGH] ve3-bribe-manager — authoritative bribe registry (replaces PD-file merge)
**Confirmed via live query.** `terra1tuuwm8yrj54qeg0c8xu00aha9ryatyhtczq8qq2q8tntuw0auzas9037wh` (ve3-bribe-manager v1.8.1).
Queries: `config`, `next_claim_period`, `bribes`, `user_claimable{user}`.
- **`bribes`** → authoritative ACTIVE/pending bribe registry. Per entry: `gauge` (bucket), `asset` (the POOL the bribe is on, keyed by cw20/native asset addr), `assets[]` (bribe token info + amount). This is exactly what we currently reconstruct by merging the PD-file (frozen usd_per_epoch) with on-chain-active bribes. The contract gives it directly and live. Example (current): stable LUNA-USDC ← 20,019 ASTRO; project ← 100,000 CAPA + an ibc/4B44… token (36,000); single ampCAPA ← 100,000 CAPA.
- **`config`** → `allow_any: true`, fee 10 LUNA (`uluna` 10000000), `global_config_addr`. The 10-LUNA add-bribe fee is visible in the "Add Bribe" txs.
- **`user_claimable{user}`** → per-wallet claimable bribes (address-arg) → feeds the wallet-lookup feature (3.x).
- **Reward flow confirmed**: "Distribute Bribes" txs send ASTRO from each bucket-staking contract INTO bribe-manager (terra1…9037wh); "Claim Bribes" txs send tokens out to voters. So bribe-manager holds pending bribes between distribution and claim.

**Recommendation:** bribes cron reads `bribes` directly each epoch keyed by pool asset addr, values bribe tokens at live prices — replaces the PD-file dependency for active bribes (keep PD-file only for historical/frozen-usd reference if still wanted). Resolves the bribe side of the pipeline to a single source.

**Token-registry gap:** the project-bucket bribe token `ibc/4B44179AC2F0BEE50C16A673B3B886398988692885B2848A1C8AEF27148B3961` (36,000 units) is not in our standard registry — add it so it resolves rather than showing as unknown.

### 2.14 [HIGH] global-config `all_addresses` = master contract directory (bootstrap query)
**Confirmed via live query.** `terra1hwxg6s732eparz3ys7sa4t5f64ngpd2w8syrca6z7ckv3fs9uqnsvrpcqa` (ve3-global-config v1.0.0).
Queries: `ownership`, `address{...}`, `addresses{...}`, `all_addresses`, `address_list{...}` (singular/list forms take an arg; `all_addresses` is the no-arg dump).
- **`all_addresses`** -> the canonical role->address map for the ENTIRE Eris ve3 system. A cron reads this ONE query and gets every contract, then queries each -- no hardcoded addresses anywhere, and it auto-tracks any future migration.
- **`ownership`** -> owner terra1kefa... (deployer), no pending transfer.

**Full directory (current):**
- ASSET_GAUGE terra1hfksr...msd3lj (mapped 2.12)
- ASSET_STAKING__stable terra1v399...qswspq (2.10/2.12) / __project terra1awq6...gw3lpa / __bluechip terra14mmv...lf4arv / __single terra1qdz5...w6e23k
- CONNECTOR__stable terra1ym24...r4q6n8 / __project terra1x8v9...cnhyh / __bluechip terra16l43...nl8h53 / __single terra1u72y...vmf8u (these are the "Distribute Rebase" targets in the txs)
- BRIBE_MANAGER terra1tuuw...9037wh (mapped 2.13)
- VOTING_ESCROW terra1uqhj...3l62zg (still to map -- VP/lock source)
- FEE_COLLECTOR terra1rgggsspquaxjp4lmegx7a3q4l9lg44hnu7rjxa (the ...u7rjxa fee dest in every tx)
- TAKE_RECIPIENT / ASSET_WHITELIST_CONTROLLER / BRIBE_WHITELIST_CONTROLLER / DELEGATION_CONTROLLER / VE_GUARDIAN / PDT_CONTROLLER = all terra1k8ug...4lppjg (this is the take-rate recipient seen in every "Distribute Take Rate" tx)
- ZAPPER terra1qdjs...dgmaxl, PDT_CONFIG_OWNER terra1kefa...klgzl

**Architectural recommendation (ties 2.10-2.14 together):** the contract-reading cron should bootstrap from `global-config.all_addresses` each run, then fan out: gauge `last_distributions` (registry + VP + reward split + canonical epoch), each `ASSET_STAKING__*` (staked + take rate + realized rewards), `bribe-manager.bribes` (active bribes), `voting-escrow` (VP source). Store all keyed by gauge-asset address. This one cron becomes the master source the dashboard reads, and every other derived cron reconciles against it. Highest-leverage change for trustworthiness -- collapses the hardcoded-address fragility entirely.

### 2.15 [HIGH] ve3-voting-escrow — authoritative VP source (veLUNA NFT lock collection)
**Confirmed via live query.** `terra1uqhj8agyeaz8fu6mdggfuwr3lp32jlrx5hqag4jxexde92rzkamq3l62zg` (ve3-voting-escrow v1.9.3). It is a CW721 NFT collection named "Vote Escrowed LUNA" / veLUNA — each lock is an NFT (token_id).
Queries: `total_vamp`, `total_fixed`, `lock_vamp{token_id}`, `lock_info{token_id}`, `config`, `nft_info{token_id}`, `all_nft_info{token_id}`, `owner_of{token_id}`, `tokens{owner}`, `all_tokens`, `num_tokens`, `blacklisted_voters`, `minter`, plus approvals/operators.
- **`total_vamp`** -> authoritative total VP: `voting_power` 23,740,592 (decaying) + `fixed` 2,848,552 (permanent) = `vp` 26,589,144 total. This is the system-wide VP from source.
- **`total_fixed`** -> the permanent-lock portion alone.
- **`num_tokens`** -> 431 total veLUNA locks. **`all_tokens`** -> paginated list of token_ids. **`tokens{owner}`** -> a wallet's lock NFTs.
- **`lock_info{token_id}`** -> THE per-lock source (full typed detail: asset, underlying_amount, start, end.period, slope, fixed_amount, voting_power, owner). Use this for everything.
- **`tokens{owner}`** -> CONFIRMED: array of a wallet's lock NFT ids (example wallet holds 8: 1157,118,151,18,227,23,540,585). One wallet = N locks; enumerate then lock_info each to roll up a wallet's full lock position.
- **`lock_vamp{token_id}`** -> CONFIRMED REDUNDANT: just `{fixed, voting_power, vp}` (vp=fixed+voting_power). lock_info already has these. Skip.
- **`nft_info{token_id}`** -> CONFIRMED: CW721 metadata, cosmetic fields null; lock data packed as attributes[] traits (asset "cw20:addr:amount", start, end) — a subset of lock_info. Skip for data; lock_info wins.
- **`config`** -> `deposit_assets` = the lockable assets: uluna, ampLUNA (terra1ecgaz…, w/ exchange_rate oracle), arbLUNA (terra17aj4ty…), stLUNA (ibc/0809…), + one more cw20 — each with its exchange_rate contract for VP conversion. `push_update_contracts` = [asset-gauge]. This is the authoritative LST list + the exchange-rate oracle addresses (relevant to the Votion arbLUNA/ampLUNA × ratio × 10x reconciliation in memory).
- **`blacklisted_voters`** -> currently [] (empty). Worth capturing so blacklisted VP can be excluded if it ever populates.

**Reconciliation note (verify carefully when building):** total veLUNA VP = 26.59M (from total_vamp). Per-bucket `total_gauge_vp` from the gauge: stable 27.22M, project 26.15M, bluechip 21.94M, single 23.83M. Each bucket's total_gauge_vp is the VP *allocated to that gauge* (a voter can put their full VP in each bucket independently), so per-bucket can exceed/differ from the 26.59M global — confirms "max bucket VP" as canonical TLA VP is a per-bucket view, NOT the same as voting-escrow total. The voting-escrow total_vamp (26.59M) is the true count-once total locked VP; the gauge per-bucket totals are allocation views. Keep these two concepts distinct on the dashboard.

**Map now complete:** global-config (directory) -> asset-gauge (registry/VP/rewards/epoch) + ASSET_STAKING__* (staked/take/rewards) + bribe-manager (bribes) + voting-escrow (VP/locks). All five core contracts mapped and verified against our reconstructed data — every check matched.

### 2.16 [HIGH] Self-discovering pool registry — LPs auto-add/remove (no manual list)
**The goal (Camron):** stop maintaining a pool list by hand. The dashboard should discover which LPs exist, in which bucket, automatically — so when Eris whitelists a new pool it appears next epoch, and when one is delisted it drops off, with zero manual edits.

**Why this is now possible (it wasn't before):** the gauge + global-config + staking contracts ARE the registry. The full pool set, bucket membership, VP, reward share, and whitelist status all come from queries we've confirmed:
- `asset-gauge.last_distributions` → every pool currently receiving distribution, grouped by gauge (bucket), keyed by asset address. This IS the live pool list.
- `ASSET_STAKING__*.whitelisted_asset_details` → per-asset `whitelisted` flag (true/false) → authoritative add/remove signal. (We saw non-whitelisted husks flagged false — that's how to drop dead variants automatically, fixing the variant-collision problem at the source.)
- `global-config.all_addresses` → the contracts to read, also self-updating.

**Design:**
1. Each epoch, cron bootstraps `global-config.all_addresses`, then reads `last_distributions` (pool set + bucket + VP + reward %) and each staking contract's `whitelisted_asset_details` (whitelist flag + take rate + staked).
2. Build the pool registry from THAT — don't keep a hardcoded list. A pool exists iff it's in the gauge distribution and whitelisted=true.
3. Map asset address → display name via a small lookup (LP token → underlying pair), the ONE thing still needing a name source (Astroport/SS pool metadata). Everything else (which bucket, is it active, its VP/rewards) comes from chain.
4. New pool appears → shows up automatically. Pool delisted (whitelisted=false or absent from distribution) → drops automatically. Variant husks (whitelisted=false) → excluded automatically.

**This collapses the manual-maintenance burden AND fixes at the source:** 1.1 (canonical IDs), 2.6/2.7 (dex_subtype/migrated husks), 2.8 (name resolution — only the name lookup remains manual), the variant-collision band-aid (Rev 3.43), and the "unresolved pool" markers. The pool list becomes a derived view of chain state, not a maintained artifact.

**Only remaining manual piece:** asset-address → human pool name (e.g. cw20 terra1s275… → "LUNA-USDC stable"). Source this from Astroport's pool API / factory metadata keyed by LP address, or a tiny hand-maintained name map (~30 entries, changes rarely). Everything structural is automatic.

### 2.17 [HIGH] veLUNA lock registry — enumerate all 431 locks (unlock schedule + composition)
**Camron's insight:** `total_vamp` only gives the aggregate. To know what's INSIDE each lock — underlying LUNA, LST type (amp/arb/b/st), lock duration, time-to-unlock, VP boost/decay — you must enumerate each veLUNA NFT individually. This unlocks the single most valuable feature on the roadmap: a LUNA UNLOCK SCHEDULE (when large amounts become withdrawable and could be sold), which nobody else surfaces.

**Enumeration (confirmed feasible):**
- `all_tokens{limit,start_after}` → paginate the full token_id list (431 total, ~10-100/page → ~5-15 calls).
- `lock_info{token_id}` → per-lock detail. 431 calls/epoch.
- `owner_of{token_id}` → owner (or get owner via `tokens{owner}` reverse map).
- Cost ~440 queries/epoch; at BATCH_CONCURRENCY<=5 that's ~90 batches — trivial for an hourly cron. (Could run less often, e.g. every few hours, since locks change slowly.)

**`lock_info` schema CONFIRMED** from a real query (see "SCHEMA CONFIRMED" below): {owner, from_period, asset:{info,amount}, underlying_amount, coefficient, start, end:{period}, slope, fixed_amount, voting_power}. No further sampling needed.

**Store:** a `locks` dataset (one record per token_id) in a new *-data_2026 repo, snapshotted each epoch → gives history of the lock book over time.

**Features it powers:**
- **Unlock schedule / cliff calendar:** group locks by unlock time → "X LUNA (Y ampLUNA + Z arbLUNA) unlockable in next 7/30/90 days." Market-relevant, unique.
- **VP decay forecast:** slope per lock → project how total VP shrinks over coming epochs (affects everyone's reward share).
- **Lock composition:** permanent vs decaying, by LST type → commitment health of the system.
- **Whale watch:** largest locks, their unlock dates, owners (ties to wallet-lookup).
- Feeds the member page (each member's locks) and the early-user features.

**SCHEMA CONFIRMED** (from a real `lock_info{token_id}` query):
```
{ owner, from_period (=queried period, e.g. 186),
  asset: { info: {cw20|native}, amount },   // the LST token + raw amount
  underlying_amount,                          // LUNA-equivalent that UNLOCKS (use for sell-pressure calendar)
  coefficient,                                // lock-duration multiplier (boosts VP; e.g. 9)
  start (period lock began), end: {period},   // end.period = UNLOCK epoch
  slope,                                       // VP decay per period
  fixed_amount,                                // permanent VP floor (~= underlying)
  voting_power }                               // current boosted/decaying VP (use for VP-decay forecast)
```
Example decoded: owner terra1g6yk…, ampLUNA lock, 1.115 ampLUNA = 1.785 LUNA-equiv (rate ~1.60), coefficient 9, started period 96, unlocks period 229 (43 epochs ≈ 10 months out), current VP 6.64 (boosted ~3.7× over underlying), decaying via slope toward end.

**Two amounts — don't conflate:** `underlying_amount` = real LUNA-equiv released at unlock (UNLOCK SCHEDULE); `voting_power` = governance weight now (VP DECAY FORECAST via slope + end period). `end.period` = the unlock epoch (convert to date via epoch_1-300_date.json). LST type from `asset.info`. `underlying_amount` already appears pre-converted to LUNA-equiv — confirm across amp/arb/b/st locks whether we still need the exchange-rate oracles, or if underlying_amount is sufficient (likely sufficient). Schema is locked; the lock registry can now be built.

### 2.18 [HIGH] Per-wallet gauge queries — complete member profile from source (schemas confirmed)
**Confirmed via live queries** on ve3-asset-gauge. Together these build a full member/wallet profile — exactly what the queued dao-tla.html member page needs, sourced not reconstructed.

- **`user_info{user}`** → `{voting_power, fixed_amount, slope, gauge_votes[]}`. gauge_votes = the wallet's vote allocation per bucket: `{gauge, period, votes: [[asset, weight_bps]]}` (10000 bps = 100%). The `period` per gauge = the epoch the wallet LAST CHANGED its vote for that bucket.
- **`user_shares{user}`** → per gauge: `{gauge, asset, period (current, 186), user_vp, total_vp}`. user_vp / total_vp = the wallet's % influence on that pool RIGHT NOW. THIS is the correct source for "member's % of a pool" (the aDAO-vote column + member page). Use this (live period) over gauge_votes (last-changed).
- **`user_first_participation{user}`** → `{period}`. Bare epoch number of first participation → EARLY-USER / tenure feature. (Example: period 96.)
- **`user_pending_rebase{user}`** → `{rebase}`. Unclaimed rebase amount (ampLUNA).
- **`gauge_info`** → needs `{key: ...}` not `{gauge:}` (errored "missing field key"); low priority since `distributions` covers gauge data. Skip unless a key shape turns up.

**Behavioral insight — VOTES PERSIST:** a vote stays in effect until changed (example wallet: stable/project/single last voted epoch 173, bluechip epoch 159, all still counting at 186). So members don't re-vote weekly; allocation carries forward. IMPORTANT framing (per Camron): an unchanged vote is NOT disengagement — a rational voter only re-votes a bucket when they see a reason to move it; a bucket left alone means they're still happy with that allocation. So "last changed period" should be presented neutrally (e.g. "vote set since epoch 159"), not as a staleness/inactivity flag. Implications: (1) for live influence use user_shares (effective at current period), not gauge_votes (shows last change); (2) "vote set since" is informational context, not a judgment; (3) explains how the 46 aDAO members hold steady VP allocation without weekly action.

**Complete member profile (all from source):** user_info (VP + vote targets) + user_shares (live % per pool) + user_first_participation (tenure) + user_pending_rebase (unclaimed) + escrow tokens{owner}→lock_info each (their locks + unlock dates). This fully specs dao-tla.html and the wallet-lookup tool.

### 2.19 [HIGH] Master participant registry — enumerate all addresses, diff for history
**Camron's goal:** a master list of every address that holds locks, deposits/stakes, votes, or provides bribes — captured as they're added/removed — so we can run per-wallet queries on them and build history/trends over time.

**Key realization:** we don't discover addresses by scanning txs — each contract ENUMERATES its own participants. Union them = the master list. Sources:
- **Voters / VP holders:** `gauge.user_infos` → every address with voting power + their VP (paginated; we pulled one page already). This is the bulk of participants.
- **Lock holders:** `escrow.all_tokens` → 431 token_ids → `lock_info{token_id}.owner` (or `owner_of`) → union of owners = everyone with a veLUNA lock.
- **LP stakers:** `staking.pool_stakers{asset}` per pool across all 4 buckets → every address staked in each TLA pool. (Arg shape confirming in Batch 4.)
- **Bribers:** likely from bribe-manager "Add Bribe" tx senders (tx-history scan) — check if a bribe-manager query lists bribers; if not, tx scan is the fallback. (Lower volume, easy.)
- **Claimers:** derivable — anyone with `user_claimable` > 0 or "Claim Bribes" tx senders.

**Design:**
1. Each epoch, enumerate all four sources → union into a `participants` master set (with a per-address role flag: voter / locker / staker / briber).
2. Snapshot the master set each epoch into a history repo.
3. Diff epoch-over-epoch → who JOINED, who LEFT → the add/remove history Camron wants.
4. For each participant, run the per-wallet queries (2.18 + lock_info + pool_stakers) → full profile, snapshotted → per-wallet trends over time.

**This is the backbone** that makes every per-wallet feature (member page, wallet lookup, early-user recognition, concentration analysis, whale watch) work at scale and over time — not just on-demand for one address. Pairs with the capture-from-source PRINCIPLE: store the raw per-wallet outputs each epoch, derive views later.

**Scale:** user_infos is the big one (every VP holder — we saw it returns the full set, paginated). Per-wallet detail queries × participants × epoch is the cost; batch with concurrency ≤5 (publicnode limit). Run on the epoch cadence, not hourly, since participation changes slowly.

### REF-B: Remaining Eris contracts — assessed, intentionally NOT mapped (low data value)
Decision: the 5 mapped contracts (global-config, asset-gauge, asset-staking×4, bribe-manager, voting-escrow) carry all dashboard data. The rest are plumbing/permissions, not data sources — intentionally skipped. Recorded here so it's a deliberate choice, not an oversight.
- CONNECTOR__{stable,project,bluechip,single} — rebase distribution plumbing. Map only if showing rebase APR.
- ASSET_COMPOUNDER (terra1zly98…, ve3-asset-compounding v1.9.7) — auto-compound mechanics; power-user detail. PARTIALLY MAPPED (curiosity peek): `config{}` → {global_config_addr (points back to directory — self-consistent), **fee: 0.08 (8% COMPOUNDER fee, SEPARATE from staking's 10% take)**, fee_collector terra1rgggss…u7rjxa (confirms directory), deposit_profit_delay_s: 14400 (4h anti-gaming delay), denom_creation_fee: 10 LUNA}. TWO useful findings: (1) compounded positions pay BOTH the 10% staking take AND this 8% compounder fee → a TRUE net APR for an auto-compounded position is ~gross × 0.9 × 0.92, but ONLY for compounded positions (don't apply blanket). (2) **THIRD field-name variant: this contract uses `addr`** (user_infos errors "missing field addr") — so gauge/bribe=`user`, staking=`address`, compounding=`addr`. The three LIST queries take no args and ARE confirmed/decoded:
  • `asset_configs{}` → COMPOUNDING VAULT REGISTRY: per pool {asset_info, gauge, staking (which bucket contract), amp_denom ("factory/terra1zly98…/{N}/{bucket}/amplp", N=stable vault index 0..64), zasset_denom, reward_asset_info (ampLUNA for all — rewards compound into ampLUNA), fee (null=use global 8%), total_bond_share (reads 0 in the config list; live bond state needs a different query)}.
  • `amplp_exchange_rates{}` → current amplp→LP rate per vault {gauge, asset, amplp_denom, exchange_rate}. e.g. LUNA-USDC stable = 3.4645 LP per amplp.
  • `exchange_rates{}` → rate HISTORY + `apr` per vault. **CONFIRMED: `apr` = the PER-PERIOD (daily) compounding rate** (verified: LUNA-USDC stable measured 0.1592%/period vs apr field 0.1628% — match). Annualize ×365 simple (~59%) or compound (~81%). **This is the REALIZED, NET-OF-8%-FEE daily yield baked into the amplp token — arguably the truest "what a compounded staker actually earned" per pool, straight from source.** Strong candidate to surface as a "compounding APR" column and to cross-check item 2.10's reconstructed net APR.
  Per-arg shapes (confirmed via errors): `asset_config{asset_info:{cw20:...}, gauge:"stable"}` — needs BOTH asset_info AND gauge (asset alone is ambiguous across buckets). `user_infos{addr, assets}` — INPUT FIELDS KNOWN, exact `assets` element ENCODING DEFERRED to build time. The contract error names the top-level fields: `addr` (wallet — THIRD field-name variant: gauge=`user`, staking=`address`, compounder=`addr`) and `assets` (a LIST = a vault FILTER, not pagination). Bare `{addr}` OOGs (gasUsed 3.05M > 3M) — with no `assets` filter it scans ALL vaults; the fix is to pass the specific vaults the wallet holds (known from its amplp balances) -> cheap, targeted, no OOG. NOT yet pinned: the exact JSON encoding of each `assets[]` ELEMENT. Tried and rejected ("Invalid type") via chainscope: bare `{cw20:addr}`, `{asset_info:{cw20:addr},gauge:"stable"}`, and `[{cw20:addr},"stable"]` (tuple). RESOLVE AT BUILD TIME: generate the message type from the `erisprotocol/contracts-ve3` source / the contract's shipped JSON schema (CosmWasm contracts expose a query schema) — the (AssetInfo, gauge) element struct is exact there, a 30-second job with the type in front of you, vs blind guessing now. This is the ONLY unconfirmed query in the system AND it is the lowest-priority feature (per-wallet compounder lookup — nothing else depends on it).

**Bonus from the asset_configs output — LP token-type map (authoritative):** asset_info is a bare string-valued enum, two forms: `{"cw20":"terra1..."}` (older Astroport LPs) and `{"native":"factory/terra1.../uLP"}` (newer tokenfactory LPs). The registry tells you per pool which form each LP uses — useful for the cron's LP balance/price lookups regardless of user_infos.
- ZAPPER (terra1qdjs…) — single-asset→LP tx helper; no displayable state. Skip.
- Controllers cluster (terra1k8ug…4lppjg = ASSET/BRIBE_WHITELIST_CONTROLLER, DELEGATION_CONTROLLER, VE_GUARDIAN, PDT_CONTROLLER, TAKE_RECIPIENT) — governance/permission endpoints.
- ONE nice-to-have if ever wanted: TAKE_RECIPIENT (terra1k8ug…4lppjg) + FEE_COLLECTOR (terra1rggg…7rjxa) BALANCES = a "protocol revenue / cumulative take" stat. Grab the two balances ad-hoc; no systematic mapping needed.
Since the cron bootstraps from global-config.all_addresses, any of these is reachable later if a need arises — no work lost by deferring.

### TRANSPARENCY DOCS ITEM — explain the fee stack (Camron-requested, neutral framing)
Add to the docs section / a visible spot so users understand fees on TLA pools. Confirmed against Eris's own docs (docs.erisprotocol.com/products/amp-compounder) — fee is DOCUMENTED, not hidden; their published APY formula already bakes in the reward fee. Frame positively (convenience + tax efficiency), not as a gotcha:
- **Non-compounding (plain staked LP):** subject to the 10% take rate only.
- **Auto-compounding (amplp vaults):** subject to the 10% take rate AND an 8% compounder reward fee = ~18% total on the reward stream (fees → fee collector → weekly to Treasury + ampLUNA holders).
- **Why people still choose it:** the 8% buys daily auto-compounding (APR→APY), no manual claim/restake gas, and fewer taxable events. Verified math: for higher-yield pools the daily compounding MORE than offsets the extra 8% (e.g. 50% reward APR → simple net 45% vs auto-compound realized ~51% APY; 20% APR ≈ break-even at ~18%). For very low-yield pools the 8% can edge out the small compounding gain — so the honest nuance is "convenience + tax efficiency always; net-yield edge mainly on the higher-APR pools."
- Dashboard angle: show GROSS reward APR, REALIZED net (from compounder `exchange_rates.apr`, already net of both fees), and the spread — the kind of transparency that makes tla-stats more trustworthy than the native UI.

### 2.10b [HIGH] APR breakdown feature — "gross − take − fee = realized", all from source (SPEC + verification gates)
Camron's vision: show the REAL APR with the fee stack visible, e.g. "Reward APR xxx − 10% take = xxx − 8% compound fee = xxx realized." Bar: every line traceable to a chain query (source of truth), NOT scraped sites or layered crons. A flagship trust feature and a reason to use tla-stats over the native UI.

**Data lineage (each line → its chain source):**
- Gross reward APR = annual reward emissions (USD) / staked-in-TLA (USD). Source: gauge.distributions (pool's reward share) + staking.total_staked_balances + LP price from Astroport pool reserves.
- − 10% take: × 0.90. Source: staking.whitelisted_asset_details.yearly_take_rate (=0.1), DIRECT.
- − 8% compound fee: × 0.92 (compounded positions only). Source: compounder.config.fee (=0.08), DIRECT.
- = Realized APR. Source: compounder.exchange_rates.apr — MEASURED on-chain (amplp appreciation), net of everything.

**Correctness proof = reconciliation:** computed (gross − take − fee) must ≈ realized (exchange_rates.apr) within tolerance. If they agree, the breakdown is provably right; if not, we're missing something and DON'T ship. This sidesteps Eris's display formula entirely — we compute our own and validate against the realized on-chain number, so we never need to crack the unexplained per-pool factor in Eris's displayed APR.

**Honest gap — does NOT cleanly close yet (must resolve before publishing):** for LUNA-USDC stable, our gross reward APR ~58.7% vs realized exchange_rates.apr ~59.4% (×365) — realized ≈ our GROSS, not our net. Likely cause: realized amplp also captures Astroport BASE yield (trading fees/incentives) which is NOT in our gross (TLA emissions only). So scopes differ: realized = (TLA rewards + Astroport base) − take − fee; our gross = TLA rewards only.

**Verification gates (close all before shipping the feature):**
1. Confirm whether gauge.distributions emissions are PRE- or POST-take (the "Distribute Take Rate" tx flow to terra1k8ug… suggests gross comes in then 10% splits off — so distributions may be gross/pre-take; verify).
2. Add Astroport base yield to the gross side and confirm whether the 10% take / 8% fee apply to base yield too (so the breakdown reconciles against realized).
3. Pin the USD price source to chain (Astroport pool reserves / oracle) so it's source-of-truth and consistent across all lines.
4. Then verify: computed net ≈ realized exchange_rates.apr per pool. Ship only on reconciliation; surface any non-reconciling pool as a data-health flag rather than a number.

**Confidence:** mechanism + sources are solid and chain-native; the reconciliation does not yet close, so the feature is SPEC'd not READY. Build it one pool at a time, reconcile against realized, expand only when it matches.

### 2.10c [HIGH] Price/oracle source — pin it, and make it robust (Camron's standing concern)
Camron's worry (valid): DEX prices can be manipulated/broken (thin-pool spot manipulation → bad valuations, IL, "LP rebalance wreck"). He wants source-of-truth pricing with multiples + backups, and to KNOW what Eris/Astroport actually use vs CoinGecko.

**What Astroport actually uses (confirmed from docs.astroport.fi):**
- PCL pools (most TLA pools — LUNA-USDC, ampLUNA-USDC, etc. are "pcl"): the pool has a BUILT-IN price oracle = an exponential moving average (EMA) of recent-trade prices, param `ema_half_time`. It is NOT a raw spot read. The PCL design explicitly addresses IL/volatility: as volatility widens the gap between Price Scale and Price Oracle, the pool raises fees to compensate LPs and can re-peg toward the oracle. So PCL spot is already smoothed on-chain.
- xy=k pools: a separate Astroport Oracle contract computes a 1-DAY TWAP. On-chain, queryable.
- PCL pools also expose historical balances / a backward-offset price observation query for TWAP-style reads.
- **Eris's price source — CONFIRMED: an OFF-CHAIN backend at https://backend.erisprotocol.com/prices that uses CoinGecko.** Base assets carry coingecko_id (USDC→usd-coin, LUNA→terra-luna-2, ASTRO→astroport-fi, ampLUNA→eris-amplified-luna, SOLID→solid-2, CAPA→capapult, ATOM→cosmos, etc.); LP/amplp tokens have NO coingecko_id and are computed from constituents. This is a CENTRALIZED endpoint with a CoinGecko dependency — NOT source-of-truth chain data. It also contains real internal inconsistencies (same display name, divergent prices): KUJI 18× spread, ASTRO 9× (a stale cw20 entry $0.0076 vs ~$0.001), stATOM 2× ($7.18 vs $3.63), MARS 146×, HAVA 973× — mostly thin/multi-bridge assets, but concrete proof a single endpoint carries stale/divergent prices. USE IT as ONE cross-check source + a convenient LP/amplp USD shortcut (it hands us LP prices directly, e.g. LUNA-USDC LP $0.53647), NEVER as sole truth. Useful identity confirmed: amplp_usd = LP_usd × amplp_exchange_rate (LUNA-USDC: 1.85856/0.53647 = 3.46445 ✓ matches the decoded rate). Astroport also runs its own backend; so we have ≥2 off-chain endpoints + on-chain oracles = good material for multi-source cross-checks.

**Pricing principle for tla-stats (source-of-truth + robust, matches Camron's instinct):**
1. Anchor on a stable: USDC = $1 (or a reference). Derive every token price by walking on-chain pool reserves/oracle from a USDC pair — never trust a third-party USD endpoint as the truth.
2. Prefer the on-chain TWAP/EMA (PCL internal oracle; xy=k 1-day TWAP) over raw spot — manipulation-resistant by construction.
3. Use the DEEPEST pool for each token's price (thin pools are the manipulable ones).
4. Multi-source sanity bounds: cross-check (e.g. LUNA via LUNA-USDC vs LUNA via ampLUNA path); if sources diverge beyond a tolerance, FLAG the pool as a data-health warning instead of showing a number. This is the "multiples + backups" Camron wants and doubles as manipulation detection.
5. Store the price + its source each epoch (history + audit trail).

This pricing layer feeds 2.10b's gross-APR denominator (staked USD) and any USD comparison, so it must be settled FIRST — an unreliable price makes every downstream APR wrong. It's also a feature in itself: a "price source + freshness + cross-source agreement" panel is exactly the trust signal that differentiates tla-stats.

### IDEA (spec'd) — amplp wallet/holdings + transfer helper tool
Camron's long-wanted feature. amplp tokens are TRANSFERABLE tokenfactory denoms: an LP position tokenized so it can be sent wallet-to-wallet WITHOUT unstaking/withdrawing the underlying LP. Most dashboards never expose this. Tool spec:
- Input: any address (wallet-lookup style).
- Per amplp vault the address holds, show: raw amplp balance (reads as huge integer — base units), USD value, price of 1 amplp, and the denom to import into Keplr so the wallet displays the token.
- Send helper: input "$X" → compute amplp amount = X / amplp_price, then × 10^decimals for base units. (Verified: $100 LUNA-USDC amplp ≈ 53.8 amplp ≈ 53,805,091 base units at $1.85856.)
- Amplified vs non-amplified: they are DIFFERENT denoms for the SAME pool. Plain Astroport LP (e.g. LUNA-USDC terra1s275… $0.53647) vs Eris amplp (factory/terra1zly98…/0/stable/amplp $1.85856). amplp_price = LP_price × exchange_rate. The compounder `asset_configs` query is the AUTHORITATIVE mapping: each entry pairs underlying `asset_info` (plain LP) ↔ `amp_denom` (amplp). Use that to link the two.
- DECIMALS GOTCHA: most amplp vaults are decimals=6 (1.0 amplp = 1e6 base units) but some (BTC-style) are decimals=8. READ each denom's decimals from source (price feed / tokenfactory metadata) — do NOT hardcode 6, or the send-helper is off by 100× on the 8-decimal vaults.
- Data sources: balances via bank/tokenfactory balance query on the address; amplp price via compounder amplp_exchange_rates × LP price (our own pricing, 2.10c) with the Eris backend as cross-check; mapping via asset_configs.
- **EXTENSION — underlying-token breakdown ("$100 amplp = 47 LUNA + 31k CAPA"):** decompose a holding into its constituent tokens (amounts + USD each) so users see real exposure and can watch composition drift (= seeing IL form). Path: amplp_balance × exchange_rate = LP_amount (HAVE); then your_token_i = pool_reserve_i × (LP_amount / total_LP_supply); then × token price (HAVE). **RESOLVED (item 2.10i): the Astroport PAIR `pool`/`share` query** returns reserves + LP supply (and `share` returns the per-token decomposition directly). Confirmed on-chain, reconciles with the off-chain API. **CRITICAL: do NOT assume 50/50.** xyk pools sit ~50/50 by value but PCL pools (most TLA pools) concentrate liquidity and are NOT necessarily balanced — must read ACTUAL reserves from the pool query or token amounts are wrong. With real reserves the breakdown is exact. This same pool-reserves query also feeds 2.10c (LP USD pricing from reserves) and 2.10b (gross-APR denominator) — so pulling it serves three features at once.

### 2.10d [HIGH] Astroport `pools.getAll` API — the missing pool data, all in one call (FROM HAR)
HAR capture of app.astroport.fi/pools revealed the frontend's data endpoint:
`https://app.astroport.fi/api/trpc/pools.getAll?input={"json":{"chainId":"phoenix-1"}}` (tRPC GET; response at result.data.json = array of 275 phoenix-1 pools). This single call provides nearly everything the APR + decomposition features needed. Per-pool fields:
- `poolAddress`, `lpAddress`, `name`, `poolType` (concentrated/xyk/stable), `feeRate`.
- `poolLiquidityUsd` + `poolLiquidity` (total LP supply, base units) → implied LP unit price = liqUsd/(supply/1e6).
- `poolStakedLiquidityUsd` + `poolStakedLiquidity` → staked portion.
- `assets[]`: per token {symbol, address/denom, amount (RESERVE, base units), price, precision} → THE pool reserves we were missing (item 2.10b/2.10c/decomposition).
- APR pre-split: `tradingFees.apr` (base fee yield), `astroRewards.apr` (ASTRO incentives), `totalRewards.apr` (their displayed total), plus `rewards[]` with per-token emission rate + USD/day. `dayVolumeUsd`.

**Verified against known data (LUNA-USDC):** reserves × prices = $688,280 ≈ poolLiquidityUsd $687,891 ✓; implied LP price $0.53592 ≈ Eris backend $0.53647 ✓; value split 50.5/49.5 (NOT 50/50 — confirms must-read-reserves). Decomposition now exact: $100 LP = 810.13 LUNA + 49.57 USDC, from real reserves.

**CLOSES THE RECONCILIATION GAP (item 2.10b):** Astroport's APR here (tradingFees + astroRewards ≈ 2.4% for LUNA-USDC) is ONLY the Astroport side. It does NOT include TLA/Eris gauge bribe rewards (~58% gross we computed). So FULL gross for a TLA-staked LP = Astroport total (~2.4%) + TLA rewards (~58%). The realized compounder exchange_rates.apr (~59%) captures BOTH sides; our earlier "gross" had only the TLA side, which is why it nearly matched realized instead of exceeding it. Now both components are sourced → the breakdown can reconcile: (Astroport base+ASTRO) + (TLA gauge rewards) − 10% take − 8% compound fee ≈ realized. THIS is the full picture.

**Source caveat (Camron's standing concern):** pools.getAll is Astroport's OFF-CHAIN tRPC API (convenient, complete, but centralized — same trust class as the Eris backend). Use it as a fast primary + cross-check, but the source-of-truth path remains the on-chain pair `pool` query (reserves) + on-chain oracle (price); reconcile API vs chain and flag divergence. The HAR also shows Astroport prices via `tokens.byChain` and `tokens.getPrice` tRPC endpoints (another off-chain price source for cross-checking).

**Revised feature dependency:** the Astroport pool reserves/supply (needed by 2.10b denominator, 2.10c LP pricing, and the amplp decomposition) are now available BOTH off-chain (pools.getAll, instant) and on-chain (pair `pool` query, source-of-truth). Build against the API first to ship fast, swap/cross-check with on-chain for the trust layer.

### 2.10e [MED] Skeletonswap API migration — resolved from HAR (closes open item #5)
HAR of skeletonswap.backbonelabs.io confirmed the new endpoint (the migration that broke the SS cron / made test.html hide SS lines):
`https://dex.warlock.backbonelabs.io/api/pools/phoenix-1` → `{chain_id, pools[], total, limit, offset, timestamp}` (40 pools, paginated via limit/offset). Per pool: `pool_id` (e.g. "ampLUNA-LUNA"), `pool_address`, `block_height`, `reserve_0`, `reserve_1`, `total_share`, `tvl_usd`, `volume_24h_usd`, `volume_7d_usd`, `apr_7d` (some null currently), `token_0`/`token_1` each {denom, symbol, name, decimals, logo_url}. This is the same reserve+share+tvl shape as Astroport's pools.getAll — so the SS cron should point at this URL and map these fields. SS also proxies prices via skeletonswap.backbonelabs.io/api/coingecko?ids=... → {id:{usd:price}} (a CoinGecko passthrough — another off-chain cross-check source, same trust caveat). Action: update SS cron to /api/pools/phoenix-1; once flowing, re-enable SS lines in test.html (currently hidden).

### 2.10f [MED] Token icons — source strategy (Camron wants icons on site, incl. blended LP pairs)
Both DEXs pull token icons from GitHub raw (free, hotlinkable, no API key):
- **Cosmos Chain Registry** (SS uses this): `raw.githubusercontent.com/cosmos/chain-registry/master/<chain>/images/<token>.{svg,png}` — broadest coverage, multi-chain (terra2, cosmoshub, neutron, injective, migaloo, stride, _non-cosmos/ethereum). Best PRIMARY source.
- **Astroport token list** (Astroport uses this): `raw.githubusercontent.com/astroport-fi/astroport-token-lists/main/img/<token>.svg` — good FALLBACK for Terra-specific tokens.
- The SS `/api/pools` and Astroport `pools.getAll` responses BOTH include a per-token `logo_url`/`logoUrl` directly → simplest path: read the icon URL straight from the pool data we already fetch, no separate icon registry needed. Cache/proxy through our own domain to avoid hotlink fragility + control sizing.
- **Blended LP icon (Camron's idea):** for an LP/amplp, render the two token icons overlapped (e.g. token_0 back-left, token_1 front-right, slight offset) — pure CSS/SVG composition of the two `logo_url`s, done client-side. No special asset needed; build a small `<PairIcon a b />` that takes the two token icon URLs from the pool/asset_configs mapping. For amplp specifically, the underlying pool's two tokens are known via asset_configs (asset_info → the LP → its two constituents from pools.getAll assets[]), so the same blended icon represents the amplp.

### 2.10g [KEYSTONE] How Eris builds its displayed numbers — confirmed reproducible from source (Eris HARs)
HARs of erisprotocol.com liquidity + vote tabs reveal the ENTIRE data assembly — and it confirms we can reproduce every number Eris shows:
- `backend.erisprotocol.com/prices` → token USD prices (CoinGecko-backed; have it, 2.10c).
- `app.astroport.fi/api/pools?chainId=phoenix-1` (REST variant — simpler than the tRPC pools.getAll; 275 pools) → per pool: totalLiquidityUSD, poolTotalShare, assets (reserves), dayVolumeUSD, dayLpFeesUSD, rewards[], and `yield`:{poolFees, astro, externalRewards, total}. (LUNA-USDC: yield.total 2.423%, poolFees 2.200%, externalRewards 0.223% — matches the tRPC numbers; both Astroport endpoints serve the same data.)
- `phoenix-rpc.erisprotocol.com` → batched `abci_query` (10 wasm reads/batch) against the gauge/staking/compounder/escrow contracts WE ALREADY MAPPED.
**Conclusion: Eris's displayed APR/VP/rewards are computed CLIENT-SIDE from (these contract queries + Astroport pool data + prices). Nothing proprietary server-side.** This is the keystone that validates the whole reconciliation plan (2.10b): every figure on Eris's UI is reconstructable from sources we now hold. The earlier "unexplained per-pool factor" in Eris's displayed APR is just their client-side formula combining these inputs — not hidden data. We can match it AND show our own first-principles net APR beside it.

### Icon source PRIORITY (trust-ordered)
Priority order for token icons (chosen for reliability/uptime of the asset host):
1. **Eris** — `www.erisprotocol.com/assets/tokens/<token>.webp` (self-hosted, .webp, fast). PRIMARY.
2. **Astroport** — `raw.githubusercontent.com/astroport-fi/astroport-token-lists/main/img/<token>.svg`. FALLBACK.
3. Cosmos Chain Registry as last-resort wide-coverage fallback.
Not Skeletonswap (lowest reliability for asset hosting). Cache/proxy through our own domain regardless. Blended LP/amplp icon = client-side overlap of the two constituent icons (PairIcon), sourced in this priority.

### 2.10h [HIGH] Oracle/price-health panel — Camron's feature: surface + compare all price feeds live
Camron's vision: a panel that surfaces prices from ALL feeds side by side and shows their health + parity:
- **Astroport** (its EMA/TWAP-derived pool price + its `/api/pools` yield/price) — "fancy formulas".
- **Eris** (`backend.erisprotocol.com/prices`).
- **CoinGecko** (direct, and as seen via the SS /api/coingecko passthrough).
- **Our own on-chain** (pool reserves / on-chain oracle — the source-of-truth path, 2.10c).
Per token show: each source's price, %-divergence from the median/anchor, feed freshness/last-update, and a green/amber/red parity status. Flag when any feed diverges beyond tolerance (we already found real cases: ASTRO 9×, KUJI 18×, MARS 146× across bridged variants in the Eris feed). This is BOTH a user trust feature AND our own data-health guard — if feeds disagree, downstream USD/APR numbers are suspect and we show a warning instead of a confident-but-wrong figure. Directly addresses Camron's standing oracle-manipulation/parity concern. Feeds: all identified (Eris backend, Astroport REST+tRPC, CoinGecko, on-chain reserves) — nothing left to discover, this is buildable now.

### 2.10i [CONFIRMED] Astroport pair on-chain queries — source-of-truth reserves + the on-chain oracle
Ran the LUNA-USDC pair (terra1v3lqxl0…) on-chain. CONFIRMED + cross-checked vs the off-chain API:
- **`pool{}`** → `{assets:[{info,amount}], total_share}`. On-chain reserves + LP supply. Cross-check vs pools.getAll: total_share IDENTICAL to the unit (1,283,564,891,005); reserves differ ~0.22% (different block — pools trade continuously). **This is the parity check working: API ≈ chain within trade-noise, supply exact.** The oracle-health panel (2.10h) would PASS this pair.
- **`pair{}`** → `{asset_infos, contract_addr, liquidity_token, pair_type}`. LUNA-USDC: liquidity_token terra1s275… (matches registry), pair_type {custom:"concentrated"} → CONFIRMS PCL. Use pair_type to auto-classify PCL vs xyk vs stable per pool.
- **`share{amount}`** → for N LP base units, returns each underlying token amount. THE decomposition primitive computed BY THE CONTRACT (no manual math): 1.0 LP = 4.351450 LUNA + 0.265066 USDC, matches reserve/supply calc exactly. Either path works; `share` is the authoritative one-call version. ($100 LP = ~811 LUNA + ~49 USDC, fully on-chain.)

**Additional pair queries available (from contract-state screenshot) — pull as needed:**
- **`cumulative_prices`** [HIGH, CONFIRMED] → `{assets, total_share, cumulative_prices:[[asset_a, asset_b, cumulative_value], ...]}` — two entries, both directions. Each cumulative_value is a price-SECONDS ACCUMULATOR (running total of price×time). **A single read is meaningless; the TWAP = (cum_T2 − cum_T1)/(T2 − T1) between two reads.** So the cron stores (cumulative_value, timestamp) each epoch → derives the manipulation-resistant TWAP from consecutive snapshots. SAME mechanism as Astroport's 1-day-TWAP oracle. This is the on-chain "truth" price feed for the oracle-health panel (2.10h) and 2.10c.
- **`lp_price`** [NEEDS UNIT CONFIRMATION] → returned 1.0741 for LUNA-USDC. **This is NOT USD** (our reserves×price USD LP value ≈ $0.5365; lp_price ≈ 2× that). Most likely an internal PCL valuation in price-scale/peg units or an xcp-style virtual price — NOT directly displayable. **Do NOT show lp_price as USD without pinning its denomination.** The trustworthy USD LP value remains reserves×price (or `share` × token prices). lp_price is a cross-check only once its unit is confirmed. (~60% confidence it's peg/scale-denominated; flagged rather than guessed.)
- **`simulation{offer_asset:{info,amount}}`** [MED, field confirmed] / **`reverse_simulation{ask_asset:{info,amount}}`** → swap quote (return amount incl. spread/fee) → effective spot price + slippage; useful for thin-pool manipulation-risk detection. (Empty {} errors name the required field.)
- **`asset_balance_at{block}`** [MED] → historical reserve at a block height → free historical TVL/LP-value backfill straight from the pair, no own-snapshot needed.
- `reverse_simulation` / `compute_d` / `config` [LOW] → swap-input inverse / PCL invariant D / fee+ema_half_time params. Diagnostic/reference only.

**Net:** the on-chain source-of-truth layer for pool data is now CONFIRMED and reconciles with the off-chain APIs. pool/pair/share cover reserves+supply+type+decomposition; lp_price + cumulative_prices give the contract's own LP price and the manipulation-resistant oracle for the health panel.

### 2.10j slippage / zap-impact / arb-signal — features off `simulation` + `cumulative_prices`
All three build on the pair `simulation` query, now CONFIRMED with real output:
- `simulation{offer_asset:{info,amount}}` → `{return_amount, spread_amount, commission_amount}` (all in OUTPUT-asset base units).
- `reverse_simulation{ask_asset:{info,amount}}` → `{offer_amount, spread_amount, commission_amount}`.
**Two distinct costs:** `commission` = the pool fee (≈0.147% flat on LUNA-USDC, the PCL fee tier); `spread` = price impact (grows with size). Show them SPLIT ("fee X% + price impact Y%").
**Measured slippage curve (LUNA-USDC, sell LUNA):** 1 LUNA → 0.016% spread; 100 → 0.019%; 10,000 (~$620) → 0.21%; 100,000 (~$6.1k) → ~2.0% spread / ~1.9% impact. Curve is ~flat then accelerates with size — exactly what a large-zap user must see before committing. Effective price = return/offer; impact = deviation from the 1-unit (or TWAP) reference.

1. **Slippage display** [HIGH, easy] — quote several sizes ($100/$1k/$10k/$100k) per pool → a price-impact CURVE. effective_price = return/offer; impact = deviation from TWAP. Real, on-chain, trustworthy.

2. **Zap-impact preview** [HIGH, on-mission — the standout] — Eris zap = swap ~half the input then add LP; the internal-swap slippage is invisible until after committing ("put in and pray"). Simulate the swap leg BEFORE commit → show "zap $X LUNA → ~$Y LP, costing ~$Z (W%) to slippage+fees", compare across pools for cheapest entry. GROUNDED example: a $10k LUNA zap swaps ~half (~80k LUNA) → from the curve ~1.5% spread + 0.147% fee ≈ ~1.6% on that half ≈ ~0.8% on the whole zap, shown live before commit. Pure consumer-protection (informs, doesn't transact). All from simulation + reserves we have.

3. **Arb signal / alerts** [MED] — spot-vs-TWAP divergence beyond tolerance = an arbitrage signal. Surface as an ALERT (informational, risk-free, on-mission).

**DAO-arb bot (Camron's actual intent — recorded fairly):** the goal is DAO-treasury-funded arb to (a) support TLA pairs by adding volume and (b) generate treasury revenue — a legitimate, good-faith aim, NOT user-extractive. So intent is fine. Remaining honest cautions are PRACTICAL, not ethical: (1) Astroport/Terra arb is highly competitive — pro bots with co-located infra + mempool access capture gaps in ms; a dashboard-speed bot likely loses the race and bleeds gas on reverts, so the edge may not be real without serious infra. (2) Perception: a transparency dashboard that also actively trades invites "is it front-running me?" questions — manageable, but only with explicit disclosure (clearly separate the DAO arb wallet from the dashboard, publish what it does). (3) Risk class shifts to key custody + financial loss — needs DAO governance sign-off, not a solo deploy. VERDICT: not out-of-scope, but treat as a SEPARATE, later, governance-approved project with realistic edge analysis first; the dashboard ships the informational versions (slippage, zap-impact, alerts) now, which deliver most of the "support the pairs / protect users" value with none of the risk.

### 2.20 [CONFIRMED] ve3-connector-alliance — the gauge→Alliance-rewards bridge (NEW contract mapped)
Mapped the stable-bucket connector terra1ym2495f63mdx63tu96085x2vf3xpy9z9k5urxwhvmf9jldm99q5qr4q6n8 (label `ve3-connector-alliance` v1.0.0, code_id 3120). This is how each TLA gauge bucket plugs into Terra's Alliance module to earn the Alliance reward distribution (paid in LUNA). ONE connector PER gauge — config.gauge identifies which (this one = "stable"); expect 4 total (stable/project/bluechip/single), each with its own zluna + vt denoms (the per-bucket zasset_denom values already seen in the compounder asset_configs match these → confirms the linkage).

**`config{}`** → {global_config_addr (the master ve3 directory), reward_denom "uluna", zasset_denom (factory/<this>/zluna), alliance_token_denom (factory/<this>/vt = the Alliance virtual-stake token), alliance_token_supply (1e12), gauge ("stable"), lst_hub_addr (the ampLUNA/LST hub terra10788fkz…), lst_asset_info (cw20 ampLUNA terra1ecg…lvsct)}.
**`state{}`** → {last_exchange_rate, share_exchange_rate (zluna share→underlying), total_shares, stake_available, stake_in_contract (= the connector's ampLUNA balance; verified = on-chain balance 53,259.478648 ✓), taken, harvested (taken==harvested = rewards fully reconciled)}. This is the bucket's Alliance-stake economics.
**`validators{}`** → array of 52 terravaloper… addresses = the whitelisted validator set the connector delegates its virtual stake across (uniform distribution for decentralization).

**Relevance:** completes the reward picture — TLA pool LPs feed gauges; each gauge's connector stakes ampLUNA into Alliance and earns LUNA rewards distributed back. The vt (virtual-stake) and zluna (receipt) denoms are how this is tracked. For the dashboard this is mostly backend/accounting context (it explains where part of the reward APR originates), not a per-user tile — but the per-bucket connector state is a clean source for "Alliance rewards earned by bucket" if we want to show reward provenance.

### COVERAGE — Eris TECH footer fully accounted for (complete contract inventory)
The Eris site footer lists the full ve3 contract set. Coverage confirmed against it:
- **5 singletons (all schema+data confirmed):** asset-gauge (distributions + per-wallet user_info/user_shares/user_first_participation/user_pending_rebase), bribe-manager (user_claimable, next_claim_period), global-config (all_addresses directory), voting-escrow (lock_info, tokens{owner}, all_tokens — 431 locks), asset-compounder (config, asset_configs, amplp_exchange_rates, exchange_rates; user_infos input-fields known, element-encoding deferred to build time).
- **2 repeating contract TYPES, one instance per gauge bucket (stable/project/bluechip/single):**
  - ve3-asset-staking ×4 (code_id 3585, v1.8.2) — schema: config, whitelisted_assets, whitelisted_asset_details, reward_distribution, staked_balance, pending_rewards, all_staked_balances, all_pending_rewards, all_pending_rewards_detail, total_staked_balances, pool_stakers. Confirmed on stable + project instances; bluechip/single are same code_id = guaranteed same schema (only per-bucket VALUES differ).
  - ve3-connector-alliance ×4 (code_id 3120, v1.0.0) — schema: config, state, validators. Confirmed on stable + project; bluechip/single same code_id = same schema.
- **Reward-share-by-bucket** (stable earns most, others split differently) is DATA, not a new schema — read from asset-gauge `distributions` + gauge vote weights per epoch. Already-mapped query; the split falls out of the values.

**Cron pattern:** write the query logic once per TYPE, iterate the 4 bucket addresses (from global-config all_addresses). No per-bucket schema differences.

**→ Discovery phase COMPLETE.** Every footer contract is schema-confirmed. The only deferred detail is the asset-compounder user_infos element encoding (resolve from the contracts-ve3 schema at build time; lowest-priority feature, nothing depends on it).

### IDEAS: site features the contract data unlocks (running list)
Things the on-chain data makes possible — capture now, prioritize later. Each notes the source query.

**Per-wallet / member tools**
- Wallet lookup: enter any address → VP, locks, claimable bribes + rewards, first participation (gauge `user_info`+`user_shares`+`user_pending_rebase`, bribe `user_claimable`, staking `pending_rewards`, escrow `tokens{owner}`).
- **Early-user / loyalty recognition**: `user_first_participation` gives the epoch each wallet first voted → "OG voter" badges, longevity leaderboards, retro-reward eligibility lists. Cheap to compute, strong community signal.
- Member detail page (dao-tla.html, already queued): real VP + lock breakdown per member from escrow `tokens{owner}` + `lock_info`, instead of reconstructed.
- Lock explorer: all 431 veLUNA locks (escrow `all_tokens` → `lock_info` each) — size, decay, owner, time-to-unlock. A unique view nobody else has.

**Health / trust (the data-integrity direction)**
- Cross-contract invariant panel: gauge total_gauge_vp vs escrow total_vamp; pool emissions sum vs bucket pot; staked-on-chain vs our staked. Green/yellow/red. (This is the trust-builder from earlier.)
- "Verified against chain" badge per tile: show that a number was reconciled to its source contract this epoch.
- Reward-flow tracker: watch "Distribute Take Rate"/"Distribute Bribes" txs → confirm each epoch's distribution actually fired, flag if missed.

**History / trends (the "save for history" goal)**
- Per-pool VP & reward-share over time (gauge last_distributions each epoch) → see pools gaining/losing favor.
- APR history per pool (net APR from staking contracts each epoch) → trend lines, not just current.
- Bribe history per pool (bribe-manager bribes each epoch) → which pools attract bribes, ROI of bribing.
- Total VP / fixed vs decaying split over time (escrow total_vamp) → lock-commitment health of the whole system.
- Take-rate revenue over time (sum of take-rate txs) → protocol revenue trend.

**Self-maintaining registry (your dynamic-LP point — see 2.16)**
- LPs auto-add/remove as the gauge whitelist changes; no manual pool list to maintain. New pool whitelisted on Eris → appears on our dashboard next epoch automatically; delisted → drops off. The gauge + global-config make this fully hands-off.

### REF-A: Address/ID-argument queries (errored on empty {} — need an arg)
These all failed with "missing field" only because they were run with `{}`. Each needs an argument — documented here so wiring them later is trivial. Arg shape is the chainscope/CosmWasm smart-query JSON.

**ve3-asset-gauge** (terra1hfksr…msd3lj) — need `{"user": "<addr>"}`:
- `user_info{user}` → that wallet's VP {voting_power, fixed_amount, slope}
- `user_shares{user}` → wallet's share of each gauge
- `user_first_participation{user}` → the epoch/time the wallet first voted ← EARLY-USER signal
- `user_pending_rebase{user}` → unclaimed rebase for the wallet
- `gauge_info{...}` / `gauge_infos{...}` → likely need a gauge name + period arg

**ve3-bribe-manager** (terra1tuuw…9037wh) — need `{"user": "<addr>"}`:
- `user_claimable{user}` → `{start, end, buckets:[{gauge, asset:(the pool voted), assets:[{info,amount}]}]}` — CONFIRMED with a populated example. start/end = period window (e.g. 186..186 = current epoch claimable). Per pool the wallet voted, the bribe tokens + amounts claimable. Empty `{start:0,end:0,buckets:[]}` when nothing to claim. Example wallet: 9 pools, totals 1,735 ASTRO + 371 LUNA + 672 CAPA. This is "what bribes can I claim" for any wallet, and ties a wallet's votes (user_shares) to its bribe income.

**ve3-voting-escrow** (terra1uqhj…3l62zg) — CW721, need `{"token_id":"<id>"}` or `{"owner":"<addr>"}`:
- `lock_info{token_id}` / `lock_vamp{token_id}` → per-lock detail + VP
- `nft_info{token_id}` / `all_nft_info{token_id}` / `owner_of{token_id}` → NFT metadata + owner
- `tokens{owner}` → all lock NFTs a wallet holds
- `approval{token_id,…}` / `approvals{token_id}` / `all_operators{owner}` → CW721 approvals

**ve3-asset-staking** (per bucket) — address-arg (from the earlier query set):
- `staked_balance{...}` / `pending_rewards{...}` / `all_pending_rewards_detail{...}` / `pool_stakers{...}` → per-wallet staked + pending rewards

Pattern: the plural/no-arg forms (`user_infos`, `bribes`, `all_tokens`, `total_staked_balances`) give the whole set for crons; the singular arg forms give one wallet/lock for an interactive lookup.

### 3.x [FEATURE] Per-wallet lookup on the site (address-input gauge queries)
The address-arg queries (`user_info`, `user_shares`, `user_first_participation`, `user_pending_rebase` on the gauge; plus `pending_rewards`, `staked_balance` etc. on the staking contracts) enable a "enter any Terra address → see their VP, lock shares, first participation, pending rebase + rewards" tool, all sourced from chain. Natural extension of the existing member view. Candidate page feature once the gauge cron lands.

---

## PART 3 — Cross-cutting capture standards (apply to every cron)

These are the lessons that should become **house rules** for all current and future crons, so we don't keep rediscovering them.

1. **`gauge_pool_id` on every per-pool record.** It's the only unique key. Name and name+dex collide.

2. **Distinguish "failed" from "empty."** A query that was rate-limited and returned null must be recorded as `null`/error, NOT coerced to `[]`. Silent `Array.isArray(r) ? r : []` or `r || []` after a failed `queryContract` produces empty data with no error trail — we lose the fact that capture failed. Pattern: retry with backoff across both LCD endpoints; record `entries: null` (failed) distinctly from `entries: []` (genuinely no data); push failures to a `_errors` array in the output. Keep `BATCH_CONCURRENCY <= 5` to avoid saturating publicnode LCD.

3. **Record the invariants, not just the values.** Each snapshot should be self-validating. Stamp and (ideally) assert:
   - constant-product (non-LST, non-stable) pool ≈ 50/50 by value;
   - Votion zero-sum within each bucket;
   - bucket pool-% sums to 100;
   - `staked_in_tla_usd <= depth_usd`;
   - a `gauge_pool_id` in exactly one bucket; one active gauge per name+dex+bucket.
   Emit a warning/flag in the output when an invariant breaks rather than emitting a silently-wrong number. This is how we catch the *next* class of error before it reaches the page.

4. **Stamp price source + timing per token.** Especially LSTs (redemption ratio + source). Most "errors" we chased were really a ~1% capture-timing offset vs a live UI — being able to see capture time and source instantly separates timing noise from real bugs.

5. **Heartbeat on every cron**, same schema, so the footer health check covers all of them (fuel currently lacks one).

6. **Same filenames on re-publish** (no `_v2`/`_v3`), and keep each cron's output independent so one failure doesn't cascade.

---

## PART 4 — Quick reference: what the page already defends against (so cron work can prioritize)

The page now handles these *gracefully* (so they're not user-visible emergencies), but each is a band-aid over a cron issue above:
- Duplicate gauge variants → demoted to inactive on the page (PART 1.1).
- Gauge split across buckets → collapsed to largest on the page (PART 1.1).
- Votion exceeding a pool's VP → clamped on the page (PART 1.1).
- LST uneven split → labeled "normal for an LST pair" (PART 1.2).
- Stale/missing per-epoch files (Votion, SS) → fail gracefully to null/single-epoch (PART 1.4/1.5).
- Threshold "will fall inactive" → currently shown as current-share + trend, because the Votion-only projection moves bucket % <0.05pt; a real forward projection needs PART 2.1 (boundary aDAO/member VP) to be meaningful.

**Bottom line for the cron chat:** prioritize **1.1 (gauge_pool_id keying)** and **2.1 (boundary aDAO/member VP)** — the first removes the root of most bugs we've been patching, the second unlocks the biggest missing feature and can't be backfilled. Then 1.4 (SS) for the one visible gap, then the rest. Adopt PART 3 as standing rules so new capture is right and inclusive from day one.

---

## PART 5 — BUILD & ARCHITECTURE PLAN (the new pipeline) — handover from the architecture session

This part captures the *how we build it* decisions. Parts 1–4 above are the WHAT (every contract, query, result shape, feature). Part 5 is the HOW (pipeline order, cadence, cost defense, backend target, migration). Nothing here is built yet — it's the agreed plan.

### 5.0 Status & guiding principles
- The 5 Eris singleton contracts + 2 repeating types (asset-staking ×4, connector-alliance ×4) are ALL schema-confirmed. Plus Astroport pair layer, DEX APIs, price feeds. Discovery phase COMPLETE. Only deferred query detail: asset-compounder `user_infos` assets[] element ENCODING (resolve from contracts-ve3 schema at build time; lowest priority, nothing depends on it).
- Core principles carried throughout: (1) capture RAW at every layer (reserves AND price AND source AND timestamp) — never store only computed values, because future features need to recompute history under assumptions not yet formed; (2) one change at a time, verify against real production data; (3) capture-from-source over reconstruct; (4) a failure in one layer must never corrupt another.

### 5.1 The layered pipeline (STRICT dependency order — each layer depends only on layers above it)
- **Layer 0 — Discovery/bootstrap.** `global-config all_addresses` → every contract address. Self-discovering pool registry from gauge `distributions` + DEX APIs (auto-add new / flag abandoned LPs). Everything downstream queries what this returns. (This is the user's "step 1".)
- **Layer 1 — Pricing (MUST be second — gate for everything).** Pull ALL sources: on-chain TWAP (`cumulative_prices` Δaccumulator/Δtime), reserves×price (Astroport pair `pool`), Eris backend, Astroport REST+tRPC, CoinGecko. Anchor USDC=$1, price off DEEPEST pool, prefer TWAP over spot. STORE price + source + cross-source agreement per token (feeds the oracle-health panel). A lock/stake/bribe is just a token amount until valued here — so pricing precedes all entity data. If pricing is wrong/stale, every USD downstream is wrong.
- **Layer 2 — Entities.** Pools (reserves, supply, type, APR components), locks (total count → each lock's full detail via escrow), per-bucket staking + connector state. "What exists and what's it worth."
- **Layer 3 — Participants (the expensive layer; daily/epoch ONLY, concurrency ≤5).** Build the participant set ONCE by UNION: voters (gauge `user_infos`) + lockers (escrow `all_tokens`→owners) + stakers (`pool_stakers`) + bribers (bribe data). THEN enrich each wallet with per-wallet detail. One enumeration then enrich — far cheaper than walking each list separately. Covers user's steps 4–7 (votes+participation history, staked LP detail, lock detail/VP/unlock/boost, bribers user+project).
- **Layer 4 — Rollups/derived (NO chain queries).** Weekly/monthly/yearly wrap-ups, concentration stats, "what changed" diffs, change-alerts. Pure functions of stored Layers 0–3. A monthly view reads 30 daily snapshots — it must NEVER re-query the chain.

### 5.2 Cadence / freshness tiers (driven by VOLATILITY × CONSEQUENCE, not by "importance")
- **LIVE (refresh-button; cache-served; server-side globally-capped chain refresh):** prices, exchange rates, wallet balances, pool RESERVES/depth, slippage/zap quotes. Fast-moving AND decision-driving. A mass-exit/large-inflow IS a reserve change → already on this tier, so the "finger on the pulse" case is covered here.
- **HOURLY (light cron; "recent feel"):** headline TVL, total staked, APR figures. These move slowly; APR/TVL FEEL like they should be live but barely move hour-to-hour — hourly is honest and within noise.
- **DAILY (full authoritative raw snapshot — the backbone & permanent history):** all epoch-scoped data — votes, gauge distributions, lock registry, bribes, full participant set + per-wallet detail, per-bucket connector state. Daily gives a clean dated series; participant layer never runs more than daily.
- **ROLLUPS (weekly/monthly/yearly; derived from dailies; zero chain queries).**
- **CHANGE-ALERTS:** "pool X liquidity dropped 30% in 1h" — falls out of comparing consecutive snapshots. Often MORE valuable than a live number for catching exits/inflows early (site TELLS you what moved vs you having to watch).

### 5.3 Cost & abuse defense (critical — user fears spam-to-bankrupt / being pushed out)
- The CRONS are not the attack surface — they run on your schedule regardless of traffic. The exposed surface is the LIVE-REFRESH path and any on-load fetch.
- **RULE: the browser reads YOUR CACHE; only YOUR SERVER reads the chain; the chain is backup-and-refresh, NEVER the default per-visitor fetch.** A JS/per-user rate cap is NOT real protection (anyone can script the endpoint directly). The real defense = browser hits your cached snapshot; "live" triggers ONE server-side, GLOBALLY-rate-limited chain refresh shared by all users. A million spam clicks collapse to cheap cached reads + at most one upstream refresh per interval.
- Baseline cost control is in the CODE: every loop has a hard ceiling; every cron has a max runtime + "don't start if previous still running" guard; concurrency ≤5 (publicnode limit).
- IMMEDIATE stopgap (do today): set billing caps on Render + Vercel. Imperfect (cap hit = pause, not absorb) but turns "surprise big bill" into "site pauses" — buys time to build the real defense. Does NOT require shutting down or starting over.

### 5.4 UX: "feels live & coherent" WITHOUT timestamp clutter (user: "wish it all just worked")
- Goal is COHERENT/CONFIDENT, not literally-everything-live (impossible to deliver honestly + is the bankruptcy surface). These are the same design: "feels live, is actually cached+capped" IS the cost defense.
- Page loads INSTANTLY, fully populated from cache (instant-complete feels more alive than spinners). Live values (prices/reserves/balances) tick over smoothly IN PLACE on refresh. ONE quiet ambient "updating…/updated just now" cue for the whole page — NOT per-tile "snap taken XXXX" stamps. Change-alerts surface big moves so the site TELLS you what's happening. Show last-known value (never a blank/perpetual spinner) in gaps.

### 5.5 Target architecture: NestJS + Postgres backend (Eris's recommendation — endorsed, with sequencing caveat)
- A real backend (NestJS, same TS language as the site) + Postgres DB is the proper-foundations version of EXACTLY this design: runs the layered jobs in order, schedules all cadences in one place, and — critically — IS the "server that accepts requests" required for the server-side rate-capped refresh in 5.3. Postgres makes rollups + change-alerts trivial (query rows vs aggregating JSON files) and fixes the current GitHub-JSON-files mess.
- **SEQUENCING (important): the backend is the DESTINATION, not the starting point.** The hard part — the pipeline LOGIC (which contract, which query, what order, valuation, source reconciliation) — is identical whether it runs in scripts-writing-JSON or NestJS-writing-Postgres, and that logic is ALREADY worked out in Parts 1–4. Recommended path: (1) prove the new pipeline's logic in the familiar tool (scripts, like current crons), produce correct trustworthy snapshots, verify against old system; (2) THEN port the proven logic onto NestJS+Postgres as its permanent home. Avoids learning unfamiliar infra AND building the pipeline at the same time (one hard new thing at a time). The backend migration IS the "clean up the mess later" step, done properly.

### 5.6 Migration approach (NOT a rip-and-replace)
- New pipeline is the eventual AUTHORITATIVE layer but is migrated INCREMENTALLY, never in one cutover (a wholesale swap reintroduces single-point-of-failure risk the whole design avoids).
- Name new Render crons with a `v2`-style suffix so the old ones are unambiguously safe to delete later (inverse of the no-`_v2`-on-files rule — here the suffix marks parallel infra to retire).
- PARALLEL-RUN + DIFF: run new alongside old (new writes to new repos/DB), diff new vs old vs live chain field-by-field. Discrepancies = the lingering hardcoded things to fix — surfaced WITHOUT taking down production. The TEST PAGE is the safe surface for this: it's hosted but hidden/WIP, so commit it to the tla-stats slot running off the OLD system, then strip old wiring from the test page to see what's hardcoded/lingering, then tie in the new pipeline and verify it functions + is trustworthy. Production users never see a broken page.
- Flip ONE tile/data-type to the new source only after its new output matches (or provably beats) the old for long enough to trust. Old system stays as safety net + correctness oracle until then.

### 5.7 OLD DATA: freeze, don't delete (intellectual-honesty point)
- User's instinct: "we don't trust the old system, so we don't really have history — start trustworthy history now." HALF right: yes, the new raw/multi-source/verified series is the trustworthy basis going forward. HALF a trap: "don't trust it" ≠ "it's worthless" (not audited which parts are wrong); and most of it CANNOT be re-queried (can't read past block state later). Asymmetry: keeping old data = a few ignored cheap repos; deleting = the ONLY copy of the past, gone permanently, on a hunch.
- DECISION: FREEZE old data where it is, mark "legacy / unverified," start the trustworthy new series now, leave old in cold storage. Never touch it again = lost nothing; want a 2-yr trend later = very glad it's there to stitch in with an asterisk. Cleanup of the messy Render/GitHub (multiple old cron attempts, scattered repos) happens LATER, once the new system is proven — not now.

### 5.8 Context to retain (the WHY behind the rigor — keep in conversation, NOT in public/committed files)
- The whole verify-everything-from-source discipline is grounded in a real concern about trusting data in an ecosystem with opaque inter-protocol relationships. Mission: NOT to accuse anyone — to give the community VERIFIABLE FACTS so no one is hurt by any bad actor. The symmetric transparency tools (oracle-health panel showing all feeds + divergence, APR breakdown from first principles, fee transparency, everything-from-source) embody "show facts, don't accuse" — radical transparency is neutral by construction.
- KEEP all sensitive inter-protocol context in conversation only; never write it to committed/crawlable files. Health-panel wording stays descriptive ("sources disagree by X%") never attributive ("Y is misreporting"). Public files have been scanned clean of sensitive terms — maintain that.

---

## PART 6 — QUERY COOKBOOK (confirmed request → response → use). Build off THIS.

Every query below is CONFIRMED against real chain/API output. Format: REQUEST JSON, RESPONSE shape, USE.
Amounts are base units (÷10^decimals; most tokens 6, wBTC/BTC-style 8, some ETH-side 18 — read decimals from source, never hardcode). Bootstrap all addresses from global-config `all_addresses`.

### CONTRACT ADDRESSES (the fixed set)
- global-config: `terra1hwxg6s732eparz3ys7sa4t5f64ngpd2w8syrca6z7ckv3fs9uqnsvrpcqa`
- asset-gauge: `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj`
- bribe-manager: `terra1tuuwm8yrj54qeg0c8xu00aha9ryatyhtczq8qq2q8tntuw0auzas9037wh`
- voting-escrow: `terra1uqhj8agyeaz8fu6mdggfuwr3lp32jlrx5hqag4jxexde92rzkamq3l62zg`
- asset-compounder: `terra1zly98gvcec54m3caxlqexce7rus6rzgplz7eketsdz7nh750h2rqvu8uzx`
- asset-staking (×4): stable `terra1v399cx9drllm70wxfsgvfe694tdsd9x96p9ha36w7muffe4znlusqswspq` · project `terra1awq6t7jfakg9wfjn40fk3wzwmd57mvrqtt3a39z9rmet7wdjj3ysgw3lpa` · bluechip `terra14mmvqn0kthw6sre75vku263lafn5655mkjdejqjedjga4cw0qx2qlf4arv` · single `terra1qdz5qgafx88kp5mf6m2tah8742g4u5g2cek0m3jrgssexexk7g4qw6e23k`
- connector-alliance (×4): one per gauge; bootstrap from all_addresses (stable connector seen = `terra1ym2495f63mdx63tu96085x2vf3xpy9z9k5urxwhvmf9jldm99q5qr4q6n8`)
- Astroport pair example (LUNA-USDC): `terra1v3lqxl0eyte9x3nhdgcj8hwvjq76aupnnzz0yll8mxs5cckc29pqvg2scu`

### GLOBAL-CONFIG
- REQ `{"all_addresses":{}}` → RESP: directory of every ve3 contract by role. USE: Layer-0 bootstrap; everything else uses these addresses.

### ASSET-GAUGE (pool reward registry + per-wallet votes)
- REQ `{"distributions":{}}` → RESP: per-pool reward distribution registry. USE: authoritative pool list + per-bucket emission weighting (the reward-share-by-bucket split is DATA here).
- REQ `{"last_distribution_period":{}}` → RESP `{...}` (e.g. 186). USE: current distribution epoch.
- REQ `{"user_info":{"user":"<addr>"}}` → RESP `{voting_power, fixed_amount, slope, gauge_votes:[{gauge,period,votes:[[asset,bps]]}]}`. USE: a wallet's VP + exactly which pools it votes and weights.
- REQ `{"user_shares":{"user":"<addr>"}}` → RESP per-gauge `{asset,period,user_vp,total_vp}`. USE: live % pool influence (use THIS for member %).
- REQ `{"user_first_participation":{"user":"<addr>"}}` → RESP `{period}`. USE: tenure / early-user signal.
- REQ `{"user_pending_rebase":{"user":"<addr>"}}` → RESP `{rebase}`. USE: unclaimed rebase.
- NOTE field name here = `user`.

### BRIBE-MANAGER
- REQ `{"user_claimable":{"user":"<addr>"}}` → RESP `{start,end,buckets:[{gauge,asset,assets:[{info,amount}]}]}` (empty = `{start:0,end:0,buckets:[]}`). USE: per-pool claimable bribe tokens for a wallet.
- REQ `{"next_claim_period":{}}` → RESP claim period info.

### VOTING-ESCROW (veLUNA locks — 431 total)
- REQ `{"num_tokens":{}}` → RESP `{count}` (431). REQ `{"all_tokens":{"limit":N,"start_after":"<id>"}}` → paginated token_id list.
- REQ `{"tokens":{"owner":"<addr>"}}` → RESP a wallet's lock NFT ids.
- REQ `{"lock_info":{"token_id":"<id>"}}` → RESP `{owner, from_period, asset:{info,amount}, underlying_amount, coefficient, start, end:{period}, slope, fixed_amount, voting_power}`. USE: THE per-lock source — unlock epoch (end.period), VP, underlying, decay (slope). Skip lock_vamp/nft_info (redundant).
- REQ `{"total_vamp":{}}` → decaying + fixed total VP.

### ASSET-STAKING (×4 buckets — same schema each; field name = `address`)
- REQ `{"pool_stakers":{"asset":<asset_info>,"limit":N,"start_after":<asset_info>}}` → RESP paginated `[{user,shares,balance}]`. USE: staker enumeration + concentration; filter balance>0 (dust = shares:1/balance:0).
- REQ `{"staked_balance":{"asset":<asset_info>,"address":"<addr>"}}` → RESP balance; **"not found" storage error when no position = treat as ZERO** (not a failure).
- REQ `{"all_pending_rewards":{"address":"<addr>"}}` → RESP `{data:[]}` clean-empty when none.
- `yearly_take_rate` = 0.1 (10% staking take).

### ASSET-COMPOUNDER (auto-compounding vaults; field name = `addr`)
- REQ `{"config":{}}` → RESP `{fee:0.08 (8% compounder fee), fee_collector, deposit_profit_delay_s:14400, denom_creation_fee, ...}`.
- REQ `{"asset_configs":{}}` → RESP array, each `{asset_info, gauge, staking, amp_denom, zasset_denom, reward_asset_info, fee(null=global 8%), total_bond_share}`. **asset_info is a bare string enum: `{"cw20":"terra1..."}` OR `{"native":"factory/.../uLP"}`** (older Astroport = cw20, newer tokenfactory = native; per pool). USE: AUTHORITATIVE LP↔amplp map + LP token-type map.
- REQ `{"amplp_exchange_rates":{}}` → current amplp→LP rate per vault. REQ `{"exchange_rates":{}}` → rate history + `apr` field (PER-PERIOD/daily realized net-of-fee yield).
- IDENTITY: amplp_usd = LP_usd × amplp_exchange_rate (verified LUNA-USDC: 1.85856/0.53647 = 3.46445).
- DEFERRED (build time): `{"user_infos":{"addr":"<addr>","assets":[...]}}` — fields known (addr + assets-list filter, NOT pagination; bare {addr} OOGs), but exact `assets[]` element JSON encoding unconfirmed (bare asset_info, {asset_info,gauge}, and [asset_info,gauge] tuple all rejected "Invalid type"). Resolve from contracts-ve3 schema / CosmWasm query schema. Lowest priority.

### CONNECTOR-ALLIANCE (×4 buckets — gauge→Alliance reward bridge; same schema each)
- REQ `{"config":{}}` → RESP `{global_config_addr, reward_denom:"uluna", zasset_denom, alliance_token_denom(vt), alliance_token_supply, gauge, lst_hub_addr, lst_asset_info(ampLUNA)}`.
- REQ `{"state":{}}` → RESP `{last_exchange_rate, share_exchange_rate, total_shares, stake_available, stake_in_contract(=its ampLUNA bal), taken, harvested}`. USE: per-bucket Alliance-stake economics + reward provenance.
- REQ `{"validators":{}}` → RESP array of ~52 terravaloper… (delegation targets).

### ASTROPORT PAIR (per pool — on-chain source-of-truth; this is the "truth" layer)
- REQ `{"pool":{}}` → RESP `{assets:[{info,amount}], total_share}`. USE: reserves + LP supply (source-of-truth; total_share matches off-chain API to the unit, reserves within trade-noise).
- REQ `{"pair":{}}` → RESP `{asset_infos, contract_addr, liquidity_token, pair_type}`. USE: classify PCL (`{custom:"concentrated"}`) vs xyk vs stable; get LP token addr.
- REQ `{"share":{"amount":"<N base units>"}}` → RESP `[{info,amount},...]`. USE: contract-computed decomposition (N LP → each underlying). The authoritative one-call decomposition.
- REQ `{"cumulative_prices":{}}` → RESP `{assets, total_share, cumulative_prices:[[a,b,cum],...]}` both directions. cum = price-SECONDS accumulator. **USE: TWAP = (cum_T2−cum_T1)/(T2−T1) — store (cum, timestamp) per snapshot; manipulation-resistant oracle for pricing + health panel. A single read is meaningless.**
- REQ `{"simulation":{"offer_asset":{"info":<asset_info>,"amount":"<N>"}}}` → RESP `{return_amount, spread_amount, commission_amount}` (output-asset base units). USE: effective price, slippage. commission ≈ flat pool fee (~0.147%); spread = size-dependent price impact — show SPLIT. Quote multiple sizes → slippage curve → zap-impact preview.
- REQ `{"reverse_simulation":{"ask_asset":{"info":<asset_info>,"amount":"<N>"}}}` → RESP `{offer_amount, spread_amount, commission_amount}`. USE: input-for-target-output.
- REQ `{"lp_price":{}}` → RESP a decimal. **CAUTION: NOT USD** (returned 1.0741 vs ~$0.5365 USD per-LP; likely internal PCL peg/scale unit). Do NOT display as USD until denomination confirmed. Use reserves×price or share×prices for trustworthy USD LP value.

### OFF-CHAIN APIs (convenient + cross-check; centralized = NOT source-of-truth; reconcile vs chain, flag divergence)
- Astroport tRPC: `https://app.astroport.fi/api/trpc/pools.getAll?input={"json":{"chainId":"phoenix-1"}}` → result.data.json = 275 pools, each {poolAddress, lpAddress, poolLiquidityUsd, poolLiquidity(LP supply), poolStakedLiquidityUsd, assets[](reserves+price+precision), tradingFees.apr, astroRewards.apr, totalRewards.apr, rewards[], dayVolumeUsd, poolType, feeRate}.
- Astroport REST (Eris uses this): `https://app.astroport.fi/api/pools?chainId=phoenix-1` → same data, `yield`:{poolFees,astro,externalRewards,total}. NOTE: Astroport APR = base+ASTRO ONLY; full gross for TLA-staked LP = Astroport(~2.4%) + TLA gauge rewards(~58%); realized exchange_rates.apr captures both.
- Eris prices: `https://backend.erisprotocol.com/prices` → per-denom {price_usd, decimals, display, coingecko_id?}. CoinGecko-backed for base assets; LP/amplp computed. HAS real inconsistencies across bridged variants (ASTRO 9×, KUJI 18×, MARS 146×) — cross-check only.
- Skeletonswap: `https://dex.warlock.backbonelabs.io/api/pools/phoenix-1` → {pools[]: pool_id, pool_address, reserve_0/1, total_share, tvl_usd, volume_24h/7d, apr_7d, token_0/1{denom,symbol,decimals,logo_url}}. Lowest reliability — data + icons last priority.

### PRICING METHOD (Layer 1)
Anchor USDC=$1 → derive each token by walking on-chain reserves/oracle from a USDC pair → prefer TWAP (cumulative_prices) over spot → price off DEEPEST pool → cross-check vs Eris/Astroport/CoinGecko → store price+source+agreement, FLAG divergence beyond tolerance (feeds health panel + is the manipulation guard). LP USD = reserves×price (NOT lp_price). amplp USD = LP_usd × amplp_exchange_rate.

### ICONS
Priority: Eris self-hosted `www.erisprotocol.com/assets/tokens/<token>.webp` → astroport-token-lists `raw.githubusercontent.com/astroport-fi/astroport-token-lists/main/img/<token>.svg` → Cosmos Chain Registry. NOT Skeletonswap. Cache/proxy through own domain. Blended LP/amplp icon = client-side overlap of the two constituent token icons (PairIcon); amplp's two tokens via asset_configs→LP→pool assets[].
