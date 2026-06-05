# TLA Catalog Changelog

This is the change history for the TLA Chain Registry catalog system:
- `tla-catalog.html` (verification surface)
- `cron-scripts/chain/tla-registry/tla-registry.js` (the producer cron)
- `tla-chain-registry/2026/current.json` (the data artifact)

Newest revisions on top. Times are UTC. Cron-side and page-side changes are interleaved by date — when both shipped together, they share a Rev entry.

---

## Rev 0.13 — 2026-06-05 (wallet names + avatars)

User-reported issue: the catalog page showed "a bunch of addresses but really not member names." Investigation found that DAODAO PFPK profile names (160 wallets) and avatars (43 wallets) WERE being captured by the cron — but the page didn't render them. The data layer was good; the rendering layer was incomplete.

### Coverage before / after

| State | Wallets with meaningful name on card |
|---|---|
| **Before Rev 0.13** | 2 (just curated `aDAO Treasury` and `aDAO Council Multi-Sig`) — other 666 showed truncated address as title |
| **After Rev 0.13** | 668 of 668 (100%) — every card has either a real name, a PFPK profile name, or a "{DAO} member" identifier |

Breakdown after fix:
- 2 wallets show curated labels (existing)
- 160 wallets show PFPK profile names (existing data, newly rendered)
- 506 wallets show `{DAO} member` synthesized label (new from cron Rev 0.13)
- 43 wallets render their PFPK NFT avatar as the card icon (was generic fa-user)

### Page changes

**Name resolution priority chain extended.** `daodao_name` now in the fallback chain for card titles and sort comparisons:
```
headline_name → display_name → label → daodao_name → symbol → truncated address
```

**Wallet card icon renders PFPK avatar when available.** ipfs:// URLs rewritten to ipfs.io public gateway; `<img onerror>` falls back to fa-user if 404.

**Wallet card subline now shows primary DAO membership** instead of generic "member · DAODAO". Picks the most prominent membership (TLA > highest VP > first). Format: "TLA (+2 more)" when wallet is in multiple DAOs.

**Wallet detail view gets a PFPK profile panel** at the top showing avatar + name + brief explanation when present.

### Cron changes — `headline_name` for wallets

Post-PFPK-enrichment loop computes a canonical `headline_name` per wallet using priority:

1. Curated `label`
2. `daodao_name` (PFPK)
3. `{DAO} member` synthesized from primary `dao_memberships` entry
4. (left null — page falls through to address)

Result: downstream consumers (future `dao-tla.html` Member Stats page, `tla-stats.html`, any other page) read one canonical field instead of duplicating the priority logic.

### What this doesn't (yet) solve

- The 506 wallets with synthesized "{DAO} member" labels still don't show a real person's name — those people just haven't registered a PFPK profile. Out of our hands; PFPK is opt-in.
- The ~46 specific TLA council member wallets (the ones holding voting_escrow NFTs) include some without PFPK names that are nonetheless known to the community. These should be curated into `wallets.json` for full coverage. See "Open items" in CHANGES_PENDING.md (P2 — TLA council member curation).

### Deploy state

Not yet deployed:
- **Cron**: 138,771 bytes (+2.0 KB vs Rev 0.12.2)
- **Page**: 122,787 bytes (+4.8 KB vs Rev 0.12.2)

Both files syntax-validated. Page change works immediately on deploy (reads existing `daodao_name` field from current.json); cron change populates the new `headline_name` field on next run.

---

## Rev 0.12.2 — 2026-06-05 (cron CDN cache bypass)

User reported logos still broken even after Rev 0.12.1 deploy. Investigation revealed the corrected URLs WERE in the deployed `token_overrides.json` on GitHub, but the cron read STALE data when it ran.

### Root cause — GitHub raw URL CDN caching

`raw.githubusercontent.com` is fronted by Fastly with a 5-minute cache. Sequence of events:

1. User pushed corrected `token_overrides.json` to GitHub at T+0
2. User triggered cron manually at T+2 min (well-intentioned but premature)
3. Cron's `fetchCurated()` hit `raw.githubusercontent.com/.../main/curated/token_overrides.json`
4. Fastly served the OLD cached file (from before the push) — `x-cache: HIT`, `source-age: 164s`
5. Cron wrote stale URLs into `current.json`

The deployed file on GitHub had the correct URLs, but the cron's view of it was 2-3 minutes behind.

### Things tried that DIDN'T work

- Query-string cache-buster `?_=${Date.now()}` — Fastly ignores query strings for these URLs
- `Cache-Control: no-cache` request header — Fastly ignores
- `Pragma: no-cache` request header — Fastly ignores

### The actual fix — SHA-pinned URLs

Raw GitHub URLs are cached by full path. Different SHA = different path = different cache key = guaranteed cache miss for unseen SHAs.

`fetchCurated()` now:

1. Hits GitHub API once: `api.github.com/repos/{repo}/commits/{branch}` → returns latest commit SHA
2. Builds curated file URLs using that SHA: `raw.githubusercontent.com/{repo}/{sha}/curated/{file}.json`
3. Fetches all curated files from those SHA-pinned URLs

If the SHA lookup fails (rate limit, API down), falls back to the branch-name URL with a warning logged. So worst case, behavior matches pre-fix — no new failure mode.

GitHub API has 60 unauthenticated requests/hour limit. We use 1 per cron run. Trivial budget impact.

### Why the user's report was correct

State after Rev 0.12.1 deploy + first cron run:

| Token | Deployed override file | Cron's view of override (current.json) |
|---|---|---|
| arbLUNA | `arbluna.svg` ✓ | `arbLUNA.png` ⚠ (stale) |
| xASTRO | `xAstro.svg` ✓ | `xASTRO.png` ⚠ (stale) |
| FUEL | `neutron/.../fuel.png` ✓ | `migaloo/.../fuel.png` ⚠ (stale) |
| ampWHALE | `ampwhale.svg` ✓ | `ampWHALE.png` ⚠ (stale) |
| ampLUNA | (no override needed) | `ampluna.svg` ✓ (picked up from chain-registry source via SVG fix) |

So ampLUNA fixed via the SVG fallback, but the 4 with overrides got stale data.

### Recovery procedure for Rev 0.12.1 (one-time)

Before deploying Rev 0.12.2, the user just needed to trigger ONE more cron run — by then the 5-minute Fastly cache had expired, so the cron would get the correct curated data.

After deploying Rev 0.12.2, this class of bug is permanently fixed — future curated pushes can be immediately followed by manual cron triggers without waiting.

### Deploy state

- **Cron**: 136,746 bytes (+1.6 KB vs Rev 0.12.1) — single function rewrite in `fetchCurated()`
- Other files unchanged

This rev fixes the MECHANISM. The user should additionally trigger a manual cron run after deploy to refresh `current.json` with correct logo URLs.

---

## Rev 0.12.1 — 2026-06-05 (logo URL hotfix)

User-reported visible failures on arbLUNA, ampLUNA, xASTRO, FUEL after Rev 0.12 deploy. Root-caused two distinct issues:

### Issue 1 — Most curated URLs in Rev 0.12 were wrong

Of the 17 new URLs added to `token_overrides.json`, **12 were 404**. The chain-registry uses:
- **Filename casing** that doesn't match the symbol (`arbluna.svg` not `arbLUNA.png`, `xAstro.svg` not `xASTRO.png`)
- **SVG file extensions** for many newer entries (we assumed PNG)
- **Different chains** than we assumed (FUEL is on Neutron, not Migaloo)

**Fix:** audited every URL against the actual chain-registry assetlist.json files. Found correct URLs for 11 of 12. The 12th (rSWTH) doesn't have its own logo in chain-registry; fell back to parent SWTH icon as a pragmatic substitute. All 20 URLs now HTTP 200 verified.

| Token | Old (404) | New (200) |
|---|---|---|
| arbLUNA | `terra2/images/arbLUNA.png` | `terra2/images/arbluna.svg` |
| dATOM | `cosmoshub/images/dATOM.png` | `neutron/images/dATOM.svg` |
| xASTRO | `neutron/images/xASTRO.png` | `neutron/images/xAstro.svg` |
| wstETH | `_non-cosmos/ethereum/images/wsteth.png` | `_non-cosmos/ethereum/images/wsteth.svg` |
| ampWHALE | `migaloo/images/ampWHALE.png` | `migaloo/images/ampwhale.svg` |
| FUEL | `migaloo/images/fuel.png` | `neutron/images/fuel.png` |
| wETH.wh / WETH.axl | `_non-cosmos/ethereum/images/weth.png` | `_non-cosmos/ethereum/images/weth.svg` |
| wSOL.wh | `solana/images/sol.png` | `_non-cosmos/solana/images/sol.svg` |
| wBNB.wh / wBNB.axl | `bsc/images/bnb.png` | `_non-cosmos/binancesmartchain/images/bnb.png` |
| rSWTH | `carbon/images/rSWTH.png` | `carbon/images/swth.png` (parent fallback) |

### Issue 2 — Cron chain-registry extractor skipped SVG-only entries

The cron's `indexChainRegistry` function only picked up `.png` URLs from chain-registry's asset entries. Many newer tokens (ampLUNA, arbLUNA, xAstro, ampwhale, etc.) are SVG-only in chain-registry — so the cron silently dropped them from its source data.

This explained why `sources.cosmos_chain_registry.logo_uri` was null for ampLUNA in our live data, even though ampLUNA HAS a logo in chain-registry. SS source happened to have it too (which is what was actually being used).

**Fix:** `indexChainRegistry` now picks up PNG OR SVG (preferring PNG when both exist). Both render fine in `<img>`. Future-proofs as chain-registry continues migrating to SVG.

### Verification

- 36 single tokens still have resolved logos (same count, but now all 20 curated ones actually work)
- The 4 user-reported failures (arbLUNA, ampLUNA, xASTRO, FUEL) all verified to load correctly with new URLs
- Cron's SVG fix will add 1+ more chain-registry logos on next run (modest immediate gain, important future-proofing)

### Deploy state

- **Cron**: 135,181 bytes (+464 bytes vs Rev 0.12) — single SVG-fallback fix in `indexChainRegistry`
- **token_overrides.json**: 10,948 bytes (+321 bytes vs Rev 0.12) — 12 URL corrections, no schema changes

Page (`tla-catalog.html`) does not need a hotfix — its fallback chain works correctly; the data underneath was the problem.

---

## Rev 0.12 — 2026-06-05 (token logos)

Spans all three layers — curated data, cron aggregation, page rendering — implementing a unified token-logo system across the catalog. Three layers of fallback so users always see *something* recognizable for every token.

### Curated — `token_overrides.json` extended with `logo_url`

Added a `logo_url` field to 20 single-token override entries (3 existing wBTC variants now have logos; 17 new entries covering ATOM, dATOM, USDC, USDt, EURe, ASTRO, xASTRO, wstETH, INJ, SWTH, rSWTH, WHALE, ampWHALE, bWHALE, FUEL, arbLUNA, stLUNA, wETH.wh, wSOL.wh, wBNB.wh, wBNB.axl, WETH.axl, wBTC.creda.a).

URLs sourced from the `cosmos/chain-registry` repo's per-chain folders (cosmoshub, neutron, migaloo, carbon, stride, terra2, `_non-cosmos/ethereum`, solana, bsc). The pattern is `https://raw.githubusercontent.com/cosmos/chain-registry/master/{chain}/images/{symbol}.png`. Where uncertain, the page's `<img onerror>` handler falls back to a colored letter circle — no broken-image experience.

Schema extended in `_meta`: `logo_url` documented as the canonical field.

### Cron — Stage 7d added (logo aggregation)

After Stage 7 applies curated overrides, the new Stage 7d resolves a single canonical `logo_url` per token using priority order:

1. **`token.override.logo_url`** — curated (highest priority)
2. **`sources.cosmos_chain_registry.logo_uri`** — covers terra-2-native tokens like LUNA, ROAR, ampLUNA
3. **`sources.skeletonswap.logo_url`** — covers additional wrapped tokens
4. *(future)* Eris CDN, Astroport API, CoinGecko per-coin endpoint — none implemented yet

**Simulation result against current live data:**
- 36 of 173 single tokens resolve to a direct logo URL
  - 20 via curated override
  - 13 via chain-registry
  - 3 via SkeletonSwap
- 137 remaining tokens have no direct logo (will use letter fallback)

For LPs and amplps the cron does not compute a composite — that's a rendering concern handled page-side using the existing `scope.lp_to_underlyings` + `amplp_mappings` data.

### Page — three rendering paths

New helper functions before `renderCard`:

```
entryLogoHtml(entry, sizePx)   — main entry point. Picks single vs composite vs FA icon.
tokenLogoHtml(token, sizePx)   — single-token <img> with onerror letter fallback.
lpLogoHtml(token, sizePx)      — composite of 2 underlying tokens (overlapping circles).
logoFallbackInitials(name)     — derives 2-letter initials (handles "X-Y LP", "ampLUNA", etc).
logoFallbackColor(str)         — stable color hash so the same token always gets the same color.
```

Used in two places:
- **Card header** at 28px (replaces the FontAwesome `fa-coins` icon)
- **Detail modal header** at 56px (large, prominent)
- **"Wraps" panel** on amplp detail at 40px (shows the wrapped LP's composite)

**Three rendering paths:**
1. **Single token w/ logo** → `<img>` with `onerror` swap to letter circle
2. **LP / amplp** → composite of 2 overlapping circles (each 72% of card size, second offset right)
3. **No logo data** → deterministic letter circle (colored by hash of token name)

**Page-side fallback:** `tokenLogoHtml` re-does the cron's priority lookup directly from `token.sources` if `token.logo_url` isn't yet populated. This lets the page work as soon as curated logos are committed to GitHub — no need to wait for cron deploy.

**Composite coverage:** 133 of 137 LP/amplp tokens get a full composite (both underlying tokens have resolved logos). The remaining 4 have at least one underlying without a logo — these still render gracefully with letter fallback for the missing side.

**Contracts and wallets** keep their FontAwesome icons (`fa-file-contract`, `fa-user`) — they're not asset tokens, so a logo concept doesn't apply.

### CSS additions

```css
.token-logo-fallback   — colored letter circle (stable per-token color)
.token-logo-wrap       — img wrapper with consistent sizing
.token-logo-composite  — overlapping wrapper for 2-token LP/amplp composites
```

The composite has a thin dark border + cyan glow so each component circle reads clearly when they overlap.

### Deploy state

Not yet deployed:
- **Cron**: 134,717 bytes (deployed: 133,172; +1.5 KB)
- **Page**: 118,029 bytes (deployed: 109,196; +8.8 KB)
- **token_overrides.json**: 10,627 bytes (deployed: 4,552; +6.1 KB)

All three files validated. Cron + page can deploy independently — page works in fallback mode if cron's Stage 7d hasn't run yet.

### Why three layers (instead of just one)

Could have done page-only rendering. Reason for going all three layers:

- **Curated overrides** are durable knowledge — a logo URL is just metadata about a token, no different from its display name or notes
- **Cron aggregation** makes the data reusable — future pages (tla-stats, Member Stats, Portfolio Tracker) read `t.logo_url` directly instead of duplicating the priority-resolution logic
- **Page rendering** is where the visual composition happens (single vs LP composite vs letter fallback)

Each layer has a single responsibility. Adding a new logo to a token means editing one JSON file; everything downstream picks it up next cron run.

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
