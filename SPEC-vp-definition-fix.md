# SPEC — VP definition fix (fixed_amount + voting_power)

Status: **APPROVED direction 2026-07-13 — build pending** · Owner: capture layer
Evidence session: 2026-07-13 (TLA UI paste + 4 chain probes, all reconciled)

## 1. The defect

Every VP figure the platform publishes is the **boost component only**
(`voting_power` = underlying_at_lock × coefficient). The vAMP escrow's actual
voting power is **`vp = fixed_amount + voting_power`** (fixed = underlying × 1),
and **the money follows `vp`**: the gauge controller's `distributions` query —
the record rewards are actually paid against — carries fixed+boost totals.

Consequences today: all absolute VP ~11% low (exact factor per lock =
`(coeff+1)/coeff`, so max-locks +11.1%, 1-week locks +100%); bucket
percentages skewed wherever lock mixes differ (LUNA-stLUNA was 26% off);
canonical "Total TLA VP ≈ 24M (max bucket)" is wrong — canonical total is
`total_vamp.vp` (~27.96M, matches TLA UI header).

## 2. Evidence (chain-confirmed 2026-07-13, keep with the code)

- `voting_escrow terra1uqhj…l62zg` → `{total_vamp:{}}` returns
  `{fixed: 3007423707566, voting_power: 24956216696464, vp: 27963640404030}`
  — vp = fixed + voting_power **by the contract's own definition**, = UI 27.96M.
- `asset_gauge terra1hfks…msd3lj` → `gauge_infos{gauge,time:"next"}` returns
  **per-pool `voting_power`, `fixed_amount`, `slope`** — we have been receiving
  fixed_amount in every response and discarding it.
- `distributions{}` per-pool `total_vp` ≈ gauge_infos boost+fixed (residual =
  votes between period flip and query — verified against our own vote-events).
- Predictive test, 9/9 project pools vs same-time UI paste, including the
  discriminating case: **LUNA-stLUNA boost+fixed = 315.17K = UI to 5 sig figs**
  (boost-only was 249.4K; a ×10/9 scalar would also fail this pool).
- User level: `user_info` voting_power 1,179,504.21 + fixed 131,056.04
  = 1,310,560.25 = UI "1.31M VP".
- ❌ The comment in `lib/capture-engine.js` (~line 1038) claiming
  "voting_power … is the actual VP that determines vote weights" is **FALSE**
  — rewrite it citing this spec. `display_voting_power_human` (fixed×10) is
  only coincidentally correct for coeff-9 locks — **retire the field**.

## 3. Changes (one patch per cron, mock-run each against real captured responses)

**A. `cron-scripts/tla-snapshot/tla-snapshot.js`**
- Per-pool: read `fixed_amount` + `slope` alongside `voting_power` from
  gauge_infos entries. Publish `vp_boost_human`, `vp_fixed_human`,
  `vp_total_human` (= boost+fixed), `slope`. `vp_human` becomes an alias of
  `vp_total_human` (schemaVersion bump; site shielded by
  `buildLegacyDataShape()` transform — verify tla-stats consumers).
- Bucket sums and `pct_of_bucket` computed over **vp_total**.
- Capture `total_vamp` (all three fields) into `totals` as the canonical
  platform-wide VP.

**B. `platform-crons/lib/capture-engine.js` + `cron-scripts/lib/capture-engine.js`**
- Member/treasury `total_voting_power_human` → fixed + voting_power; keep both
  components as explicit fields (already captured — only the total changes).
- Fix the false comment; delete `display_voting_power_human`.

**C. `cron-scripts/tla-locks/tla-locks.js`**
- Per-lock `vp_total_human` = voting_power + fixed (both already captured);
  aggregates/tiers use vp_total. System block already stores all three ✓.

**D. Rollups** (`pool-status-history`, `vp-attribution`) — regenerate after A
lands; historical exactness via SPEC-distributions-capture harvest (pcts) +
lock/vote-events derive (absolutes). No scalar fudge on history — fields not
rebuilt exactly get renamed `*_boost_legacy`, never silently reinterpreted.

## 4. Acceptance test (run at deploy)

Timestamp-matched pair: `gauge_infos{gauge:"bluechip",time:"next"}` vs TLA UI
bluechip tab screenshot (≤2 min apart). Pass = every listed pool's
vp_total matches UI display rounding; ghost pools (wstETH-SS, wBTC.osmo-*)
present in gauge_infos but absent from distributions are labeled
`unlisted_no_distribution`, not mixed into listed-pool pct math.
Second check after next epoch flip: distribution fractions × bucket rewards ≈
actual reward deltas per pool.

## 5. Doctrine updates riding this fix

- PROJECT_KNOWLEDGE "Total TLA VP = max bucket ≈ 24M": replace with
  `total_vamp.vp` (fixed+boost, ~28M). Bucket tallies remain per-gauge facts.
- Bucket-tally-vs-total comparisons are only valid **like-for-like by period**
  (a period-N freeze may legally exceed the current total_vamp).
- New metric available free: per-pool + system `slope` (VP decay forecasting)
  and `voting power factor` (vp/fixed) — capture, expose later.
