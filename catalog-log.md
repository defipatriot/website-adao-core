# TLA Catalog Changelog

This is the change history for the TLA Chain Registry catalog system:
- `tla-catalog.html` (verification surface)
- `cron-scripts/chain/tla-registry/tla-registry.js` (the producer cron)
- `tla-chain-registry/2026/current.json` (the data artifact)

Newest revisions on top. Times are UTC. Cron-side and page-side changes are interleaved by date — when both shipped together, they share a Rev entry.

---

## Rev 0.11 — 2026-06-05 (amplp classification fix)

Real bugs found by user inspection of the amplp_tokens tab. Two cron bugs root-caused, plus three page-side improvements to make the amplp tab self-explanatory.

### Cron — Bug A: 55 of 65 amplps had wrong subtype

**Symptom:** the amplp_tokens tab showed only ~10 entries despite `amplp_mappings` knowing about 65. The other 55 amplps were classified as `subtype='native'` (factory denoms inherited this from the generic catch-all) or `subtype='lst'` (the LST regex falsely matched amplps whose names contained "luna" — e.g. `arbLUNA-LUNA AMPLP`).

**Root cause:** Stage 5c only set `subtype='amplp'` on tokens it created from scratch. Tokens that were already in `tokens[]` (because Eris's `/prices` returned them) got their subtype set by the later generic logic at the bottom of `buildTokenCatalog`. Two paths corrupted them:

```js
// Generic inference at line 1407-1413:
if (t.type === 'factory') t.subtype = 'native';   // wrong for amplps
// LST regex at line 1415 unconditionally overrode:
if (/^(amp|arb|b|st)/.test(t.symbol) && /luna/i.test(sym)) t.subtype = 'lst';
```

**Fix:**
- New Stage 5d normalizes ALL entries in `amplp_mappings` to `subtype='amplp'` regardless of how they entered the catalog
- LST detection at line 1415 now guards against overriding `'amplp'`

### Cron — Bug B: every amplp showed `tla_pools_count: 0`

**Symptom:** every amplp displayed "Appears in TLA pools: no" on the catalog page. But amplps DEFINITELY appear in TLA pools — staking the amplp IS how you participate in a TLA gauge.

**Root cause:** Stage 5 credits the LP token entries in `pools[]`. Stage 5b backfills underlying tokens via `lpToUnderlyings`. Neither touches amplps because amplps are wrappers, not pool entries and not underlyings. So they never got their `tla_pools_count` incremented.

**Fix:** Stage 5d (same new stage) mirrors `appears_in.tla_pools` from the underlying LP onto the amplp (or falls back to `mapping.bucket` if the underlying LP isn't itself a tracked TLA pool — happens for legacy amplps). Also inherits `gauge_status` and records `wraps_lp_address` for the page to use.

**Simulation against current live data:**
- 55 amplps got their subtype corrected
- 65 of 65 amplps got TLA pools backfilled
- Examples: `ampWHALE-ampLP → single`, `LUNA-USDC-ampLP → stable`, `LUNA-ampLUNA-ampLP → project`

### Page — DEX badge on amplp/LP cards

Derives from name pattern + the new `wraps_lp_address` link:
- Underlying LP name ends in `(S)` → **Skeleton Swap** badge (amber)
- Underlying LP is a hyphenated pair → **Astroport** badge (blue)
- Amplp wraps a single token (no hyphen, no "LP") → **Single-asset vault** badge (purple) — Eris's compounder, not a DEX pool

Renders on both amplp_tokens AND lp_tokens cards. Tooltip explains "white_whale-pool architecture, operated by Backbone Labs" for (S) variants.

### Page — "Wraps" relationship panel on amplp detail view

Top-of-detail panel for any amplp showing:
- **Wraps:** the underlying LP's display name
- **Wrapped LP address:** monospace, truncated, hover for full
- **Underlying DEX:** Astroport / Skeleton Swap / Single-asset vault (with hover explainer)
- **TLA bucket:** which bucket the staked position counts toward

Makes the "amplified version of [X]" relationship visible without forcing the user to deduce it from the name.

### Page — Category subtitle banner

Above the grid, when a specific tab is active (not "all"), shows a one-liner explaining what's in that tab. Especially important for `amplp_tokens`:

> *Amplified LP tokens — auto-compounding wrappers around regular LPs. Stake the underlying LP to receive these; rewards auto-compound back into more LP. The non-amplified LP version lives in the "lp tokens" tab.*

Helps users navigating between `lp_tokens` and `amplp_tokens` understand the relationship (you're depositing the regular LP to GET the amplp back; amplp is the wrapper that does the auto-compounding work).

### Deploy state at end of Rev 0.11

Not yet deployed (bundled for review):
- **Cron**: 133,172 bytes (deployed: 129,407; +3.8 KB)
- **Page**: 109,196 bytes (deployed: 101,730; +7.5 KB)

Both files syntax-validated. Cron simulation against current `current.json` shows expected results (55 subtype corrections, 65 TLA-pool backfills).

---

## Rev 0.10 — 2026-06-02 (audit night)

**Major data-quality audit + Phase 0 documentation pivot.** This is the closing entry of a multi-hour session that identified ~10 systemic bugs in the cron's data layer, verified 17 cross-DEX same-named LPs against on-chain `pair{}` data, and produced the first round of durable knowledge files (this changelog, `queries.md`, updates to `PROJECT_KNOWLEDGE.md` and `CHANGES_PENDING.md`).

### Cron — what changed (built; deploy ready)

**Local cron file: 129,407 bytes** (deployed is 114,834 — diff ~14.5 KB).

#### Self-referential vault detection in scope phase
Eris's single-asset compounder vaults (ampCAPA at `factory/terra186rpf.../ampCAPA`, and any future similar) respond to `pair{}` as if they were 2-asset LPs, returning `asset_infos = [input_asset, self]`. Without intervention, `lp_to_underlyings` ended up with self-references that double-counted into `tla_pools_count` and incorrectly cascaded `is_amplp_underlying` to the vault itself.

The scope phase now detects `lpAddr ∈ underlyings`, strips the self-reference, and tags the entry `_is_vault: true`. Defense-in-depth: Stage 5b dedup catches anything that slips past; Stage 6 cascade skips wrappers when computing the underlying-flag.

#### Stage 5c — synthesize records for unpriced amplps
`amplp_mappings` (from `asset_compounder.asset_configs`) had 65 entries, but only 54 had corresponding `tokens` records. The 11 missing ones were amplps wrapping legacy/inactive LPs that Eris's `/prices` endpoint doesn't publish prices for (arbLUNA-LUNA AMPLP, WHALE-bWHALE AMPLP, WETH.wh-wstETH AMPLP, LUNA-wSOL.wh variants, USDC-USDt amplp duplicates).

Stage 5c now synthesizes minimal records from `amplp_mappings` data so all 65 wrappers appear in the catalog with correct subtype, bucket assignment, and `is_wrapped_by_amplp` flags. They're priced as `null` (honest data over false positives).

#### Stage 6 — cascade `is_amplp_underlying` to the right layer
The previous logic stamped `is_amplp_underlying: true` on LP tokens themselves, which is incorrect — the LP token IS what's wrapped; its UNDERLYING assets (LUNA, ATOM, etc.) are the things "underlying an amplp." Two distinct fields now:

- `is_wrapped_by_amplp` — set on the LP/wrapper itself (the thing the amplp factory consumed)
- `is_amplp_underlying` — set on the underlying assets of any LP that's wrapped

FUEL now correctly shows "yes" — its LP is wrapped, so FUEL is underlying an amplp via that wrapping. Page filter "amplp underlying = yes" matches either flag with a clarifying parenthetical.

#### Stage 7b — Hardcoded override system + source propagation
Some naming conflicts aren't data bugs, they're branding disputes. Eris's `/prices` API returns `"display": "bLUNA"` for Backbone Labs' staked LUNA, but Eris's official UI calls it `"boneLUNA"`. The catalog now has a small `HARDCODED_OVERRIDES` dict (Stage 7b) for these cases.

Override now propagates to BOTH the headline_name AND `sources.eris.display`, preserves the original raw value as `sources.eris._display_original`, and sets `_display_overridden: true` for page-side transparency. Without this, a later headline-name recompute clobbered the override.

#### Stage 7c — CG verification with bridge-trace fallback
Eris's `/prices` sometimes claims a CoinGecko ID that's wrong. The Stage 7c verification stage independently looks up each claimed CG ID against CoinGecko's `terra-2` platform index and adds a fallback that follows `bridge.all_traces` to source chains (e.g., ethereum) for tokens CG indexes by source rather than by Terra.

Catches caught this round:

| Token | Eris claimed | CG actually has | Status |
|---|---|---|---|
| USDC | `usd-coin` (Circle's generic) | `ibc-bridged-usdc` (Noble variant, "USDC.N") | mismatch |
| EURe | `monerium-eur-money` (deprecated v1, labeled "EURe [OLD]") | `monerium-eur-money-2` | mismatch |
| WETH.axl | `ethereum` (native ETH!) | `weth` | mismatch |
| ATOM | (none provided) | `cosmos` (terra-2 direct) | discovered |
| INJ | (none provided) | `injective-protocol` | discovered |
| WBTC.axl | (none provided) | `axlwbtc` | discovered |
| wBTC.atom | (none provided) | `eureka-bridged-wbtc-terra` | discovered |
| PAXG | (none provided) | `pax-gold` (via Ethereum 0x45804880De... bridge trace) | verified_via_bridge |

Scoring stage no longer overwrites match status with `'matched'` whenever a CG ID exists. Uses precise verification result (`verified` / `verified_via_bridge` / `discovered` / `mismatch` / `unverified_no_terra_addr` / `hardcoded_override` / `no_mapping`).

#### Stage 8b — Auto-suggested acquisition from bridge data
For tokens with no curated `acquisition_guides.json` entry, Stage 8b synthesizes a guide from `bridge.all_traces` data. Surfaces useful info like "USDt → bridged from Kava via channel-X" without requiring a council member to write it from scratch first.

#### `source_coverage` block in snapshot
The snapshot now exports per-source asset counts + `fetched_ok` status. Powers the page's informative tooltips when a row shows "— not listed" — instead of an ambiguous empty cell, users see "chain-registry indexed 58 assets; this address not among them."

### Page — what changed (built; deploy ready)

**Local page file: 101,721 bytes** (deployed is 96,805 — diff ~5 KB).

- **`verified_via_bridge` badge** — teal "✓ via bridge" pill with tooltip showing source chain (e.g., "via Ethereum 0x45804880...")
- **Source coverage tooltips** — every "— not listed" row now has informative tooltip from the new `source_coverage` block
- **"n/a (single-asset stake)"** for Astroport/SS rows on tokens that aren't tradeable pairs (vault assets like ampCAPA)
- **amplp underlying = OR of both flags** with clarifying parenthetical "(wrapped by amplp)"
- **`override` badge** (purple) on the Eris UI row when hardcoded override propagated, with tooltip showing the original raw value (e.g., "Eris API: bLUNA → display override: boneLUNA")
- **Filter button** for "amplp underlying" now matches either flag

### Verified by deposit + on-chain query

User tested by depositing standard-IBC ATOM into BOTH Astroport's ATOM-LUNA LP and Skeleton Swap's ATOM-LUNA LP. Both accepted. Then on-chain `pair{}` queries on 17 same-named pairs across both DEXes returned **identical underlying token addresses** in every case. Conclusion: there's ONE ATOM token on Terra (and ONE USDC, ONE LUNA, etc.) — SS's API just labels its denom field with misleading metadata strings. The catalog's `lp_to_underlyings` data is authoritative.

This empirical verification killed three prior wrong theories (different IBC denoms across DEXes; same pool contracts across DEXes; need for new "DEX scope expansion" pulling out-of-TLA pools into scope) and clarified Phase 0 scope: **TLA-only**.

### NOT deployed (intentionally)

The `dex-scope-fix.zip` Step E — would have added Astroport- and SS-pool API addresses into the catalog scope, pulling in out-of-TLA pools. Based on the wrong-theory chain above. Step E was reverted from the local cron file before this Rev was packaged.

### Documentation pivot

Phase 0 documentation completed:

- New file `catalog-log.md` (this file)
- New file `queries.md` — comprehensive on-chain query reference (all current cron queries + wishlist queries for Vote Intelligence, Portfolio Tracker, future query tool)
- Updated `PROJECT_KNOWLEDGE.md` — added `tla-registry` to cron infrastructure, added `tla-catalog.html` to Current pages table, added a major new section on the TLA Chain Registry catalog system, added a Critical catalog gotchas section with all the bug-class learnings
- Updated `CHANGES_PENDING.md` — added catalog work items (SS indexer rewrite, contract-version surfacing, label fixes, acquisition guide curation)

Future sessions should be able to resume context in ~30 minutes by reading the existing PROJECT_KNOWLEDGE.md + CHANGES_PENDING.md + this log file + queries.md.

---

## Rev 0.9 — 2026-06-02

Earlier in the same audit night, before the documentation pivot. These bundles built but not all deployed:

- `data-trust-fix.zip` — **DEPLOYED**. Foundation for CG verification stage. Cron jumped to 114,834 bytes.
- `headline-override-fix.zip` — built, superseded by source-transparency
- `amplp-completeness-fix.zip` — built, superseded
- `systemic-fix.zip` — built, superseded
- `bridge-cg-verification.zip` — built, superseded by source-transparency
- `source-transparency.zip` — built, **this is the correct cron head to deploy** (local file matches)

All these intermediate bundles were stepping stones toward the consolidated Rev 0.10 state above. They're listed here because the deploy timeline matters for any future audit reading the cron's commit history.

---

## Pre-Rev — initial Phase 0 build

The catalog system was bootstrapped over preceding sessions before the 2026-06-02 audit:

- **`tla-registry` cron** — created in `defipatriot/cron-scripts/chain/tla-registry/`. Pulls from 5 external sources (chain-registry, Eris `/prices`, Astroport `/api/pools`, SS `/api/pools/phoenix-1`, CoinGecko `/coins/list?include_platform=true`) and reconciles against on-chain queries on the TLA gauge + asset-staking + asset-compounder contracts.
- **Output repo** — `defipatriot/tla-chain-registry`. Daily output to `2026/current.json` + `2026/heartbeat.json`.
- **Catalog page** — `tla-catalog.html` in `aDAO-links-site`. Renders the catalog data with filters, tabs (tokens / lp_tokens / amplp_tokens / contracts), CG verification badges, take-rate panels for dewhitelisted entries, cron-status footer widget.
- **Curated files** in `website-adao-core`: `categories.json`, `wallets.json`, `protocols.json`, `known_contracts.json`, `token_overrides.json`, `acquisition_guides.json`.

The Pre-Rev period established the architecture but had many silent data bugs that the Rev 0.10 audit surfaced and root-caused. Detailed retrospective lives in the bug-history section of Rev 0.10 above.
