# NOTE — arbLUNA pricing: RESOLVED (the hub-ratio was right) — updated 2026-06-14

## TL;DR (corrected)
Earlier today this note claimed our hub-ratio over-priced arbLUNA ~14% vs market.
**That was WRONG — confirmed against CoinGecko ground truth.** The hub-ratio is
CORRECT. The misleading low number came from Votion's UI + a thin Astroport pool,
not from our feed. **No fix needed to final prices.** We kept hub-ratio as final.

## The ground-truth check that settled it
- CoinGecko **arbLUNA: $0.1523**; CoinGecko **LUNA: $0.05241**; Eris ratio **2.9527**.
- Hub-ratio math: $0.05241 × 2.9527 = **$0.1548** → vs CoinGecko $0.1523 = **1.6%**
  (just price-tick timing). ✓ Hub-ratio matches the real market aggregator.
- The "market" price our code pulled from a single Astroport pool was **$0.1361**
  — about **11–14% BELOW** real market. That thin/stale pool was the OUTLIER, not
  the hub ratio. Votion's UI showed a similarly low number; both were low, and we
  mistook them for truth earlier.

## What we changed (and didn't)
- **Kept hub-ratio as `final_price_usd`** for all calculated LSTs — it's the most
  accurate AND most robust (continuously updated from the staking contract,
  rarely breaks). No downstream USD values shift. Nothing breaks.
- Added a **weak cross-check**: we still read the single-pool price, surface it as
  `pool_market_price_usd`, compute `price_divergence_pct`, and FLAG (not flip) when
  the gap exceeds 10% — a data-quality signal that the pool may be thin/stale OR
  the asset genuinely depegged. A thin pool can NEVER override the proven hub price.

## The real lesson (this is the keeper)
**When two sources disagree, the "market" reading is not automatically right — it
can be the broken one.** A thin or stale single-pool price will lie. The robust
move is: hub-ratio primary (proven), CoinGecko as the trustworthy market
cross-check, single-pool as a weak signal only. Validate against an independent
aggregator (CoinGecko) before ever flipping a price.

## Still worth doing
- **ampCAPA / ampROAR**: still only hub-priced, no cross-check configured. Add a
  CoinGecko/market reference where one exists to confirm they're not genuinely
  depegged (amp-strategy tokens *could* depeg, unlike clean LSTs). Until then,
  hub-ratio stands as best-available.
- Consider wiring CoinGecko directly as the cross-check source (more reliable than
  a single Astroport pool) for the divergence guard.
