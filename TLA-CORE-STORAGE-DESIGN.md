# tla-core — Unified Repo Storage Design (canonical)

**Settled 2026-06-24.** The storage convention for the new unified `tla-core`
repo that replaces the one-repo-per-cron `*-data_2026` sprawl. `fuel/` is the live
pilot module; `tla-flows`' `flows/` module is the second. This doc owns the
**layout convention** — capture-layer details (what tla-flows captures, the
parser, costs) live in PROJECT_KNOWLEDGE's "TLA LP-flow event capture" subsection
and are not repeated here.

> **Cron-deploy decision (the one thing the `tla-flows` Render deploy waited on):**
> the layout `tla-flows` already writes — `flows/events/{heartbeat,index,cursor}.json`
> + `flows/events/2026/MM/DD.jsonl` — **is correct and final. No output-path change
> needed.** Deploy as-is. Rationale below.

---

## 1. Target structure: module → product → files

```
tla-core/
├── README.md
├── {module}/                 ← one per cron/domain (fuel, flows, positions, …)
│   └── {product}/            ← the data SHAPE; a module may have >1 product
│       ├── heartbeat.json    ← liveness + last-run stats (ALWAYS)
│       ├── index.json        ← manifest: what exists, latest pointer (ALWAYS)
│       ├── cursor.json       ← resume point (EVENT products only)
│       ├── current.json      ← latest snapshot (SNAPSHOT products only)
│       ├── daily/ hourly/    ← rolling recent (SNAPSHOT products)
│       └── {YYYY}/           ← year partition (history + rollups)
│           └── {MM}/{DD}.jsonl   (EVENT)  or  {epoch-avg,monthly-avg}/{YYYY}.csv (SNAPSHOT)
```

**Three levels, always:**
- **module** = the cron / domain. Mirrors the `cron-scripts/{module}/` folder name.
  Examples: `fuel`, `flows`, (future) `positions`, `history`, `prices`, `nft`.
- **product** = the data shape a module emits. Most modules have one; some have
  several. Two product TYPES exist (below). Examples: `snapshots`, `events`.
- **files** = the per-product conventions (§3).

This is why the path is `fuel/snapshots/…` and `flows/events/…` — same skeleton,
different product type.

---

## 2. The two product types

### SNAPSHOT product (state at a point in time) — e.g. `fuel/snapshots/`
Captures the *current state* on a schedule. Diff-able across time. Files:
`heartbeat.json`, `index.json`, `current.json`, `daily/`, `hourly/`, and yearly
rollups (`{YYYY}/epoch-avg/{YYYY}.csv`, `{YYYY}/monthly-avg/{YYYY}.csv`).

### EVENT product (append-only stream) — e.g. `flows/events/`
Captures *discrete events as they happen*, resumable from a cursor. Append-only,
never rewritten. Files: `heartbeat.json`, `index.json`, `cursor.json`, and
date-partitioned `{YYYY}/{MM}/{DD}.jsonl` (one JSON event per line).

A single module can have BOTH (e.g. a future module with `events/` for the raw
stream and `snapshots/` for derived rollups) — they're sibling product folders.

---

## 3. Per-product file conventions

### `heartbeat.json` (every product) — schema v1
Same contract as the legacy crons, so the System Health monitor reads it
unchanged. The monitor watches `tla-core/{module}/{product}/heartbeat.json`.
```json
{
  "schemaVersion": 1,
  "cron": "flows",                       // module name
  "capturedAt": "2026-06-24T17:50:18.079Z",
  "capturedAtUnix": 1782323418079,
  "runId": "flows-20260624175018",
  "runMode": "events",                   // "hourly" | "daily" | "events" | "backfill"
  "status": "ok",                        // "ok" | "partial" | "error"
  "stats": { /* product-specific counters */ },
  "next_expected_run_at": "2026-06-24T18:05:18.079Z"
}
```
Event products add to `stats`: `events_appended`, `cursor_height`, `head_height`,
`contracts_scanned`. Snapshot products add their domain metrics (as fuel does:
`price_usd`, `tvl_usd`, …).

### `index.json` (every product) — the manifest
The "what exists here" pointer so consumers don't have to list the tree.
- **Snapshot** (as fuel): `{ latest, daily:{weekday→date}, history:[ {date, …rollup} ] }`.
- **Event:** `{ latest_date, first_date, years:[2026], days_present_count,
  total_events, by_type:{deposit,withdraw,claim}, partitions:{ "2026":["06/24", …] } }`.

### `cursor.json` (EVENT products only) — the resume point
Makes completeness come from a saved position, not from being always-on.
```json
{
  "schemaVersion": 1,
  "perContract": { "<contract_addr>": { "last_height": 12345678, "last_txhash": "…" } },
  "head_height_at_last_run": 12345999,
  "updatedAt": "2026-06-24T17:50:18.079Z"
}
```
**Write order is law:** append events → write `index.json` → **advance `cursor.json`
LAST** → `heartbeat.json`. A crash re-reads the unmoved window (idempotent via
txhash dedupe). Any query error returns WITHOUT advancing the cursor (fail-safe).

### Year / month partition pattern
- **Event:** `{YYYY}/{MM}/{DD}.jsonl` — one file per day, append-only, one event
  per line. Cheap appends, cheap date-range reads, naturally year-rolls.
- **Snapshot:** `{YYYY}/{rollup-name}/{YYYY}.csv` for long-horizon averages
  (epoch-avg, monthly-avg), plus `daily/` + `hourly/` for the rolling recent
  window outside the year folder.
- **Year rollover is a new folder, never a new repo** — this is the whole point of
  tla-core. 2027 = add `{module}/{product}/2027/`. No env flip, no new Render repo,
  no token juggling. (Contrast the `*-data_2026` repos, which need a 2027 twin.)

---

## 4. Module skeleton standard (how to add a module)
A new module under tla-core is: `tla-core/{module}/{product}/` containing, at
minimum, `heartbeat.json` + `index.json`, plus the product-type files from §3. A
new cron writes there by pointing its output base at a `tla-core` checkout and
committing exactly as the legacy crons commit to their data repos (the commit
helper is unchanged — only the target path differs). `fuel/` is the reference
SNAPSHOT module; `flows/` is the reference EVENT module. Copy whichever matches.

---

## 5. Migration checklist — which `*-data_2026` repos fold in, and order

**Principle:** migrate opportunistically and by value; never mass-move working
crons just to tidy. A cron moves to tla-core when we're editing it anyway, OR when
the year-folder benefit is worth it. Run new + old in parallel, prove, then retire
the old repo (freeze read-only first, delete only after consumers are repointed).

**Order (low-risk → high-value):**
1. **fuel** — ✅ DONE (the pilot; proves the snapshot pattern, live + hourly).
2. **flows** — ⏳ tla-flows, code ready, deploy pending (proves the event pattern).
   This is the first *new* capture on tla-core — no legacy repo to retire.
3. **history** (`tla-history`, votes+lock backfill) — next natural event module;
   pairs with flows. New capture, no legacy retire.
4. **Then, by value, fold existing snapshot crons** as they're touched:
   `ampcapa`, `backing` (already small, snapshot-shaped, recently edited) → easy
   early candidates. Then the bigger ones (`astroport`, `marketplace`,
   `tla-snapshot`) only when worth it.
5. **Leave alone for now:** `tla-chain-registry` (already uses a year-folder
   pattern internally and works), `nft-inventory` (large, v2 path stable),
   `network-and-prices` (high-traffic, leave until a reason to touch).

**Per-migration steps:** point the cron's output base at a tla-core checkout
`{module}/{product}/` → run in parallel with the legacy repo → verify the System
Health monitor reads the new heartbeat path → repoint any site consumers → freeze
the legacy `*-data_2026` repo read-only → delete after a cooling period.

**Monitor note:** when a module lands in tla-core, add/repoint its entry in
`system-health.js` `MONITORED` to `tla-core` raw path
`…/tla-core/main/{module}/{product}/heartbeat.json`.

---

## 6. Why this layout (the rationale that settled the deploy)
- **One repo, many modules** kills the 18-repo sprawl and the per-repo token/
  year-rollover tax. 2027 is a folder, not 18 new repos.
- **module→product→files** cleanly separates "what domain" from "what shape," so a
  module can grow a second product without a new repo or a naming fight.
- **fuel and flows already fit it** with zero contortion — fuel as snapshot, flows
  as event — which is the proof the convention is right. **So `tla-flows` ships its
  current paths unchanged.**
