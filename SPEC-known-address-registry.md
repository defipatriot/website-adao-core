# SPEC — Known-Address Registry (Phase 0 catalog addition)

**Status:** specified 2026-06-14, NOT built. A Phase 0 foundation piece.

## The problem
We keep RE-discovering the same identities ad hoc, in multiple places:
- `tla-participants` found two "anonymous whales" (5.5M + 1M VP) that are actually
  the arbLUNA-MAX and ampLUNA-MAX **Votion vaults**.
- `adao-positions` hardcodes the aDAO treasury/council wallet labels.
- `tla-locks` hardcodes the lock-asset symbol map.
- Ally treasuries, staking contracts (DAODAO/Enterprise), marketplace contracts
  (BBL/Atrium/Boost), Eris system contracts (escrow/gauge/compounder), polytone
  proxies — all are "known" but labeled (or not) in scattered, per-cron ways.

This violates the project's own **"one fact, one producer"** rule. A wallet that's
really a protocol contract shows up as a mystery whale on one page and a labeled
entity on another.

## The fix
A central **known-address registry** in the Phase 0 catalog: one canonical map
`address → { label, type, entity, source }`. Types e.g.: `treasury` (aDAO,
ally), `protocol_contract` (Votion vault, Eris escrow/gauge/compounder),
`staking_contract` (DAODAO, Enterprise), `marketplace` (BBL/Atrium/Boost),
`lst_token`, `lp_pool`, `ibc_proxy`, `lock_vault`. Every cron and page resolves
identity from THIS, instead of each maintaining its own labels.

## Why Phase 0
The catalog (`tla-registry`) is already the canonical "what exists" layer. Known
addresses are the identity half of that. Putting it here means:
- The "is this a whale or a known contract?" question is answered once, everywhere.
- New crons (Votion etc.) PRODUCE new known identities (the 6 vault addresses)
  that feed the registry — harvest from what we've already built.
- Pages can show "arbLUNA-MAX Votion vault" instead of `terra13aae…` automatically.

## Seed contents (harvest from existing crons + this session)
- **aDAO**: main wallet `terra1sffd4…m5vzm`, treasury/council wallets (from
  adao-positions constants), DAO voting contract `terra1c57ur…`.
- **Votion vaults** (6, code_id 3677): arbLUNA/ampLUNA × MAX/3mo/1wk (see
  SPEC-votion-capture.md for addresses), + their vdenom factory tokens.
- **Eris TLA system**: escrow `terra1uqhj8…`, gauge controller `terra1hfksrh…`,
  bribe manager, asset compounder, global-config.
- **Allies**: Pixel Lions core `terra1c690…`, Lion DAO core `terra1tkers…`, their
  voting modules + treasuries.
- **LSTs**: ampLUNA `terra1ecgaz…`, bLUNA `terra17aj4ty…`, arbLUNA `terra1se7rvue…`,
  stLUNA ibc denom.
- **Staking**: DAODAO + Enterprise NFT-staking contracts (from nft-inventory).
- **Marketplaces**: BBL/warlock, Atrium, Boost (from nft-inventory).
- **Known non-Votion contracts**: polytone-proxy `terra1nmnrc…` (ROAR/WHALE IBC
  bridge — NOT a TLA participant; flag to EXCLUDE from holder counts).

## Build approach
Could be a static curated JSON in the catalog repo (simplest, since these change
rarely) with a small verification cron that flags any large lock-holder /
top-staker NOT in the registry (so new unknown whales surface for labeling). The
verification angle keeps it honest — it's how we'd have caught the Votion vaults
automatically instead of by hand.

## Consumers
Every cron's holder/member output gains an `is_known_entity` + `entity_label`
field by joining against the registry. Pages stop showing raw addresses for known
entities. tla-participants' "top holders are contracts" note becomes automatic
labeling.
