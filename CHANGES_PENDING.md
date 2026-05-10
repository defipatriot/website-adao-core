# Changes Pending — aDAO website

> Rolling list of identified work for upcoming sessions. See PROJECT_KNOWLEDGE.md "Tracking responsibilities" for what goes here vs. there.
> Older completed items have been pruned — they live in changelog files (`index-log.md` etc.) instead.

---

## 🛠 Active / next round

### 🔥 NEXT SESSION — Begin full TLA snapshot cron automation
**See `DESIGN_tla_full_cron_automation.md` for the full spec.** 

**Goal:** retire `tla_tool.html` + `tla-tool_ext.html` as manual capture tools. Replace with a Vercel cron that produces equivalent JSON snapshots daily. Existing tools kept as fallback.

The design doc inventories every field in the snapshot file, classifies each by automation status (🟢 auto already / 🟡 chain query known / 🟠 flaky API / 🔴 still manual / ⚫ unavoidably manual), and lays out 6 phases of work. Bottom line: full automation is feasible — no field requires unavoidable human input.

**Phase 1 next session:** Vote tab + LP registry on-chain fetcher. 4 `gauge_infos` queries against `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj` + a pool-id-to-name resolver. Browser tool only (cron migration is Phase 5). HAR data already captured in our context — start of next session: read the design doc and begin implementing.

Subsequent phases: resolver cache + multi-DEX → PD bribes from chain → xASTRO/LST APYs → cron migration → 3-week verification soak.

### Resilience prereqs (per index-log Rev 3.27 / 3.29 follow-ups)
- [ ] Per-fetch try/catch isolation in `index.html`'s `fetchLiveOnChainData` so one failed treasury balance fetch doesn't poison every downstream calculation
- [ ] Fallback LCD endpoint on 5xx — both `terra.publicnode.com` and `terra-lcd.publicnode.com` returned 500s on May 8, 2026 and the cascade null-safety added in 3.27 was the only thing preventing total dashboard failure
- [ ] Apply the same parallel epoch-fallback pattern from `fetchTlaFromGitHub` to `fetchTlaData` (TLA Stats page) — works correctly today but unnecessarily slow on stale fallback
- [ ] Per-fetch timeout in `fetchLiveOnChainData` for the slow Terra LCD calls — 14.67s load times mostly come from there

### Admin-tool follow-ups (after May 9 2026 admin-tool stabilization pass)
The May 9 fixes shipped in `tla-tool_ext.html`:
- Astroport per-pool error tracking + red-banner status + diagnostic dialog
- Votion per-lockup error tracking + diagnostic dialog (root cause confirmed: Eris API HTTP 500 from origin)
- Per-attempt 8-second timeouts on both CORS proxy chains
- FUEL price now auto-fetches from on-chain LUNA-FUEL pool reserves (Terra LCD `{pool: {}}` query, same pattern as `fuel-tool.html`). Source becomes `'lcd-pool-derived'` instead of `'manual'`. Falls through to manual entry on chain failure.
- Staking APR + PD bribes + staging/historical/current GitHub raw fetches now cache-busted (`?t=Date.now()` + `cache: 'no-store'`) — fixes the "still showing 6-day-old data" issue
- PD bribes table now has pencil-edit icons + × delete + "+ Add bribe to gauge" buttons. Edit modal matches project modal pattern (X / ESC / backdrop close).
- PD bribes paste flow changed from confirm() to 3-way Replace / Merge / Cancel prompt. Merge sums USD+LUNA for matching `(gauge, pool)` and appends new ones, so launch-incentive bribes layer on top of routine ones cleanly. Bribes that were merged or hand-added show a small `+merged` / `+added` badge in the table.
- **Astroport D90 export bug fixed** — added `selectBestAstroRange()` helper. Was the cause of the "all zeros in export despite log showing $24K, $713K, etc." problem. Astroport's TRPC dropped `D90` as a valid `dateRange` enum value; D7+D30 still work. Helper falls through to whichever range actually has data. Five call sites updated. Astroport error truncation increased from 80→200 chars so future schema changes show their full error messages.
- **Vote-tab parser rewritten to be DEX-agnostic and token-agnostic.** Was hardcoded to `{Astroport, Skeleton Swap, WhiteWhale}` DEXes and `[ampCAPA, xASTRO]` single-sided tokens. The `wBTC.creda.a` pool on the new "Creda" DEX was silently dropped from every registry parse — and that silent deletion was the upstream cause of the user's earlier "wBTC.creda.a never recognized in asset metadata" symptom. New parser detects entries by shape (a name line followed within 6 lines by a `VP <amount> <pct>%` line) rather than by name lookup. Adapts automatically to new DEXes and tokens going forward.
- **Votion fetch resilience.** The previous "always fails" behavior was actually intermittent 500s — verified via HAR capture showing the same backend URL returning 200 when the app loaded. New behavior: retry with exponential backoff (3 attempts, 0s/2s/4s), per-run failed-proxy memory so subsequent lockups skip already-failed proxies, 500ms stagger between lockups to avoid rate limit, smart proxy ordering (cached working proxy first, untested next, blacklisted last-resort), inline per-lockup progress display. Tested against the user's failure pattern: `direct=500, allorigins=timeout, corsproxy=403` on attempt 1 → 2s backoff → `direct=500, allorigins=200` on attempt 2. Works.
- **Side-fix:** null-safed `$('step-export').classList.add()` in `downloadExtJson` so the post-download status update doesn't throw when the element doesn't exist in the current layout.

Still open:
- [ ] **Fix `epoch: "unknown"` in workflow exports.** May 9 2026 Astroport + Skeleton workflow files both have `meta.epoch: "unknown"` and Skeleton pulled epochs 168/169 (old hardcoded fallbacks?) instead of current. Find where the export-builder reads the epoch and ensure it uses `store.liveEpochInfo.currentEpoch`.
- [ ] **Quiet "Pool not found" errors for deprecated duplicates.** Astroport correctly returns `Pool not found` for deprecated pool addresses — the dedup logic flags them. These currently count toward `failed` in the new error reporting and clutter the diagnostic dialog. Better: detect deprecated addresses BEFORE fetching, OR detect "Pool not found" specifically and categorize as `expected-deprecated` rather than failure.
- [ ] **Per-fetch isolation in `tla_tool.html`** (the main tool) — same pattern as the resilience items above. The main tool consumes the ext file and silently rolls failed pools into zeros across the dex_performance / lp_registry sections.
- [ ] **Export-time validation in main tool** — block (or hard-confirm) export if astroport_data has all-empty epochs. The May 9 ext-tool fix disables export when nothing's captured; main tool should mirror this.
- [ ] **Astroport TRPC schema canary** — periodically test what `dateRange` enum values are accepted, since `D90` silently dropped. Future-proof against another silent change.

### NFT Explorer / dashboard tile work
- [ ] DAO Broken/Held NFTs: live-filter the `nfts` array against the 3 multisig wallet addresses. Hardcoded `1000` per Props 64-69 is correct but not future-proof. Need the two liquidity-wallet addresses (`...8ywv`, `...417v`) added to the codebase first.
- [ ] LST hardcoded ratios (`bLUNA || 1.6048` etc., ~10 places) — soft Design Principle #1 violation but ratios drift slowly. Decide: keep or replace with spinner.
- [ ] Asset metadata for `wBTC.creda.a` — token never appeared in any epoch JSON (not in pool registry, not in token prices, not in asset_metadata). Most likely root cause is upstream (TLA hasn't whitelisted a pool for it yet, or the Astroport pool-discovery bug above is hiding it). Re-check after Astroport fix lands.

---

## 🚀 Future projects — separate threads

### TLA data collection automation — see `DESIGN_tla_data_automation.md`
6–10 focused sessions, plus testing/verification weeks. **Don't start until admin-tool stabilization above is complete.** Phases:
- **Phase 0** — Investigations (Eris Amp Compounder farm registry, bucket VP query format, Astroport TRPC stability, Phoenix Directive API existence, Votion full API surface, Astroport pool registry endpoint)
- **Phase 1** — Daily cron with chain-deterministic data → new `tla-daily-YYYY-MM-DD.json` written to a new `tla_daily_storage` repo, runs *alongside* `tla_tool.html` (does NOT replace it)
- **Phase 2** — Score replication (port compute logic from `tla_tool.html` into the cron), validate parity for ≥3 weeks of overlap
- **Phase 3** — Replace manual paste steps where APIs exist; thin paste fallback retained otherwise
- **Phase 4** — Cutover; manual tool deprecated but kept as fallback

### Slim manual capture by prefilling chain-derivable fields
Pair this with the cron work or do standalone. The Rev 3.31 `fetchDaoTlaVp` work proved the live-query pattern. Identifying which manual-paste fields are actually fetchable from chain is mostly mechanical — start with whatever the user pastes most often.

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

- [ ] Should the cron run daily or hourly? (Astroport TRPC ratelimits unknown — moot until cron is on the agenda)
- [ ] Where does the new cron write — existing `tla_json_storage`, or a new `tla_daily_storage` repo for daily files?
- [ ] How does the cron handle the multi-week capture gap (epochs 183, 184 missing as of May 9, 2026)?
- [ ] LST ratios: keep the hardcoded fallbacks or remove? (See Design Principle #1; current call is keep.)
