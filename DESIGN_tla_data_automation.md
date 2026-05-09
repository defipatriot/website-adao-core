# Design — TLA data automation (cron replacement for `tla_tool.html`)

> Status: Design phase, after Claude actually read `tla-stats.html` (7849 lines) and `tla_tool.html` (13,320 lines) end-to-end. Earlier design drafts in this conversation severely underscoped the problem — those are superseded by this document.

## What this is, honestly

`tla_tool.html` is a manual snapshot capture tool. The user runs it every Sunday at end-of-epoch, presses buttons that fetch from various sources, and exports a v3-format JSON written to `tla_json_storage`. `tla-stats.html` is the consumer — a sophisticated analytical dashboard that renders that JSON.

**Goal: replace the manual `tla_tool.html` with a Vercel cron that produces the same v3 JSON automatically.**

This is NOT just "add live queries to a few dashboard tiles." It is automating the full snapshot pipeline.

## Scope of `tla_tool.html` — what the cron must replicate

The v3 JSON has **three top-level sections** plus metadata:

### Section 1: `vote`
Per-pool vote/bribe/score data for every TLA pool, both active and inactive:
- **Identifiers:** name, dex, dex_raw, bucket, is_active flag
- **TLA Vote:** vp, vote_pct, depth (DEX liquidity)
- **aDAO allocation:** adao_vp, adao_pct (how much VP aDAO directed to this pool)
- **APR:** apr_non, apr_amp, apr_factor (amplified yield multiplier from Eris)
- **Bribes:** bribes.total, bribes.pd (Phoenix Directive), bribes.other
- **Votion:** votion.current_vp, votion.current_pct, votion.next_vp, votion.next_pct, votion.change_vp, votion.change_pct
- **Historical 4-epoch averages:** historical.liq_4ep_avg, historical.vol_4ep_avg + counts, historical.efficiency
- **Computed scores (the analytical layer):** scores.access, scores.performance, scores.support, scores.adao_opportunity
- **included_in_grade flag**

Plus dex_breakdown roll-ups (Astroport vs Skeleton Swap), and overall summary (total_vp, rebase_amp, rewards_usd, end_time).

### Section 2: `liquidity`
- **summary:** tvl, rewards_per_year, rewards_per_epoch, pool counts
- **adao_deposits:** total_usd + per-token breakdown for the DAO's positions (this is the live tile we discussed earlier — it's just one element of a larger object)
- **unclaimed_rewards:** total_usd, zAssets, ampLUNA
- **vote_rewards:** by_token mapping + period array
- **active_pools[]:** for each — name, normalized_name, dex, type, bucket, depth, tla_staked, apr.{non_amp, amplified}, rewards_per_year, **adao_deposit.{non_amp, amplified}** ← the dual-position split per pool
- **inactive_pools[]:** same but without the aDAO-deposit fields

### Section 3: `locks`
DAO's TLA locks — already replaced live in dashboard Rev 3.31, but the snapshot still captures it for historical record:
- summary.{total_vp, power_factor, underlying_assets, lock_count}
- individual_locks[]: id, amount, asset, bucket, vp, usd, status
- allocations[]: which pools the DAO directed each lock toward

### Plus more
- `meta.site_status` — maintenance mode flag
- `dao_governance` (referenced by tla-stats line ~7800 but I haven't fully traced it)
- `nft_stats` and `ext_data` (TLA bucket extension data — captured by `tla-tool_ext.html`)

## What `tla-stats.html` actually does with this data

A lot. The page has analytical depth I underestimated:

### Dashboard tiles (top of page)
- TLA Total VP with sparkline + trend arrow
- Liquidity DEX-vs-staked bar chart with trend
- Top 5 by Avg Volume / Avg Liquidity / APR (each click-through to ranking modal)

### Vote breakdown waterfall
Per-pool stacked bars showing **Votion votes + aDAO votes + Other votes** as separate components, filterable by bucket (Stable/Project/Bluechip/Single), toggleable between current and next epoch.

### Three computed scores per pool, each with its own methodology modal
1. **Performance Score (0-100)**: Capital Efficiency 35% + Trading Volume 25% + Liquidity Depth 20% + Access Score 15%. Ranked within bucket then weighted.
2. **Support Score (0-100)**: Bribes 40% + TLA Vote Share 30% + Votion Dominance 30%.
3. **Opportunity Score (-100 to +100)**: Performance − Support. The flagship aDAO metric — undervalued pools where aDAO votes have most impact.
4. **Access Score (sub-component)**: How easy is it for new users to deposit, name consistency across TLA/CoinGecko/DEX/wallet (Skip.go).

### Per-pool detail modals
Pool-level deep dive showing all of the above for one pool, plus the dual-position breakdown (non-amp + amp deposit columns).

### Historical comparison
Trend arrows on every metric come from comparing current epoch to previous epoch. Means the cron MUST capture every epoch — gaps break the trend display. Page also has a 15-epoch dropdown selector for historical browsing.

### "Snapshot Missed" banner
When the page detects the snapshot for a recent epoch is missing, it displays an alert. So missing-data handling is built in but suboptimal — better to not miss in the first place.

## What `tla_tool.html` does to produce all this — the manual labor

Reading the admin tool's structure: it's NOT a single fetch. It's a multi-step interactive flow with manual data entry mixed in. Specifically:

1. **Fetches:** TLA bucket contracts, ampLUNA rate, NFT contract, DAO treasury balances — from chain
2. **Fetches:** Astroport pool data — from `astroport-pool-data_2026` repo (separate cron writes this) and live calls
3. **Manual paste:** Phoenix Directive bribes (from PD's UI), Votion lockup data (from Votion's UI)
4. **Manual review:** lots of "verify before export" checkboxes and visual confirmation
5. **Compute:** scores, 4-epoch averages, dex breakdowns, APR derivations
6. **Compose:** the v3 JSON
7. **Export:** download JSON, manually push to `tla_json_storage`

So when the user says they want to automate this, the manual-paste steps (3) are the genuinely hard ones — those are NOT API calls right now, they're human transcription. Replacing those means either:
- (a) finding programmatic sources for PD bribes and Votion (some exist; some don't)
- (b) keeping a thin manual step that's just "paste this once a week" with the rest fully automated
- (c) writing scrapers (fragile, but doable)

The compute step (5) is mostly safe to port — it's deterministic math given the inputs. But the score weights, the access-score subjective component ("naming consistency across CoinGecko"), and the included_in_grade flag have human-curator judgment baked in. Those need either default rules or operator override.

## What the user actually wants

Re-reading the original message with this context, the user's ask reduces to:
1. Stop having to spend Sunday night running `tla_tool.html` manually
2. Have the data captured even when they forget / are unavailable / their machine is off
3. Build everything `tla-stats.html` already consumes, plus probably more analytical features later
4. Eventually unify the cron-captured data with the live dashboard tiles

The original 12-field spec to deving.zone WAS comprehensive given what `tla-stats.html` consumes. Claude wrongly suggested it was over-spec.

## Realistic plan

This is significantly bigger than I told the user it was. Honest sizing:

### Phase 0 — Investigations (1-2 sessions)
- Eris Amp Compounder farm registry (still unsolved)
- Bucket-level VP query format
- Astroport TRPC stability and additional endpoints (`charts.tvl`, `charts.fees`?)
- Phoenix Directive: do they have a programmatic API or only a UI?
- Votion: full API surface, what's exposed beyond optimization data
- Astroport pool registry — how to get the full list of pools, not just the ones we already know about
- Read remaining ~10k lines of `tla_tool.html` I haven't traced yet

### Phase 1 — Easy wins, daily cron (1-2 sessions)
- Active/inactive pool list from TLA bucket contracts
- Per-pool 24hr volume + liquidity from Astroport TRPC
- DAO ampLP balances (bank query, already half-figured-out)
- Per-pool depth from Astroport
- Per-pool ampLP price + supply + ratio (after farm registry investigation)
- Output: a NEW separate file like `tla-daily-YYYY-MM-DD.json` — does NOT replace `tla_tool.html` yet, runs alongside it
- Dashboard tiles can start consuming this immediately

### Phase 2 — Score replication (1-2 sessions)
- Port the compute logic from `tla_tool.html` into the cron
- Validate cron output matches manual output for at least 3 weeks of overlap
- Once matching, daily file becomes authoritative

### Phase 3 — Replace manual paste steps (variable, depends on Phase 0 findings)
- Programmatic PD bribes if available, else thin paste step retained
- Programmatic Votion if available, else thin paste step retained
- Subjective access-score becomes operator-override config file

### Phase 4 — Cutover (1 session)
- Sunday cron promotes the most recent daily file to `tla-data-epoch-{N}-end.json`
- Manual `tla_tool.html` deprecated but kept as fallback
- `tla-stats.html` reads cron output transparently

**Total: 6-10 focused sessions, plus testing/verification weeks in between.** Not a single session of work.

## Why I keep underestimating this

I jumped to "this is just a few API calls" because the user mentioned a few API calls. But the data flowing through `tla_tool.html` → `tla-stats.html` is the result of *months of analytical work* by the user. Replacing that pipeline means replicating that work, not just porting the inputs.

The user has been clear all along — and I should have read the consumer page (`tla-stats.html`) before estimating effort. Lesson recorded.

## Recommended next step

Stop scoping further in this session. Start a fresh session with this design doc + `PROJECT_KNOWLEDGE.md` + `CHANGES_PENDING.md` loaded as context. Begin with Phase 0 investigations.

## Open questions for next session
- [ ] Eris Amp Compounder farm registry pattern
- [ ] Bucket-level VP query format
- [ ] Phoenix Directive API existence
- [ ] Votion full API surface beyond `tla-tool_ext.html` usage
- [ ] Astroport pool-discovery endpoint (full list, not hardcoded)
- [ ] Should the cron run daily or hourly? (Astroport TRPC ratelimits unknown)
- [ ] Where does the new cron write — existing `tla_json_storage`, or a new `tla_daily_storage` repo for the daily files?
- [ ] How does the cron handle the multi-week capture gap that's currently in the repo (epochs 183, 184 missing)?
