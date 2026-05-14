# Changes Pending — aDAO website

> Rolling list of identified work for upcoming sessions. See PROJECT_KNOWLEDGE.md "Tracking responsibilities" for what goes here vs. there.
> Older completed items have been pruned — they live in changelog files (`index-log.md` etc.) instead.

---

## 🛠 Active / next round

### 🔥 Epoch numbering off-by-one bug — fix across 5 cron scripts
**Discovered 2026-05-14.** All Node.js cron scripts compute epoch as `Math.floor((now - 2022-10-31) / 7days)` which is 0-indexed. The canonical source (`epoch_1-300_date.json` in `tla_json_storage`) is 1-indexed. Week of May 11-18, 2026 is canonically epoch 185; crons label it 184. Dates are correct, only the integer label is off.

Affected scripts (in `defipatriot/cron-scripts`):
- `tla-snapshot/tla-snapshot.js` — `currentEpochInfo()` function ~line 202
- `adao-positions/adao-positions.js` — `currentEpochInfo()` function ~line 273
- `astroport/astroport-snapshot.js` — `timestampMsToEpoch()` function ~line 317
- `votion/votion-snapshot.js` — uses Eris API's `period` field directly (already canonical) — verify, may not need change
- `skeletonswap-lp_data/` — need to find epoch calculation
- `bribes-history/` — need to find epoch calculation

**Coordinated fix needed.** Adding `+ 1` is the math fix, but several crons also use the epoch number in archive filenames (`weekly/epoch-{N}.json`, `astroport-epoch-{N}.json`, `2026-epoch-{N}.csv`, `by-epoch/epoch-{N}.json`). Decision needed: rename existing files, OR accept a one-epoch gap in archives going forward (epoch-184 then epoch-186 with no 185).

Recommended: accept the gap, document in each data repo README, fix forward. Note that `votion-data_2026` numbering doesn't need changing — it uses Eris's `period` field which is already 1-indexed.

### 🟢 dao-tla.html — new Member Stats page (Pass 2)
Member-level breakdowns deferred from the `tla-stats.html` V6 rebuild. Data already collected in `adao-positions/current.json` members array (46 members, each with summary VP, vote allocations, locks, rewards, bribes claim status). Spec:
- Standalone page (linked from tla-stats.html "Member Stats" tab)
- Search-as-type member list (filter by handle / wallet address)
- Per-member portfolio panel: locked VP, individual locks, vote allocations, pending rewards, pending bribes
- Optional: leaderboard by VP, recent activity

### Trend chart accumulation (low priority — passive)
The stat-tile mini sparklines on `tla-stats.html` are currently empty because only `epoch-184` weekly archive exists (cron started capturing this week). Will populate naturally as weekly snapshots accumulate — probably 4+ weeks needed for visual signal. No action needed; just wait.

### Token grade scoring formula refinement
Current `computePoolScores()` in `tla-stats.html` is a simplified stub. Real scoring should weight access/performance/support based on the criteria from `tla_config.json`. Refine once enough historical data accumulates to validate output.

### Resilience prereqs (per index-log Rev 3.27 / 3.29 follow-ups)
- [ ] Per-fetch try/catch isolation in `index.html`'s `fetchLiveOnChainData` so one failed treasury balance fetch doesn't poison every downstream calculation
- [ ] Fallback LCD endpoint on 5xx — both `terra.publicnode.com` and `terra-lcd.publicnode.com` returned 500s on May 8, 2026 and the cascade null-safety added in 3.27 was the only thing preventing total dashboard failure
- [ ] Apply the same parallel epoch-fallback pattern from `fetchTlaFromGitHub` to `fetchTlaData` (TLA Stats page) — works correctly today but unnecessarily slow on stale fallback
- [ ] Per-fetch timeout in `fetchLiveOnChainData` for the slow Terra LCD calls — 14.67s load times mostly come from there

### NFT Explorer / dashboard tile work
- [ ] DAO Broken/Held NFTs: live-filter the `nfts` array against the 3 multisig wallet addresses. Hardcoded `1000` per Props 64-69 is correct but not future-proof. Need the two liquidity-wallet addresses (`...8ywv`, `...417v`) added to the codebase first.
- [ ] LST hardcoded ratios (`bLUNA || 1.6048` etc., ~10 places) — soft Design Principle #1 violation but ratios drift slowly. Decide: keep or replace with spinner.

---

## 🚀 Future projects — separate threads

### ✅ TLA data collection automation — COMPLETED 2026-05-12 → 2026-05-14
**Done.** 7 production crons live on Render (votion, skeletonswap, astroport, bribes-history, network-and-prices, tla-snapshot, adao-positions). All writing to their respective `*-data_2026` GitHub repos. `tla-stats.html` rebuilt (V6) to consume from the new continuous data sources instead of per-epoch manual snapshots. See PROJECT_KNOWLEDGE.md "TLA cron infrastructure" section for the as-built architecture. `tla_tool.html` and `tla-tool_ext.html` retained as manual fallback but no longer the primary capture path.

Known remaining work (see Active section above):
- Epoch off-by-one fix across all crons
- `dao-tla.html` Member Stats page (Pass 2 of the website rebuild)
- Score formula refinement once data accumulates

### Slim manual capture by prefilling chain-derivable fields
Pair this with the cron work or do standalone. The Rev 3.31 `fetchDaoTlaVp` work proved the live-query pattern. Now mostly moot since the new crons cover most of what was manual.

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

- [ ] **Epoch numbering fix — rename existing archives or accept gap?** When fixing the off-by-one bug, do we rename the existing `epoch-184.json` files to `epoch-185.json`, or just go forward with corrected numbering (creating a one-epoch gap in archives)? Recommendation: accept the gap, document it, fix forward.
- [ ] LST ratios: keep the hardcoded fallbacks or remove? (See Design Principle #1; current call is keep.)
- [x] ~~Should the cron run daily or hourly?~~ — **Resolved.** Hourly for tla-snapshot + network-prices (data freshness matters), daily for DEX/bribes captures (less time-sensitive), weekly for adao-positions + votion (epoch-aligned).
- [x] ~~Where does the new cron write?~~ — **Resolved.** One `*-data_2026` GitHub repo per cron. Independent systems principle.
- [x] ~~How does the cron handle the multi-week capture gap?~~ — **Resolved.** Crons started fresh in May 2026 with no historical backfill. Historical data before this lives in legacy `tla-ext_json_storage` files (used for trend charts only).
