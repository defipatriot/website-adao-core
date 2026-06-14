# SYSTEM AUDIT & OPERATIONS — the master map (living doc)

**Started 2026-06-14.** The single place that captures: every cron, every data
repo, the dependency/timing order, the cleanup hitlist, and the year-rollover +
token-expiry plan. Goal: a coherent, year-proof platform that's trivial to extend
to a new collection/community. **This doc is the "get organized" deliverable.**

> ⚠️ Some facts only live in the **Render dashboard** (actual schedules) or in
> Camron's head (which old repos are dead). Those are marked **[NEEDS CAMRON]** —
> fill them and this becomes the authoritative operations manual.

---

## 1. CRON INVENTORY (19 crons in `cron-scripts`)

| Cron | Entry file | Lines | Writes → repo | Reads (deps) | Schedule (README) | **[NEEDS CAMRON] real Render schedule** |
|---|---|---|---|---|---|---|
| **network-and-prices** | network-and-prices.js | 917 | network-and-prices-data_2026 | — (CG/Astroport APIs) | Daily | ? — MUST be earliest (foundation) |
| **tla-registry** (chain/) | tla-registry.js | ? | tla-chain-registry | — (global-config bootstrap) | ? | ? — MUST be early (foundation) |
| bribes-history | bribes-history.js | 789 | bribes-data_2026 | network-and-prices | every 4h | ? |
| astroport | astroport-snapshot.js | 972 | astroport-pool-data_2026 | Astroport API | daily | ? |
| backing | backing-snapshot.js | 247 | backing-data_2026 | chain | ? | ? |
| tla-participants | tla-participants.js | 381 | tla-participants-data_2026 | **bribes-data** | daily | ? — must run AFTER bribes |
| nft-inventory | nft-inventory.js | 2207 | nft-inventory-data_2026 | prices + **tla-chain-registry** | :30 hourly | ? — after prices+registry |
| marketplace-stats | marketplace-stats.js | 957 | marketplace-data_2026 | prices | hourly | ? |
| dao-dashboard | dao-dashboard.js | 706 | tla-snapshot-data_2026 | prices + tla-ext | :20 hourly | ? |
| ampcapa | ampcapa-snapshot.js | 301 | ampcapa-data_2026 | tla data | ? | ? |
| tla-snapshot | apr-history-rollup.js | 271 | tla-snapshot-data_2026 | (self) | :40 hourly | ? — ⚠ shares repo w/ dao-dashboard |
| tla-vp-holders | tla-vp-holders.js | 362 | tla-vp-holders-data_2026 | chain | ? | ? |
| tla-locks | tla-locks.js | 408 | tla-locks-data_2026 | chain | daily | ? |
| adao-positions | adao-positions.js | 735 | adao-positions-data_2026 | chain + adao | Weekly⚠ | ? — README says weekly, CODE wants DAILY |
| adao-allies | adao-allies.js | 215 | adao-allies-data_2026 | chain (+engine) | daily | ? — after adao+tla |
| votion-positions | votion-positions.js | 409 | votion-positions-data_2026 | chain | daily | ? |
| votion | votion-snapshot.js | 830 | votion-data_2026 | chain | daily | ? — ⚠ vs votion-positions: overlap? |
| skeletonswap-lp_data | index.js | 1207 | ss-pool-data_2026 | prices | daily | ? — ⚠ UPSTREAM FROZEN, see cleanup |
| fuel | fuel-hourly-snapshot.js | 406 | fuel-data_2026 | chain | hourly | ? |

---

## 2. DEPENDENCY ORDER (the timing logic must respect this)

```
LEVEL 0 (foundation — run FIRST, earliest hours):
   network-and-prices   (prices for everything)
   tla-registry         (catalog/directory for everything)

LEVEL 1 (read foundation):
   bribes-history       ← prices
   astroport, backing   ← chain/APIs

LEVEL 2 (read level 1 + foundation):
   tla-participants     ← bribes-data        ⚠ MUST run after bribes-history
   nft-inventory        ← prices + registry  ⚠ MUST run after both foundations
   marketplace-stats    ← prices
   dao-dashboard        ← prices
   ampcapa              ← tla data

LEVEL 2 (independent chain reads — order flexible, just stagger):
   tla-locks, tla-vp-holders, adao-positions, adao-allies,
   votion-positions, votion, fuel, ss
```

**The rule:** a cron must not run before the data it reads is fresh. Today the
hourly crons (nft :30, dao :20, snapshot :40) are interleaved but it's unclear if
prices/registry refresh *before* them each hour. **[NEEDS CAMRON]** the real
Render times so we can verify no dependent runs ahead of its source.

---

## 2b. TIMING MODEL — what actually needs careful timing (and what doesn't)

Camron will set schedules directly; this is the RECOMMENDED layout. The insight:
**timing only matters where the chain changes on a schedule.** Two classes:

**Continuous / claim-anytime — snapshot whenever, any representative time is fine:**
- LP rewards (paid frequently), rebase/unclaimed (claim anytime), total VP & % of
  VP (people add anytime), APR (dynamic, depends on TLA staked — always moving).

**Epoch-bound — timing MATTERS, capture around the weekly boundary:**
- Vote rewards (paid weekly), votes & reward-to-LP adjustments (weekly), the
  end-of-epoch finalized state, and the start-of-epoch post-distribution state.
- **Votion forward-projection:** show *throughout the week* what Votion is going
  to do to NEXT week's balance (so users can act before the boundary) — needs
  regular capture across the week, not just at the boundary.

### Recommended schedule (staggered; foundation first)
```
00:00  network-and-prices      ← foundation, earliest
00:10  tla-registry            ← foundation (catalog/directory)
00:30  bribes-history          (also every 4h for freshness)
00:40  astroport, backing
01:00  tla-participants        (after bribes)
01:10  nft-inventory           (after prices+registry)
01:20  tla-locks, tla-vp-holders, adao-positions, adao-allies,
       votion-positions        (stagger ~5min apart, all read foundation)
01:50  ampcapa, dao-dashboard, marketplace-stats
+ hourly tiles (nft :30, dao :20, snapshot :40, fuel, marketplace) keep their
  cadence but ensure prices refresh hourly BEFORE them, OR have them tolerate
  slightly stale prices (they already cache).
+ votion (system snapshot) WEEKLY near epoch boundary.
```
Rule: foundation (prices, registry) before everything; bribes before participants;
prices+registry before nft-inventory. Everything else is order-flexible — stagger
~5 min to avoid hammering publicnode (BATCH_CONCURRENCY ≤5).

## 3. EPOCH-BOUNDARY TIMING (the part that genuinely needs design)

Some data must be captured at **end of epoch** (finalized votes, settled bribes,
completed migrations, newly-locked LUNA) and some at **start of epoch** (to show
the *effect* of the epoch turnover). Current crons are daily/hourly — they don't
specifically fire ON the boundary.

**[DESIGN NEEDED]** An epoch-boundary trigger:
- Capture-at-END: a snapshot just before epoch close (final VP, final votes, final
  bribe allocations, lock states). Source: gauge `last_distributions`, escrow.
- Capture-at-START: a snapshot just after open (post-distribution balances,
  rebase applied, new VP) — diffing start-vs-prior-end shows the epoch's effect.
- Epoch length + boundary time come from global-config (epoch timing) — the
  registry already reads this. A boundary cron could read "seconds to next epoch"
  and self-schedule, OR run frequently near the known boundary.

This feeds: epoch-over-epoch growth (portfolio), bribe-vote feedback (vote intel),
per-pool boundary attribution (CRON-FIXES-BRIEF line 48). **Build as part of the
history/accumulation layer.**

**Bribe-batch staleness (important):** PD-style bribes are added in BATCHES (e.g.
4 epochs at once), graded at add-time. Efficiency can shift across those epochs,
so a bribe optimal at epoch 1 may be stale by epoch 4. **Capture each epoch's
state so we can SHOW the gap** between when a batch was set and current efficiency
(a data-driven nudge to re-optimize). Requires per-epoch boundary snapshots.

**Bribe-source attribution → Votion swing (system-health signal):** Votion holds
a LARGE chunk of VP, so when a briber (PD, Solid, Astroport, aDAO, …) adds bribes,
they're effectively steering a big block of voting power. We should decompose the
Votion swing by WHO is pushing it: how much of Votion's vote movement is driven by
PD (centralization-risk signal) vs Solid vs Astroport vs aDAO vs others. This
matters for system health — concentrated bribe influence over a VP-heavy
aggregator is a centralization vector worth surfacing. Data: bribe events keyed by
briber × pool × epoch (bribes-history / backfill) joined to Votion's vote
allocations per epoch (votion / votion-positions). 

**Causality chain (needs backfill + boundary capture):** the product goal is to
show bribe → vote → liquidity-traffic causality: did bribes move votes, did the
moved rewards move LP deposits, where did providers chase yield. And a **bribe-
contributor leaderboard** (Astroport, Solid, etc. add frequently) showcasing who
actually supports TLA ($5,000 vs $2). Both need historical bribe/vote/deposit
events — backfillable via tx_search (SPEC-tla-history-backfill.md).

---

## 4. CAPTURE CADENCE MAP (daily / epoch / monthly / yearly)

What the portfolio/grading layers need vs what we capture:
| Cadence | Purpose | Have it? |
|---|---|---|
| **Hourly** | live tiles, freshness | ✅ prices, nft, dao, snapshot, marketplace, fuel |
| **Daily** | day-over-day growth, P&L | ✅ adao-positions, tla-locks, votion-positions, allies, participants (daily archives just added) |
| **Epoch** | epoch growth, vote/bribe settlement | 🔶 weekly/epoch archives exist (adao-positions); no dedicated boundary capture |
| **Monthly** | monthly growth | ⏳ derive from daily archives (accumulating) |
| **Yearly** | yearly growth | ⏳ derive from daily (long runway) |

Monthly/yearly are DERIVED from daily — no separate cron needed, just retain daily
and roll up. The gap is the **epoch boundary** capture (section 3).

---

## 4b. 🧹 CLEANUP MANDATE (Camron: "anything not in the new system goes away")

Rule: any live page reading OLD/dead data → convert to the new system (find the
data in the new crons; if missing, add it), or remove that feature. Old data that
"just shows some history" but comes from dead snapshots = delete; we want what
works, built forward from the new system.

### Live-page audit — what each main page still reads OLD (concrete, verified)
Most "old references" are comments/fallbacks, NOT live fetches. The REAL live
old-source reads to convert:

| Page | Old live read (CONVERT) | New-system source to use instead |
|---|---|---|
| **index.html** | `tla-ext_json_storage/tla-ext-epoch-N-end.json`, `tla-data-epoch-N-end.json`, `adao-snapshot_N_end.json` (epoch fallbacks) | `adao-positions-data_2026/current.json` (already partly reads it) + `tla-snapshot-data_2026/dao-dashboard.json`. Already reads network-and-prices + nft-inventory ✅ |
| **dao_tla_deposits.html** | `${TLA_URL}/tla-data-epoch-N-end.json` | needs the TLA deposits data from the new split crons (tla-snapshot/dao-dashboard) — **the snapshot-modal migration noted in CHANGES_PENDING** |
| **dao_treasury.html** | (mostly converted — reads dao-dashboard.json ✅) `tla-data-epoch` only in fallback | confirm fallback removable; use `aDAOLive.getDaoTreasury()` |
| **tla-stats.html** | `ssWeeklyAvgBaseUrl` (SkeletonSwap — FROZEN), epoch-N json for astroport/votion | astroport-pool-data_2026, votion-data_2026; **drop SS or label unverified** |
| fuel-tool, ally, dao, dao_governance_tool | references are non-fetch (comments/old fallbacks) | verify + strip references; low priority |

**The standalone OLD tools** (`tla_tool.html`, `tla-tool_ext.html`) are the
deprecated manual-capture flow (replaced by crons per PROJECT_KNOWLEDGE). →
**retire these pages** unless still linked/needed.

### Dead data repos to KILL (Camron confirms, then delete)
Everything not written by one of the 19 current crons:
- `tla_json_storage`, `tla-ext_json_storage` — old admin-tool storage, stopped
  2026-05-17. Convert index/tla-stats readers off, then delete.
- `adao_json_storage` — old registry, absorbed into tla-chain-registry. Delete after confirm.
- deving.zone-derived data — known-broken, migrated away.
- any pre-`_2026` data repo, any old NFT captures (pre chain-of-truth).
- ss-pool-data_2026 — upstream FROZEN; pause cron or keep best-effort but LABEL
  unverified + never feed grading/charts.

**Principle:** wrong historical data is worse than none — it silently corrupts
charts. Once a reader is converted and a repo confirmed dead, DELETE it.

## 5. (was hitlist — folded into 4b above) Original suspects

**[NEEDS CAMRON to confirm each before deletion — destructive.]**

| Suspect | Why | Action |
|---|---|---|
| `tla_json_storage`, `tla-ext_json_storage` | old admin-tool storage, STOPPED publishing 2026-05-17, dao-dashboard/index still partly read it | Migrate readers off, then archive/delete |
| `adao_json_storage` | old registry source, absorbed into tla-chain-registry | Confirm fully absorbed, then archive |
| ss-pool-data_2026 (skeletonswap) | upstream FROZEN ~30d, data unreliable, "don't use for scoring" | Keep capturing best-effort OR pause cron; LABEL unverified everywhere; never feed grading |
| ~~`votion` vs `votion-positions`~~ ✅RESOLVED | Different jobs: votion-snapshot = WEEKLY system totals (ratios/total_vp/aggregate lockups); votion-positions = DAILY per-user holdings. **Complementary, keep both.** | none — not duplicates |
| ~~`tla-snapshot` vs `dao-dashboard`~~ ✅RESOLVED | Share the repo but write DIFFERENT files (apr-history.json vs dao-dashboard.json/tla-snapshot.json). No conflict; just two crons → one repo (tidy later). | none — works |
| any `*-data` repo WITHOUT `_2026` | pre-2026 captures | list + archive |
| deving.zone-derived data | known-broken feed, migrated away | confirm no cron still writes it |
| old wrong NFT captures (pre chain-of-truth) | superseded by nft-inventory v2 | delete stale files |

**Principle:** wrong historical data is worse than no data — it silently corrupts
charts. Delete aggressively once confirmed dead, but CONFIRM first (some "old"
repos may still have a live reader).

---

## 6. 📅 YEAR-ROLLOVER PLAN (don't panic in December)

**Problem:** repos are `*-data_2026`; tokens expire end of 2026. Two cliffs at once.

**[DECISION NEEDED — recommend doing in Nov 2026]:**
1. **Repo rollover strategy.** Options:
   - (a) New `*-data_2027` repos per cron (matches current pattern; cron's
     `GITHUB_REPO` env flips 2026→2027). Clean break, 2026 frozen as history.
   - (b) Switch crons to **year-folder inside one repo** (like tla-chain-registry
     already does: `2026/`, then add `2027/`). **This is the pattern Camron
     wishes everything used.** Could migrate crons to write `data/2026/...` now so
     2027 is just a folder, no new repo. ← RECOMMENDED for any cron we touch anyway.
   - Cross-year charts need readers to span both — easier with (b).
2. **Token expiry.** All commit tokens expire end-2026. Before then: generate new
   tokens, update every Render cron's `GITHUB_TOKEN` env. **Make a checklist of
   all 19 crons** so none is missed (a missed token = silent capture failure).
3. **Heartbeat watch.** Every cron writes heartbeat.json — a simple monitor that
   alerts if any heartbeat goes stale would catch a missed-token or dead-cron
   immediately. Worth building before the rollover.

**Action now (cheap, future-proofs):** for any cron we edit going forward, switch
its output path to a year-folder (`data/2026/...`) so 2027 is a one-line change,
not a panic. Don't mass-migrate working crons just for this — do it opportunistically.

---

## 7. 🔮 THE END-STATE ARCHITECTURE (where this is heading)

Three capture modes, layered:
1. **Backfill** (one-time): events to genesis via tx_search — votes, locks,
   deposits, Votion, claims. Recovers past behavior (not past valuations). Spec:
   SPEC-tla-history-backfill.md.
2. **Live additions** (scheduled crons): the current model — periodic snapshots.
3. **Block-by-block watcher** (future): tail the chain, catch events as they
   happen, update live. The real-time layer. Biggest build; needs an indexer or a
   block-streaming service (not a cron). Design later.

Plus the **extensibility goal**: when a new collection/community asks "do for us
what you did," adding them should be: (1) one `ALLIES` entry / one collection
config, (2) curated registry entries, (3) they appear in trackers/grading. The
engine reuse (capture-engine.js, ally-capture.js) already makes member capture
near-free; the registry + portfolio-assembler + grading need the same
"parameterize by community" design so onboarding is config, not code.

---

## 8. IMMEDIATE NEXT ACTIONS (from this audit)
- [ ] **[CAMRON]** Fill the real Render schedules (section 1) — the one thing only you can see.
- [ ] **[CAMRON]** Confirm the cleanup hitlist (section 5) — which repos are truly dead.
- [ ] **[CAMRON]** Clarify votion vs votion-positions and the shared tla-snapshot repo.
- [ ] Verify dependency order holds once schedules are known (section 2).
- [ ] Design the epoch-boundary capture (section 3).
- [ ] Build a heartbeat-staleness monitor (cheap, guards token-expiry + dead crons).
- [ ] Adopt year-folder output for any cron we touch (section 6).

## 9. 📦 FULL REPO CLASSIFICATION (all 33 defipatriot repos, verified 2026-06-14)

**🟢 LIVE DATA (18 — current crons writing, pushed 06-13/14):**
network-and-prices-data_2026, tla-snapshot-data_2026, nft-inventory-data_2026,
marketplace-data_2026, fuel-data_2026, adao-positions-data_2026,
tla-participants-data_2026, adao-allies-data_2026, tla-locks-data_2026,
votion-positions-data_2026, tla-chain-registry, bribes-data_2026, ss-pool-data_2026,
ampcapa-data_2026, backing-data_2026, astroport-pool-data_2026, votion-data_2026,
tla-vp-holders-data_2026.

**🟢 LIVE INFRA (5):** cron-scripts, website-adao-core, aDAO-links-site,
nft-metadata (rarity JSONs), aDAO-Image-Files. Plus **token_logo — KEEP**
(index.html uses it for logos).

**🔴 DEAD — DELETE (0 live references, verified):**
- Safe to delete now: `astroport_json_storage` (2025-11), `archive-storage`,
  `nft-tracker` (2025-10), `transaction-tracker`, `adao_nft-tx_2025`,
  `aDAO-Image-Planets-Empty`.
- Delete AFTER converting index.html fallback reads off them: `tla_json_storage`,
  `tla-ext_json_storage` (both stopped 2026-05-17), `adao_json_storage` (absorbed
  into tla-chain-registry).

## 10. ✅ DATA-CORRECTNESS AUDIT (Camron's "is the captured data right?" — verified 2026-06-14)

**Verdict: the accumulated data is largely SOUND. No zeroing needed.**
- **adao-positions**: 28 daily files (05-18→06-14), continuous, no gaps, status
  ok, 0 member errors. Per-member VP is live + detailed (full per-bucket vote
  breakdowns). The near-constant named-total VP is CORRECT (members are mostly
  auto-max-locked → permanent VP doesn't decay), not a freeze.
- **tla-chain-registry**: 9 dailies, 75 pools consistent, epoch advancing. Sound.
- **tla-locks**: 431 locks, 0 errors, enumeration complete. Sound.
- **bribes-data**: 247 proposals / 178 bribes / 57 epochs. Sound.

**⚠ One schema-evolution hazard (not corrupt data, but charts must handle it):**
adao-positions daily schema changed mid-stream — the member model widened from 46
named (pre ~06-13) to 156 all-members. `totals_named`/`retention`/`note` only
exist in recent files. **Charts spanning the change must use the stable `totals.*`
fields** (present throughout), and treat pre/post-06-13 member-count as a known
model change, not a data jump. Early dailies = named-46 view; current.json = 156.

**Recommendation:** do NOT zero out — the history is usable. Just have the chart
layer key on stable fields and annotate the 06-13 member-model widening. If any
single early file looks off when we build charts, drop that one file rather than
the whole series.

## 11. 🔍 GAP ANALYSIS — what's still missing (thorough pass 2026-06-14)

### Needs Camron (can't see from here)
1. **Render schedules** — actual run times (we gave a recommended layout; need
   current to compare). The single highest-value thing to hand over.
2. **Confirm dead-repo deletions** (section 9) + the index.html fallback conversion.

### Data we WANT but don't yet capture (from the product vision)
4. **Epoch-boundary snapshot** — no cron fires at end/start of epoch. Blocks:
   epoch-over-epoch growth, vote/bribe settlement view, bribe-batch staleness gap.
5. **Per-address realized-rewards ledger** — needed for realized-vs-advertised APR.
   Record rewards claimed/accrued forward (claim events also backfillable).
6. **Bribe-source → Votion swing attribution** (centralization-health signal):
   bribe events keyed by briber × pool × epoch, joined to Votion vote allocations.
   Shows how much of Votion's VP swing each briber (PD/Solid/Astroport/aDAO) drives.
7. **Bribe contributor leaderboard** ($-contributed by Astroport/Solid/etc).
8. **Causality: bribe→vote→liquidity-traffic** — event backfill + boundary capture.
9. **Simulated-exit slippage** per pool (LP grading dim 3) — pool reserves + curve math.
10. **Net-APR decomposed** (fee/emissions/bribe, after take) — partially present in
    astroport-snapshot; confirm it's the trustworthy net figure per CRON-FIXES-BRIEF 2.10.

### Spec'd, not built
11. History/event backfill · 13. Portfolio-assembler · 14. LP grading engine ·
15. Heartbeat-staleness monitor (token-expiry safety net) · 16. UI pages.
12. Block-by-block live watcher (future; needs indexer, not a cron).

### Process / ops
17. **No central schedule registry** — schedules live only in Render (fragile).
    Recommend a `cron-schedule.md` in cron-scripts mirroring Render, as the
    single source of truth + the token-rollover checklist.
18. Year-rollover + token-expiry prep (Nov 2026).
19. ss-pool-data capturing frozen upstream — decide pause vs label-unverified.

### Verified SOUND (no action)
- Data correctness (section 10), repo inventory (section 9), votion vs
  votion-positions + tla-snapshot vs dao-dashboard (section 5, not conflicts).
