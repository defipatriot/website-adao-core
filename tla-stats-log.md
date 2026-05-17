# tla-stats.html — Changelog

Tracks dashboard changes for the TLA Stats page at thealliancedao.com. Cron data changes are documented in each cron's README; this file covers the rendering side only.

Reverse chronological. Newest at top.

---

## 2026-05-17

### Member Data overlay (new feature)

Header dropdown selector — pick any aDAO member, the Overview tab visuals update with their data overlaid. Pools / TLA Liquidity / aDAO tabs unchanged (member overlay is Overview-only by design).

When a member is selected:
- **VP Breakdown pie**: carves a member-colored slice out of "Other" — total VP unchanged.
- **Vote Breakdown waterfall**: adds an amber member layer to each pool the member voted in. Bucket totals row gains a member chip; per-pool tooltip gains a member row.
- **Threshold Watch**: filters to pools the member voted in. Header gains a "Filtered: {member}" badge. Empty states are member-aware ("None of {name}'s pools are at risk").
- **Member Stats Row**: 6 amber tiles below the global stat tiles — Astroport LPs, Skeleton LPs, Epoch Rewards, Epoch Bribes, Avg APR Non-Amp, Avg APR Amplified. Hidden by default; appears only when a member is selected.

### Critical bug fix: bribes resolver

`resolveTokenPriceFromInfo()` was looking up cw20 token prices at `entry.address`. The actual `network-and-prices` schema nests the address at `entry.prices.{source}.address` (or under `prices.{source}.all_chains.{chain}.address` for multi-chain tokens). Any bribe paid in a cw20 token (CAPA, ROAR, etc.) silently priced as $0.

**Impact before fix**: Global Epoch Bribes tile showed ~$820. After fix: ~$1,300 (about 58% more accurate). Member bribes tile correctly captures CAPA bribes (was 100% understated for members voting in LUNA-CAPA, ampCAPA).

Same resolver is used in `buildBribesIndex()` so this fix also corrects the per-pool bribe attribution used by waterfalls and ranking displays.

### Pool lookup keying

All member-overlay lookups now use `gauge_pool_id` (truly unique, e.g. `cw20:terra1wdz...`) instead of `name+dex` (which can collide e.g. two `LUNA-WBTC|Astroport|BLUECHIP` entries with different gauge IDs). Required adding `gauge_pool_id` passthrough to both pool normalizers in the rendering layer.

### Color scheme

- Member overlay color: amber (`#f59e0b`)
- "Other" VP: slate gray (`#64748b`) — was previously amber in waterfall, now consistent with pie chart slate
- Updated all 3 legends (waterfall totals, waterfall bottom, member tile row) for consistency

---

## Pre-2026-05-17

Prior changes not individually tracked. Major prior work that's reflected in the current file:
- Cron health registry for monitoring 9 production crons (nft-inventory and marketplace-stats added 2026-05-15)
- Tab structure: Overview / Pools / TLA Liquidity / Rankings / aDAO / Member Stats
- Live epoch + countdown header
- Loading status + data freshness indicators
- Background fetch wiring for all data repos

---

## Notes for future changes

- Always test against real production data before declaring ready — JS syntax check alone is not sufficient
- Use same filename when re-uploading (no `_v2` suffixes) so reviewers can diff against the previous version
- Member-overlay rerenders are wired through `onMemberSelectorChange()` — touch that function when adding new Overview-tab elements that should react to member selection
- Pool lookups must use `gauge_pool_id` not `name+dex`. The cron normalizer preserves `gauge_pool_id` on every pool object.
- Hard-coded pool names in render code (e.g. `if (p.name === 'LUNA-USDC')`) WILL break — there are multiple pools with that name across DEXes
