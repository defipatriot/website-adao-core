# TLA Stats Changelog

This is the change history for `tla-stats.html` (the Terra Liquidity Alliance public dashboard).
Newest revisions on top. Times are UTC.

---

## Rev 3 — 2026-05-30

Today's session. A long multi-part day on `tla-stats.html`: bribe/member tile work, the Overview leaderboard-tile and ranking-popup rework, the Vote Breakdown waterfall rebuild, a major data-accuracy audit against on-chain Eris/Votion ground truth (the variant-collision class + concentrated-pool discovery), the Pool Health / Liquidity-Growth / Threshold panels, member-position earnings, and the comprehension pass. Each sub-revision below is one test-file iteration that was reviewed and edited, oldest first. All work is in the rendering layer unless noted; cron-side items are collected in `CRON-FIXES-BRIEF.md`.

### What changed

#### Rev 3.1 — Bribe token breakdown (new)
The Epoch Bribes modal now labels each pool's bribe with the paying token (LUNA, ASTRO, CAPA, FUEL, etc.). Single-token pools show the symbol inline; multi-token pools show the largest-USD token with a "+N more" expander that reveals the full per-token breakdown (humanized amount and USD value per token). Token amounts respect per-token decimals (e.g. wBTC = 8), and any bribe token that can't be priced is labeled "(unpriced)" rather than silently dropped.

#### Rev 3.2 — Member selector now activity-filtered
The member dropdown is built live from the DAO DAO indexer roster (no manual list — new members auto-appear, names come from their PFPK profile). It now shows only members worth selecting: those with a name AND some TLA presence (an LP position, voting power, pending rewards/rebase/bribes, or an active lock). Named members with nothing to show and unnamed members (no PFPK profile) are hidden, trimming the list from the full roster to the active participants.

#### Rev 3.3 — Member status chip (new)
Selecting a member now shows a live status chip beside the dropdown: their total VP, total LP value, and a health dot — green (all their LPs are active), yellow (an LP sits in a pool at 1.0-1.5% of its bucket VP, i.e. near the 1% threshold), or red (any LP is inactive). A small legend button explains the colours. (The dot lives beside the dropdown rather than inside it because native select options can't be styled or coloured.)

#### Rev 3.4 — Member reward/bribe tiles relabeled + token breakdowns (new)
The member stat tiles now distinguish reward sources: "Member Epoch Rewards" -> "Member Deposit Rewards" (LP emissions, paid in zLUNA) and "Member Epoch Bribes" -> "Member Bribe Rewards" (vote-share estimate). Each tile has an expandable chevron that reveals the per-token breakdown — amount and USD per token — so it's clear what the rewards/bribes are actually paid in. Bribe tokens are resolved client-side through the same price resolver used for the pool bribes, fixing cases where the positions cron left IBC tokens (e.g. ASTRO) as a raw denom hash with a null USD value.

#### Rev 3.5 — Newcomer protection: per-pool risk flags + APR caveat (new)
The Pools tab now surfaces plain-language risk flags on each pool, aimed at people evaluating where to LP: "Near threshold" (active pool at 1.0-1.5% of bucket VP — may stop earning next epoch), "Thin liquidity" (pool depth < $25k — large trades move the price), and "Volatile reward (TOKEN)" (a bribe/reward token that moved >15% in 7 days). A note above the pool list explains that APRs are forward-looking estimates at current prices and emissions, lists the flags, and states this is not financial advice / do your own research. All derived from existing data (depth, bucket VP %, token price-change) — no new cron required. (Token-market liquidity isn't in the price feed, so the liquidity flag uses pool depth as a proxy.)

#### Rev 3.6 — Overview leaderboard tiles enriched (new)
The three Overview tiles now carry more context. The Liquidity tile gains a DEX TVL / TLA-staked toggle (DEX = total pool liquidity on the DEX; TLA-staked = the portion staked into the gauge that actually earns), each with its own rank movement and vs-prior delta. The APR tile now shows the drivers of an APR move inline — the pool's VP change vs last epoch and its current bribe total — so a reader can see whether APR shifted because of votes or bribes. Each tile's click-through popup is expanded into a "nerd-out" view: every row shows the full cross-metric breakdown (APR, VP + VP change, bribes with PD/Other split, DEX liquidity, TLA-staked, volume). Each tile's % delta measures its own metric's change (volume vs volume, etc.) — they are intentionally not the same number.

#### Rev 3.7 — Overview tiles: fixes + cleaner drivers
Fixed a variant-collision bug where a pool with two gauges in one bucket (e.g. LUNA-WBTC: an active ~7.7M-VP gauge plus a dead 18K-VP one) compared the active row against the wrong prior variant, producing absurd deltas like +41,884%. Prior VP/TLA-staked baselines now pick the largest (active) variant per pool name. Fixed the TLA-staked toggle showing "No liquidity data" (the field wasn't carried into store.pools). Removed the noisy "No Change" rank badge — movement now shows only when rank actually changed. APR-tile drivers reworked: voting power shown as votes with a green/red % change vs last epoch, plus Votion's projected current->next shift (purple/red), plus current bribe level. Bribe direction vs last epoch is intentionally NOT shown — per-epoch bribe history isn't captured yet, so a token-amount-vs-USD up/down arrow can't be derived honestly.

#### Rev 3.8 — Overview tiles: clean uniform rows + drivers moved to popup
Reworked per feedback. The variant-collision VP bug was actually upstream: store.poolHistoryByKey collapses two same-name|bucket gauges into one key and kept the dead variant, so even the "largest variant" fix read the wrong number. Now prior VP/staked baselines read the RAW poolStatusHistory.pools list (both variants) and keep the largest — LUNA-WBTC correctly reads +1.2%, not +41,884%. The three dashboard tiles are now structurally identical (rank · name · DEX · value · one delta) so they line up cleanly; all driver detail moved into the click-through popup. The popup now shows, per pool: APR change, votes change, TLA-staked change, Votion projected shift, and bribe level (with PD/Other split) — each colored green/red — plus a plain-language "why APR moved" line that names whether votes or staked-LP drove it (e.g. "APR down 67% — staked LP rose 40%: more LP spreads the same rewards thinner"). Bribe direction is still a level not an up/down arrow (no per-epoch bribe history yet).

#### Rev 3.9 — Overview tiles: TLA-staked field fix + honest APR comparisons
Root-caused the TLA-staked "No liquidity data" (and the bogus "staked LP fell 100%" in the popup): the votePools builder that feeds store.pools never set tla_staked (only a separate active_pools/buildTlaPool path did), so the field was always 0. Added tla_staked to votePools — the toggle now populates and the popup staked-change is real. Also removed the APR rank-movement badge and APR "% vs prev": the only prior-APR source (apr-history apr_pct_avg) is RAW and uncapped (values into the trillions of %), which is not comparable to the live capped amplified APR — comparing them produced meaningless arrows (e.g. LUNA-FUEL "▲1" when it was already #1, and LUNA-WBTC "−67%"). APR tile is now labeled "live snapshot · click for drivers". Volume tile labeled "(USD)" with a tooltip noting volume % can move on token price, not just trading activity. Popup "why" line reworked to describe live drivers (votes up → APR up; staked up → APR down) instead of an unreliable vs-prev APR %.

#### Rev 3.10 — Overview tiles: APR rank arrows restored + Eris reconciliation
Verified depth (DEX liquidity) and staked (TLA deposits) against the live Eris UI — both match within ~1-3% across all pools (the page faithfully reflects the cron). APR (apr_amp) matches Eris closely for most pools; a few (USDC-EURe, USDC-USDT, LUNA-ampLUNA) diverge because the cron's emissions-based approx_apr_pct over/under-estimates for those specific pools — a cron-side calc issue, not a page bug. Restored the APR rank-movement arrows: the prior baseline (buildAprBaselineIndex) already applies the same $20K-min filter, 200% cap, and amp factor as the live APR, so current-vs-prior rank is comparable; also capped the amplified baseline at 200% so a base pinned at the cap doesn't read as 210%. Known residual: a stale below-threshold gauge variant (e.g. an old LUNA-WBTC) can still inflate the PRIOR ranking because apr-history merges variants under one name — a clean fix needs the apr-history cron to key by gauge_pool_id, not name.

#### Rev 3.11 — Ranking popup redesigned: prior -> current -> change table
The expand popup was a confusing soup of current values with parenthetical percentages. Rebuilt each pool's detail as a clear mini-table: every metric (APR, Votes, TLA staked, DEX liq, Volume) shows Prev epoch -> Current -> Change in labeled columns, with the change colored green/red. Metrics without a reliable prior are explicit: Bribes shows current with a "level" tag (no per-epoch history yet), Votion shows current->next projection. Fixed the bogus tiny DEX/Vol values (e.g. LUNA-WBTC "DEX $53.56"): the popup was picking up a dead below-threshold gauge variant's depth — now it dedups to the highest-VP variant per pool name, so DEX/staked/APR reflect the real active pool (~$130K, matching Eris). Kept the plain-language driver line (votes up lifts APR, staked up dilutes it).

#### Rev 3.12 — Popup: bribe token amounts + Votion zero-sum note
Bribe rows now show the TOKEN AMOUNT alongside USD (e.g. "100,000 CAPA ≈ $153") so it's clear the token quantity is the fixed thing a briber committed and the $ is just its current valuation — addressing the confusion where the same token amount shows a different USD each epoch as price moves. Added two explainer notes to the detail: Votion reallocates a fixed vote pool (a gain on one pool is offset by losses on others — which is why the top-ranked pools can all show positive Votion shifts while the losers sit lower in the list), and bribes are a fixed token amount whose $ value moves with token price. Confirmed the Votion data is roughly zero-sum (22.36M -> 22.52M total) with real negatives (USDC-EURe -7%, LUNA-FUEL -1%) — they just weren't visible in the top-5 view.

#### Rev 3.13 — Overview tiles: move toggle inline for uniform tile heights
Moved the DEX/TLA liquidity toggle from a separate row under the liquidity tile's title into the title row itself (shortened to "DEX"/"TLA"), so all three overview tile headers are the same height and the pool rows line up across tiles. Confirmed LUNA-FUEL's $3.43 volume is real (source astroportDayVolumeUsd is $0/$0/$4.13) — a high-APR pool with near-zero DEX trading, so its yield is emissions/bribes not fees. Bribe "prev epoch" is correctly blank (no per-epoch bribe history captured yet).

#### Rev 3.14 — Vote Breakdown waterfall: cleaner labels + grouped inactive
Cleaned up the waterfall. Each pool's name label was rendering twice (once outside the bar on the left, once inside if wide enough) and overlapping — now it's a single label, placed inside the bar when it's wide enough (>12% of scale) or just to the right of the bar end otherwise. Inactive pools (the many tiny below-threshold slivers — e.g. 11 of them in the project bucket totaling just $453K vs $23.2M active) are now grouped into ONE striped "Inactive pools (N) · X VP" bar by default, collapsing project from 19 rows to 8 active + 1 grouped. Click the divider/bar to expand into individual inactive rows, click again to collapse. Bucket totals still include all pools.

#### Rev 3.15 — Waterfall: fixed label gutter + aligned plot area
Restructured every waterfall row into a fixed 150px name column on the left + a plot area to the right where the bar lives. Previously labels floated at each bar's position and scattered across the chart (right-side ones ran off the edge); now all names sit in a clean left gutter and every bar plots from a consistent left edge. The scale ruler (0 / 6.25M / ... / 25M), its tick marks, and the background grid lines are all left-padded by the same 150px so the axis lines up with the bars. The grouped inactive bar uses the same gutter layout. Bar sub-segment percentages are now relative to the plot area, not the whole row, so name length no longer shifts the bars.

#### Rev 3.16 — Waterfall: color scheme + planned-epoch deltas
Recolored per request. DEX text: Astro = dark blue, Skeleton = orange, Single = gray. Bar segments: Votion = green, aDAO = light blue, Other = gray, selected member = yellow (legends in both the totals row and footer updated to match). Fixed the inactive-VP label spilling outside the tile — labels near the right edge now flip to render on the left side of the bar instead of past its end. On the Epoch 188 (Planned) view, each pool now shows its projected vote change at the bar's end as "+N / +%" (green) or "-N / -%" (red) — replacing the bare up/down arrow — using votion_now -> votion_next; verified real positives and negatives in the data (e.g. LUNA-FUEL -25,180/-1%, USDC-SOLID +16,146/+1%).

#### Rev 3.17 — Waterfall: planned-delta label placement
First (left-most) bar shows its planned +/- vote delta AFTER the bar end; every following bar shows it BEFORE the bar (ending at the bar's left edge). The first bar starts at x=0 with no room before it, while later bars sit further right with empty space to their left — so this keeps every label inside the tile and reads cleanly, instead of trailing off the right edge.

#### Rev 3.18 — Waterfall: real locked-in vs planned baseline
The two epoch views now mean what they should. "Epoch N (Locked-in)" uses the votes as they stood at the epoch boundary — fetched from the daily snapshot captured just before epochStartedAt (~Sunday 23:40 UTC), which carries the full per-pool breakdown (total + Votion split). "Epoch N+1 (Planned)" uses live votes now with Votion's slice moved to its optimized target, so each bar total shifts by Votion's reallocation (live - votion_now + votion_next) — bars actually grow/shrink as Votion moves VP between pools, zero-sum across the bucket. New loadLockedInBaseline() fetches the boundary daily (walking back up to 3 days if missing), dedups to the largest-VP variant per pool name, stores store.lockedInByPool, and re-renders the waterfall when it lands (falls back to live if the fetch fails). getPoolVP/getVotionVP route totals + Votion through the active view. NOTE for later cron work: aDAO and member PRIOR (locked) positions aren't in the daily snapshot — only Votion + totals are — so attributing the lock->now total change to users vs aDAO vs members needs a cron that captures their per-pool VP at the boundary.

#### Rev 3.19 — Waterfall: fix locked-in double-count + amber inactive
Fixed the locked-in view inflating the Stable bucket to 43M (LUNA-USDC showing a false ~18M jump). Root cause: the locked-in map was keyed by pool NAME only, but LUNA-USDC trades on BOTH Astroport (20M) and Skeleton Swap (2.5M) as separate real pools — keying by name merged them and both live rows grabbed the 20M baseline, double-counting. Now keyed by name+DEX so each variant matches its own baseline; Stable locked-in total back to ~24.3M (matches the planned-view 24.59M within real week-over-week drift). Also recolored the grouped/expanded inactive section amber (was gray, blended with the Other segments) so it stands out as a distinct group.

#### Rev 3.20 — Waterfall: clamp duplicated Votion onto dead gauge variants
Fixed LUNA-arbLUNA showing a fat green Votion bar in the INACTIVE group bigger than the pool itself. The cron attaches the same Votion allocation (1.36M) to both the active gauge (2.0M VP, real) and a dead below-threshold gauge (29K VP) that share the pair name but have different gauge_pool_ids — so the dead variant inherited a Votion bar ~46x its own size. Display fix: a pool segment can never exceed its own total, so Votion is clamped to the pool VP (and aDAO to the remainder) in both renderRow and the bucket totals. The dead variant now shows its real ~29K with no oversized bar; the active pool is unaffected. Real fix remains cron-side: key Votion/aDAO attachment by gauge_pool_id, not pool name.

#### Rev 3.21 — Data audit vs Eris/Votion + two central variant safeguards
Audited every pool's VP against the live Eris vote UI. Finding: almost all pools sit at a uniform -10% to -15% vs Eris, which is just VP decay/timing between our hourly capture and the Eris view — NOT a bug. The real bugs were the OUTLIERS, all the same root class (a gauge variant leaking into the active set), now fixed CENTRALLY in store.pools instead of per-renderer:
- Safeguard 1 (name+dex+bucket): within each group only the largest-VP variant stays active; dead voted_but_below_threshold leftovers are demoted to inactive. Caught LUNA-WBTC (18.5K dead beside 7.78M real) and LUNA-arbLUNA (29K dead beside 2.0M real).
- Safeguard 2 (gauge_pool_id across buckets): the same gauge must live in one bucket; keep the largest-VP instance, demote the rest. Caught USDC-USDT and USDC-USDt, each split bluechip+single for one gauge (our 281K+2.0M = Eris's 2.28M exactly).
Both log a console.warn when they fire, so if naming/bucketing breaks in future cron data it is visible rather than silent. Demoted variants collapse into the inactive group everywhere (tiles, popup, waterfall) automatically. Real upstream fix remains: key cron history + Votion/aDAO attachment by gauge_pool_id.

#### Rev 3.22 — Fix: demoted variants inflating the inactive group in locked-in view
The inactive group showed implausibly large VP (e.g. bluechip locked-in inactive = 10.7M). Cause: the variant safeguards demote a dead gauge to inactive, but getPoolVP looked the locked-in baseline up by name+dex — and a demoted variant shares the REAL pool's name+dex, so it inherited the real pool's big locked VP (LUNA-WBTC 7.69M, USDC-USDT 1.99M) instead of its own tiny VP. Fixed: demoted variants (_demotedVariant) never pull the locked-in baseline or the optimized-Votion path — they use their own small live VP/Votion. Bluechip inactive locked-in now ~300K (real below-threshold VP) instead of 10.7M. Note: the earlier screenshots showing a huge green inactive bar were from the build before the central safeguard + Votion clamp landed.

#### Rev 3.23 — Fix: locked-in active/inactive split must reflect boundary status
Root of the persistent large inactive total in the LOCKED-IN view: the active/inactive split was always based on CURRENT status, but the locked-in view shows boundary VP. A pool active when votes locked but below threshold now was put in the inactive group, yet its getPoolVP returned its big boundary baseline — so a real multi-M locked position showed up as inactive. Fix: in the locked-in view, a pool is active if it had a boundary baseline (lockedInByPool has its name+dex key); demoted duplicate variants stay inactive regardless; the planned/live view still uses current is_active. Bluechip locked-in now splits 11 active (~26M) + 10 inactive (~166K real below-threshold) instead of dumping multi-M into inactive.

#### Rev 3.24 — Pool Health & Exit Risk: made legible
The panel showed an unlabeled orange/grey magnitude bar (size of exposure) and an unlabeled red sparkline (4-epoch trend) — neither read as anything to a viewer. Removed the meaningless magnitude bar entirely (the right-hand $ figure already conveys size). Each row now pairs the mini-chart with a plain-language trend label ("falling 4 epochs straight", "down 12% over 4 epochs", "steady"), and the per-epoch dollar flow is the labeled headline on the right ("−$8.3K / out this epoch"). Rewrote the panel description to plainly state what it shows, and added an inline legend explaining the dot colours (healthy / watch / high exit risk) and that the mini-chart is staked over 4 epochs. Applied the same de-clutter to the member-mode view (dropped its unlabeled bar; its Stake%/Value$/threshold chips already carry the detail).

#### Rev 3.25 — Remove per-card STALE-SOON badge (footer health check only)
The per-widget staleness badge (the amber STALE-SOON box + coloured outline drawn on each data card) was visual clutter on the Overview. Removed the CSS that rendered it. Cron freshness is now surfaced in ONE place — the footer cron-health popover (dot + Crons healthy / stale soon / stale), which reads the cron status directly and is unaffected. The data-stale attribute is still set on widgets but no longer renders anything; the underlying freshness logic is untouched.

#### Rev 3.26 — Pool Health: LP composition strip + IL sensitivity
Added a per-pool composition view to the Pool Health panel. For two-sided pools it shows a value-split bar (each side's USD value and %) plus each side's token-AMOUNT drift this epoch. Because constant-product pools auto-rebalance to ~50/50 by value, the value split alone is rarely a signal — the token-amount drift is, so a divergence reads out as e.g. "net selling INJ into the pool" and parallel moves as "LPs adding/pulling both sides". Auto-expanded on flagged pools (where pressure matters), one tap away on calm ones. Single-asset pools show nothing (no second side).
Added an impermanent-loss SENSITIVITY line (not realized IL — epoch-over-epoch IL here is ~0 and would be noise). It shows what IL the 50/50 pool would incur if the two tokens diverged 25% / 50% / 2x (e.g. 50% -> -2.0%), clearly hypothetical and educational, with a tooltip explaining IL and that fees/incentives offset it. Pegged pairs (stable/stable, LST/LST) instead show "IL risk: minimal — pegged pair" since their ratio shouldn't diverge by design.

#### Rev 3.27 — Composition: guard against bad LST price feeds (the 62/38 bug)
Caught by inspection: LUNA-arbLUNA showed a 62/38 USD value split, impossible for a constant-product pool (which holds ~equal USD value per side by construction). Root cause: the price feed misprices liquid-staking tokens (arbLUNA shown at 2.9x LUNA; an LST should be ~1.0-1.1x and needs a redemption-rate oracle, not a market px), so the USD values are wrong. Same for LUNA-ampLUNA. Fix: when the computed value split is >8pts off 50/50, treat the price as suspect — draw the structural 50/50 bar and show chain-reported TOKEN AMOUNTS (trustworthy) instead of misleading dollar figures, with an amber note "USD split unverified — showing token amounts (LST price feed)". The token-amount drift and directional read are price-independent and stay valid. Real upstream fix: give the cron an LST redemption-rate oracle so USD values are correct.

#### Rev 3.28 — Threshold Watch: split into the two intended warnings
Reworked to clearly separate the two signals it was meant to give. (1) AT RISK OF FALLING: active pools within 1-2% of their bucket, now sorted FALLING-first (losing share epoch-over-epoch) with forward-looking framing — these may drop below 1% and stop earning if the slide continues. (2) CURRENTLY INACTIVE: a new standing list of pools below 1% right now (not earning) — the pull-liquidity candidates, sorted closest-to-1% first so the near-reactivation ones (e.g. LUNA-stLUNA at 0.96%) surface. The prior "dropped this epoch" and "earlier drops" history is kept below as event history. Verified non-overlapping. Note: a true "planned" projection isn't added because the Votion-optimized shift moves bucket % by <0.05pt — it would be false precision; the honest signal is current share + trend direction.

#### Rev 3.29 — Revert the LST priceSuspect guard — arbLUNA price was correct
On-chain check (arb UI: 1 arbLUNA = 2.92 LUNA, arbLUNA $0.1748, LUNA $0.0604) confirmed our arbLUNA price was RIGHT — arbLUNA redeems for ~2.92 LUNA, it is not 1:1. So the 62/38 value split is REAL: these LST pools use a weighted/concentrated curve, not a 50/50 constant product. My earlier priceSuspect guard (assuming all LSTs sit ~1:1) was wrong and hid correct data; reverted. Now we show the true USD split and, for an LST pair whose split is naturally uneven, a small note "uneven split is normal for an LST pair" so it does not look alarming. IL-minimal pegged handling unchanged.

#### Rev 3.30 — Composition: pool-type aware (concentrated vs xyk vs stable)
The 62/38 USD split on LUNA-arbLUNA looked broken but is correct: dex_subtype=concentrated. Astroport PCL (concentrated-liquidity) pools concentrate reserves around the current price, so the two sides hold different USD amounts BY DESIGN — not an imbalance. 22 of the pools are concentrated (incl. LUNA-USDC, LUNA-CAPA, LUNA-FUEL, LUNA-arbLUNA), 7 are xyk (true 50/50), 6 stable. The composition strip now reads dex_subtype (carried on store.pools as type, looked up by name+bucket since pool-status-history lacks it) and adapts: concentrated pools show "concentrated pool — uneven USD split is by design" and "IL: amplified vs a 50/50 pool" instead of the xy=k IL numbers (which don't apply to concentrated pools); xyk pools keep the 50/50 treatment + IL sensitivity figures; the "LPs pulling/adding both sides" read is suppressed for concentrated pools (reserves shift with price there, so it is not an LP add/remove signal). LST note retained for pegged pairs.

#### Rev 3.31 — New: "Is TLA Liquidity Growing?" panel + Threshold reorg + per-token flows
Three additions this round.
(1) New Overview panel answering the core health question: is the liquidity base growing or being drawn down. Total TLA-staked $ moves with token prices, so a $ decline can be pure price, not extraction. The panel leads with the REAL (price-neutral) trend — every epoch's reserves revalued at today's prices — alongside the headline $ trend, plus the token drivers of the change. Current read: $ down ~8% but REAL liquidity roughly flat (LUNA base growing from auto-compounding, offset by stablecoin withdrawals) — i.e. NOT extraction, the $ drop is mostly LUNA price. Honest framing: only 4 epochs of history, read direction not precision.
(2) Threshold Watch reorg: all four sub-sections now top-3 + expandable for a uniform, neat layout. AT RISK OF FALLING sorted falling-first; CURRENTLY INACTIVE now sorted by MOST TLA-STAKED first (biggest $ being eroded by take rate) showing staked $ and bucket %; DROPPED THIS EPOCH stays green when none, top-3 expandable when some; EARLIER DROPS top-3 expandable. Member filter wired through all sections. Fixed a literal-unicode bug where the section sublabels showed raw –2 / — (escapes in static HTML don't render) — now proper 1-2% and dashes.
(3) Pool Health flow cards (Net flow / Inflows / Outflows) are now tap-to-expand to per-token amounts (price-independent), so you can see which tokens entered/left — e.g. LUNA rising (auto-compounding) vs stablecoins leaving — that the USD figure hides.

#### Rev 3.32 — Liquidity Growth: recalibrate verdict + add size/migration context
The panel labeled a -1.9% real change over 4 epochs as "Drawing down" (red), which read as alarming for what is actually statistical flatness. Cross-checked against a fresh Eris dump: TLA holds ~$2.62M TVL, pays ~$1.01M/yr rewards (~38% blended APY), pools are actively earning and being MIGRATED to more efficient curve types (xyk -> pcl/concentrated) — a functioning, healthy protocol, not a declining one. Fixes: widened the stable band to +/-5% over the window (a couple-percent wiggle at a few-$M size is noise, not a trend) so the verdict now reads "Holding steady"; renamed "Drawing down" -> "Shrinking" and reserved it for real sustained moves; greyed the metric cards inside the band rather than red/green; added a size anchor ("TLA currently holds $X staked; a few percent of movement is normal at this size"); added a note that pools migrate between curve types, shifting where liquidity sits without leaving TLA. A panel that cries wolf is worse than none.

#### Rev 3.33 — Pool Health: group by bucket, top-3 each + expandable
The Pool Health & Exit Risk list was one long flat run of pools. Now grouped by bucket (STABLE / PROJECT / BLUECHIP / SINGLE), each showing its top 3 by severity-then-stake with a per-bucket "show N more" expander. Each bucket header carries pool count, total staked, and a flagged/all-clear chip. Candidate set widened from top-10-flat to every pool >= $5K staked (or flagged), so expansion reveals real depth instead of a hard 10-row cap. Verified the grouping handles legitimately-distinct same-name pools correctly (e.g. Astroport LUNA-USDC $598K concentrated vs SkeletonSwap LUNA-USDC $82K xyk — different DEX + gauge, both active, both shown).

#### Rev 3.34 — Member positions: earnings, APR/share, amplified tag + pool-type note everywhere
Member-selected Pool Health view gained three things from data we already capture but weren't surfacing: (1) an unclaimed-earnings banner (pending_rewards + pending_rebase + pending_bribes summed in USD, shown only when >0) so a member sees what's waiting to claim; (2) a per-position earning line — USD-weighted APR + their share of the pool (e.g. "68% APR · 5.3% of pool"); (3) an "auto-compounding" tag on amplified positions (is_amplified is reliable per-position, unlike the pool-wide ratio_type). Also added the pool-type explanation to BOTH views: the TLA-wide skew chip in the row header now names the reason ("62% LUNA · concentrated pool" / "· LST pair" / "· possible depeg" for stables) instead of a bare "by design", and member rows show a "concentrated pool — uneven split is by design" note. So a lopsided split is explained the moment it's seen, not one tap away in the composition strip.

#### Rev 3.35 — Explain mode — plain-language on-ramp for newcomers
Added a global "Explain mode" toggle (top-right of Overview). Off by default, so the dashboard looks exactly as it does now for experienced users. When on, a cyan "What this means" box appears under each major panel (Liquidity Growth, Pool Health, Threshold Watch, Vote Breakdown) explaining it in plain English with an analogy, no jargon — e.g. TLA as "a big shared pot of crypto", the threshold as "needs at least 1% of the votes or it stops earning". Pure-CSS show/hide via a body.explain-mode class (no per-panel JS); one tap reveals/hides all of them. First of the comprehensibility improvements (a member reported the tool was hard to understand); next candidates are consistent key-term tooltips and a short "how to read this" intro card.

#### Rev 3.36 — Replace explain mode with per-panel expandable descriptions; header tidy-ups
Reverted the global Explain-mode toggle in favour of per-panel descriptions: every major panel (Liquidity Growth, Pool Health, Threshold Watch, Vote Breakdown) now keeps a short always-visible one-line description plus a "learn more" link that expands the full plain-language explanation inline (toggleDesc + .desc-full). Cleaner than a global mode — the explanation lives next to the panel and you open only the one you need. Also: (1) fixed the header reflow — the member summary chip (VP/LP/active) now sits on its own line below the member selector instead of inline, so selecting a member no longer pushes the Live Epoch / Epoch Ends In group around; (2) removed the "Data Snapshot — Epoch NNN" element from the header entirely (freshness already lives in the footer cron-health popover and the data-notice), replaced with a quiet green "Live" dot.

#### Rev 3.37 — Footer rev label auto-tracks the changelog + last-updated date
The footer "Rev 2.2 · Changelog" label was hardcoded in the HTML, so it never moved when tla-log.md was pushed (the changelog MODAL fetches live from GitHub, but the little rev label did not). Fixed by deriving the label from the changelog itself: on page load (and on modal open) the page reads the top `## Rev X — YYYY-MM-DD` header from tla-log.md and sets the footer to "Rev X · Changelog · updated <Mon D, YYYY>". Now it auto-tracks every future push — no more hand-editing — and adds an at-a-glance last-updated date so visitors can see the dashboard is actively maintained.

#### Rev 3.38 — Changelog: jump-to-rev navigation
The changelog modal is long (Rev 3 alone has 37 sub-entries before you reach Rev 2/1). Added a "Jump to:" pill row at the top of the modal — one pill per major revision (Rev 3, 2.2, 2.1, 2.0, 1.15, 1.14), each scrolls straight to that revision. Built dynamically from the ## Rev headers in tla-log.md (anchor ids added to each rev header during markdown parse), so it auto-includes any future revisions with no extra work. Note: the footer label intentionally shows the MAJOR rev ("Rev 3") not the sub-rev — the X.x detail lives inside the changelog.

---

## Rev 2.2 — 2026-05-29

Pool health, capital-flow, and a full member mode built on top of the Rev 2.1 member overlay. All additions live in the rendering layer; the cron data layer is untouched except for two new history rollups that ride along with the existing `tla-snapshot` cron (see `cron-scripts/` and the new data files below).

### What changed

#### Pool Health & Capital Flow panel (new)
A watchlist of TLA's largest exposures, ranked by staked capital. Each pool shows a 4-epoch sparkline of its TLA stake, dollar flow this epoch, a health dot, and in/out/net summary cards. Comparisons are made within each pool's own `pool_address` series (never by name) so old/new pool migrations don't create phantom drops. Alarms are market-normalized — a pool is only flagged when it's draining materially faster than its bucket's median, so a broad market dip doesn't trip false alarms. Tiers: combined exit signal (depth + reserves + price all down hard), draining faster than peers, and sustained bleed (down every epoch and at least 15% cumulative). Reserve skew is shown only when a pair is meaningfully off 50/50.

#### Member mode (new) — the panels become personal when you pick a wallet
Selecting a member in the header now transforms two panels into a personal view, and reverts cleanly when deselected:

- **Pool Health becomes "Your positions & flow"**: your LP positions ranked by your own capital. Each row splits two honest signals side by side:
  - **Stake** — the change in your *real* LP units (vault/ampLP shares, falling back to LP tokens or underlying), which is the true deposit/withdraw signal. Auto-compounding and deposits grow it; only a genuine withdrawal shrinks it.
  - **Value** — the USD change, which also moves with token price. Price-driven moves are tagged `(price)` so a falling token can never look like you pulling capital.
  - Summary cards: **Value change** (USD, incl. price), **Stake added** (deposits + compounding), and **Stake reduced** (actual withdrawals only).
- **Threshold Watch becomes "Your at-risk pools"**: driven by your actual positions (`status` + `distance_from_threshold_pp`), not your votes — your pools that are near the 1% line, dropped this epoch, or already inactive.

Members holding the same pool under multiple stake configs are aggregated into one row. Member flow is epoch-over-epoch against the prior weekly archive; when that archive is missing, positions still list and the flow baseline is marked unavailable.

#### Threshold Watch rework
Rebuilt to be history-driven (keyed `name|bucket`): active danger-band pools (1–2% of bucket) most-at-risk first with an epoch-over-epoch trend, pools dropped this epoch, and an expandable list of drops over the last 4 epochs.

#### Leaderboards & APR history
Leaderboards now use true 4-epoch rolling averages with rank-movement badges and percentage deltas. New APR history (per-epoch rollup feeding a page consumer) adds an APR movement badge versus the last completed epoch.

#### Fixes
- **Non-PD bribes now captured, and PD bribes attributed correctly.** The Epoch Bribes breakdown showed $0.00 in every Phoenix Dir column and was missing bribes from other sources entirely. Two root causes: (1) `buildBribesIndex` hardcoded `pd_usd: 0` and dumped everything into "Other"; (2) the token-price resolver failed to price several bribe tokens, so many pools showed $0 total. Fixes: PD bribes are matched by normalized pool name + bucket/gauge and valued as `luna_per_epoch x live LUNA price` (the PD prop commits fixed LUNA per epoch over a multi-epoch range, so the prop's frozen USD is stale for later epochs); the price resolver now also matches native/IBC denoms stored under `prices.{source}.address`, maps known cross-chain IBC denoms (e.g. ASTRO) to their feed symbol, and pulls FUEL's price from its own `fuel-data_2026` repo (FUEL isn't in the network-and-prices feed). The result reconciles against Eris pool-by-pool: PD bribes land in the PD column, third-party bribes (e.g. Astroport volume kickbacks in ASTRO, CAPA incentives) land in Other, and pools with both show the split correctly (e.g. LUNA-ASTRO = PD LUNA + a small extra ASTRO bribe).
- **Phoenix Directive bribes now attributed correctly.** The Epoch Bribes breakdown showed $0.00 in every Phoenix Dir column — `buildBribesIndex` was hardcoding `pd_usd: 0` and dumping all on-chain bribes into "Other". PD bribes are now matched (by normalized pool name + bucket/gauge, so the same pool in two buckets is handled) and valued correctly: PD commits a fixed LUNA amount per epoch over a multi-epoch range, so the USD is recomputed as `luna_per_epoch x live LUNA price` rather than using the prop's frozen post-time USD. For epoch 187 this surfaces ~$510 of PD bribes across 10 pools that were previously mislabeled. (The PD master list lives in `tla-ext_json_storage/tla_pd_bribes.json`, maintained per PD governance prop.)
- **Single-asset pools** (ampCAPA, xASTRO) were mislabeled "Skeleton" in the Vote Breakdown waterfall — the dex sub-label was a binary `Astro` / `Skeleton`, so anything not Astroport fell through to Skeleton. They now correctly read "Single." (ampROAR-ROAR is a genuine Skeleton Swap pair and is unchanged.)
- Removed the orphaned "snapshot missed" popup.
- Removed the Skeleton Swap amber data-limitation banner.
- Fixed a false STALE outline caused by the bribes-history sporadic-data flag.

#### New data (cron side)
Two history files now accumulate, written once/day by rollups folded into the existing `tla-snapshot` cron (no new Render service):
- `apr-history.json` — per-epoch APR + staked averages per pool
- `pool-status-history.json` — per-epoch VP, bucket %, status, depth, staked, and reserves per pool (keyed `pool_address|bucket`)

### Verified working
- Pool Health watchlist flags the genuinely draining pools (LUNA-arbLUNA sustained bleed, LUNA-ATOM faster-than-peers) and stays quiet otherwise; net flow reconciles
- Single-asset pools read "Single"; Astroport reads "Astro"; Skeleton Swap reads "Skeleton"
- Member mode verified against live `adao-positions` data: the stake-vs-value split correctly separates real withdrawals from price — the only genuine withdrawal DAO-wide this epoch is one member trimming USDC-SOLID ~19%; every other apparent outflow was price movement
- Member at-risk view honestly shows "none near threshold" when all of a member's pools sit comfortably above the 1% line
- Both history rollups produce output byte-identical to hand computation against real daily archives

### Known limitations (acceptable)
- Member flow is epoch-over-epoch (positions update at the adao-positions cron cadence), and depends on the prior epoch's weekly archive existing
- For compounder vaults, manual deposits and auto-compounding both grow your unit count and can't be fully separated — hence the "Stake added: deposits + compounding" label

---

## Rev 2.1 — 2026-05-17

Member Data overlay feature + critical bribes resolver bug fix. Surgical additions to the rendering layer; cron data layer untouched (separate cron-side updates ship in the same session — see `cron-scripts/` repo for those).

### What changed

#### Member Data overlay (new feature)
Header dropdown selector — pick any aDAO member, the Overview tab visuals update with their data overlaid in amber. Pools / TLA Liquidity / aDAO tabs unchanged (member overlay is Overview-only by design).

When a member is selected:
- **VP Breakdown pie**: carves a member-colored slice out of "Other" — total VP unchanged
- **Vote Breakdown waterfall**: adds an amber member layer to each pool the member voted in. Bucket totals row gains a member chip; per-pool tooltip gains a member row
- **Threshold Watch**: filters to pools the member voted in. Header gains a "Filtered: {member}" badge. Empty states are member-aware ("None of {name}'s pools are at risk")
- **Member Stats Row**: 6 amber tiles below the global stat tiles — Astroport LPs, Skeleton LPs, Epoch Rewards, Epoch Bribes, Avg APR Non-Amp, Avg APR Amplified. Hidden by default; appears only when a member is selected
- Dropdown styling: dark color-scheme to fix invisible-text issue on some browsers; sorted by VP descending

#### Critical bug fix: bribes resolver
`resolveTokenPriceFromInfo()` was looking up cw20 token prices at `entry.address`. The actual `network-and-prices` schema nests the address at `entry.prices.{source}.address` (or under `prices.{source}.all_chains.{chain}.address` for multi-chain tokens). Any bribe paid in a cw20 token (CAPA, ROAR, etc.) silently priced as $0.

**Impact before fix**: Global Epoch Bribes tile showed ~$820. After fix: ~$1,300 (about 58% more accurate, more aligned with Eris). Member bribes tile correctly captures CAPA bribes (was 100% understated for members voting in LUNA-CAPA, ampCAPA).

Same resolver is used in `buildBribesIndex()` so this fix also corrects the per-pool bribe attribution used by waterfalls and ranking displays.

#### Pool lookup keying
All member-overlay lookups now use `gauge_pool_id` (truly unique, e.g. `cw20:terra1wdz...`) instead of `name+dex` (which can collide e.g. two `LUNA-WBTC|Astroport|BLUECHIP` entries with different gauge IDs). Required adding `gauge_pool_id` passthrough to both pool normalizers in the rendering layer (`votePools` normalizer ~line 3213, `normalizePoolData` ~line 2882).

Member-vote field is `pool_gauge_id`; snapshot field is `gauge_pool_id` — same values, different field names. Both are now handled.

#### Color scheme
- Member overlay color: amber (`#f59e0b`)
- "Other" VP: slate gray (`#64748b`) — was previously amber in waterfall, now consistent with pie chart slate
- Updated all 3 legends (waterfall totals, waterfall bottom, member tile row) for consistency

### Verified working
- Member dropdown populates from `adao-positions/current.json` members array
- Picking any member updates pie, waterfall, threshold watch, and member tile row in sync
- Switching to "All members" cleanly restores the global view
- Global Epoch Bribes tile climbs to ~$1,300 (verified against cron data)
- Member bribes correctly capture CAPA — tested against members voting in LUNA-CAPA pool
- All existing tabs and features continue to work (no regression in the ~7,000 lines of preserved rendering code)

### Known minor issues (acceptable for now)
- Skeleton Swap data labeled in Member Stats row but upstream source is frozen (see audit findings in `PROJECT_KNOWLEDGE.md`)
- Avg APR tiles still use TLA-staked-USD weighting (different from Eris); methodology fix tracked in `CHANGES_PENDING.md`

---

## Rev 2.0 — 2026-05-14

Major rebuild of the data layer to consume from the new TLA cron infrastructure (7 production crons writing to per-cron `*-data_2026` GitHub repos). Rendering code (~7,000 lines of charts, tables, modals, tabs) preserved intact — surgical surgery on data flow only.

### What changed
- **Removed** epoch/phase selector dropdown and snapshot date badge from the header. Data is now continuous (hourly updates) rather than per-epoch manual captures, so picking an epoch makes no sense. Live epoch + countdown remain.
- **Removed** all references to old per-epoch file paths (`tla-data-epoch-{N}-end.json`, `adao-snapshot_{N}_end.json`) which are no longer being written.
- **Added** new data fetch pipeline in `loadEpochData()`: parallel fetches from `tla-snapshot-data_2026`, `network-and-prices-data_2026`, `adao-positions-data_2026`, `bribes-data_2026`, `tla_ext_historical_2026.json`, and `tla_pd_bribes.json`. Falls through gracefully when individual sources unavailable.
- **Added** `buildLegacyDataShape()` transform function that maps the new continuous-data schema to the v3 store shape the existing renderers expect. Preserves all rendering code untouched.
- **Added** "Member Stats" tab link to the tab strip. Points to `dao-tla.html` (page not yet built — Pass 2 of the rebuild).
- **Fixed** aDAO tab now sources from treasury wallet data (`adao-positions/current.json` treasury field). At the TLA-wide level "aDAO" = treasury entity (single voter, 757K VP). Individual members live on the separate Member Stats page.
- **Fixed** TLA Total VP donut chart now shows mathematically truthful breakdown: 24.11M total (max bucket VP = Eris convention) split into Votion VP 6.90M (28.6%), aDAO/treasury VP 757K (3.14%), Other VP 16.46M (68.3%). Reconciled exactly against Votion's actual lockup data shown on votion.money.
- **Fixed** Liquidity DEX vs TLA Staked bar chart now populates correctly (uppercase bucket names matching renderer's expectations).
- **Fixed** Vote Breakdown Waterfall chart now renders all 4 bucket views (STABLE / PROJECT / BLUECHIP / SINGLE).
- **Fixed** Top by APR rankings excluded dust pools (TLA-staked < $20K) and capped at 200% to prevent illiquid pools with huge emissions/TVL ratios from dominating. Top entries now show realistic 70-80% APRs (LUNA-INJ, LUNA-FUEL, LUNA-CAPA, etc.) matching Eris.
- **Fixed** Avg APR weighted by TLA-staked-USD rather than depth-USD. ~40% Non-Amp / ~42% Amplified.

### Verified working
- All 6 tabs render
- Header tiles (Active pools 22 Astroport + 8 Skeleton, Epoch Rewards 339K LUNA / $22.7K, Epoch Bribes $841, Avg APR 40%)
- TLA Total VP donut with truthful breakdown
- Liquidity DEX vs TLA Staked bar chart (all 4 buckets)
- Vote Breakdown Waterfall (all 4 bucket views work)
- aDAO tab matches Eris UI within ±1%: Locked VP 757K, LP $6,669, rewards $453, bribes $443
- Top by APR rankings with realistic values

### Known minor issues (acceptable for now)
- Trend mini-charts on stat tiles will be empty until 2+ weekly snapshots accumulate (~4 weeks)
- Token grade scoring is a simplified stub — needs proper formula refinement
- Avg APR shows ~40% but Eris shows ~55% (different weighting methods, order of magnitude correct)
- Epoch number labeled as 184 instead of 185 — known off-by-one bug in cron output, dates correct. Fix planned across all crons. **[RESOLVED 2026-05-15 — see Rev 2.1 notes and cron README changelogs]**

---

## Rev 1.15 — 2026-05-08

Cleanup pass after first user review of the unified chrome rollout.

### What changed
- Cleaned up the page-specific header: removed the small aDAO logo, the "← Dashboard" backlink under it, and the "by The Alliance DAO •" subtitle. The "Terra Liquidity Alliance Tracker" title and the Eris TLA link remain. Epoch / phase selector and live epoch info on the right side are unchanged
- Cleaned up the page-specific footer: removed the "Updated: 4/26/2026" line (the changelog timestamp is the source of truth now), the "Built by: DeFi Patriot · DM for edits or errors" credit, and the "© 2025 Alliance DAO Community Project. Not affiliated with Terraform Labs..." copyright notice. The disclaimer block (Not Financial Advice / Data Accuracy / Third-Party Links / Use at Your Own Risk) and the Terra Liquidity Ecosystem links row remain
- Made the `last-updated` JS update null-safe since the element it targets was removed
- Fixed changelog modal — was fetching from `/main/logs/tla-log.md` (404), now fetches from `/main/tla-log.md`

---

## Rev 1.14 — 2026-05-08

Initial entry — page brought into the unified site chrome system.

### What changed
- Added unified site header (logo + 5-tab top nav + Terra logo)
- Added mobile bottom tab bar with TLA tab highlighted as active
- Added unified footer with Rev number + Changelog link (this changelog) — appended after the existing page footer (mission statement + ecosystem links preserved)
- Original page-specific controls preserved (epoch selector, phase selector, all charts and data tables)

### Earlier history (untracked)
TLA Stats has been the primary public face for Terra Liquidity Alliance data — voting share charts, lock data, epoch tracking, ve(3,3) analysis. The data pipeline depends on weekly Sunday 23:59 UTC snapshots captured manually via the TLA admin tool (automation is on the roadmap — see CHANGES_PENDING.md). Starting point of formal changelog tracking is rev 1.14.

Going forward, each meaningful change to this page will get its own entry here.
