# On-Chain Queries Reference — aDAO / TLA tooling

> **What this document is.** A complete catalog of every on-chain query relevant to TLA tooling: what the cron currently does, what Chainscope exposes by default for each contract, what custom queries we want to build into future tools, and — most importantly — the **human-readable label**, **input shape**, and **expected output** for each one.
>
> **Why it exists.** The end goal is a query tool that lets a TLA participant ask plain-English questions like *"What does this pool hold right now?"* or *"What's my voting power decay schedule?"* — and have it work better than visiting Chainscope and pasting raw JSON. To build that, every query needs a documented input contract, output contract, and what it powers downstream.
>
> **How to maintain.** When the cron adds a new query, or we discover a new contract method, or we want to add a future tool query, **append it here**. When inputs or outputs change shape upstream, update the example block. Treat this file as the single source of truth for *"what can we ask the chain about TLA, and what comes back."*

---

## How to read this doc

Each contract has its own section. Inside each section:

- **Address** — the on-chain address on Terra phoenix-1
- **Role** — what this contract does
- **Default queries (Chainscope)** — queries you can run with no input customization, that Chainscope's default "smart query" panel exposes
- **Custom queries** — queries we use that need input parameters (and what those parameters are)
- **Pagination notes** — for queries that return cursor-paginated lists
- **Gotchas** — known traps for callers

Each query block has the same shape:

```
Q-{Contract}-{Method}
  Human label:   "<plain-english version we'd surface in a UI>"
  Used by:       <which cron/tool needs this today>
  Input shape:   <raw JSON message>
  Inputs:        <what the user/caller fills in>
  Output shape:  <what comes back from the chain, abbreviated>
  Powers:        <what features/insights this query enables>
  Notes:         <gotchas, retry behavior, etc.>
```

---

## Calling pattern (LCD smart query)

All queries below are CosmWasm smart queries via Terra's public LCD:

```
GET https://terra-lcd.publicnode.com/cosmwasm/wasm/v1/contract/{addr}/smart/{base64(query_json)}
GET https://terra.publicnode.com/cosmwasm/wasm/v1/contract/{addr}/smart/{base64(query_json)}
```

The cron uses **two endpoints** for resilience — if one times out or rate-limits, retry on the other. See `Critical data-capture gotcha — Silent coercion hides cron query failures` in `PROJECT_KNOWLEDGE.md` for the canonical retry+backoff pattern.

For native bank queries (LUNA, IBC tokens, factory denoms) — that's the bank module, not smart queries:

```
GET /cosmos/bank/v1beta1/balances/{address}
GET /cosmos/bank/v1beta1/balances/{address}/by_denom?denom={url-encoded}
```

---

# Contract Registry

## 1. Eris Global Config (root discovery contract)

**Address:** `terra1hwxg6s732eparz3ys7sa4t5f64ngpd2w8syrca6z7ckv3fs9uqnsvrpcqa`
**Role:** Eris's root config contract. Holds the addresses of every other Eris-protocol-affiliated contract — gauge controller, voting escrow, asset compounder, all four asset-staking buckets, the bribe manager, etc. **This is where every catalog run starts** — fetch the address book once, derive everything else.

### Default queries (Chainscope)

#### Q-GlobalConfig-AllAddresses
- **Human label:** "What are all the Eris/TLA contract addresses?"
- **Used by:** `tla-registry` cron (Q1, always first)
- **Input shape:** `{ "all_addresses": {} }`
- **Inputs:** (none)
- **Output shape:**
  ```json
  {
    "asset_gauge": "terra1hfks...",
    "voting_escrow": "terra1uqhj...",
    "asset_compounder": "terra1zly9...",
    "bribe_manager": "terra1tuuw...",
    "asset_staking__bluechip": "terra14mmv...",
    "asset_staking__stable":   "terra1v399...",
    "asset_staking__project":  "terra1awq6...",
    "asset_staking__single":   "terra1qdz5..."
  }
  ```
- **Powers:** Every downstream query in the catalog. Without this, the cron can't even start.
- **Notes:** If this returns null, the whole run aborts (`global-config.all_addresses returned null — both LCDs failed`). No fallback — without the address book we don't know which other contracts to call.

---

## 2. TLA Asset Gauge (the voting heart of TLA)

**Address:** `terra1hfksrhchkmsj4qdq33wkksrslnfles6y2l77fmmzeep0xmq24l2smsd3lj`
**Role:** Records every member's vote allocations per epoch. Computes per-pool distribution percentages, holds gauge configuration (epoch length, vote-weight curves), publishes distribution snapshots.

### Default queries (Chainscope)

#### Q-AssetGauge-Distributions
- **Human label:** "What % of rewards does each pool get this epoch?"
- **Used by:** `tla-registry` cron (Q2)
- **Input shape:** `{ "distributions": {} }`
- **Inputs:** (none)
- **Output shape:**
  ```json
  {
    "distributions": [
      { "asset": {"native": "uluna"}, "pct": "0.00021..." },
      { "asset": {"cw20": "terra1wdz..."}, "pct": "0.0107..." }
    ]
  }
  ```
- **Powers:** Pool distribution percentages on the catalog page; powers reward emissions math in `tla-snapshot` cron; will power vote-intelligence scoring once we build it.
- **Notes:** Returns active gauge pools only. Below-threshold and dewhitelisted pools are NOT here — that's why we also need `whitelisted_asset_details` from each bucket's staking contract (see §5).

#### Q-AssetGauge-LastDistributionPeriod
- **Human label:** "What epoch are we in right now?"
- **Used by:** `tla-registry` cron (Q3) — provides canonical epoch
- **Input shape:** `{ "last_distribution_period": {} }`
- **Inputs:** (none)
- **Output shape:** integer (the 0-indexed epoch — add 1 to match canonical 1-indexed epoch_1-300_date.json)
- **Powers:** `canonicalEpoch` field in `current.json`; epoch alignment across crons.
- **Notes:** Returns 0-indexed; `canonicalEpoch = epochIndex + 1`. **Off-by-one trap** — was the source of the May 2026 epoch numbering fix; all crons now apply `+1` consistently. See PROJECT_KNOWLEDGE.md "Storage / data repos" notes.

#### Q-AssetGauge-Config
- **Human label:** "What are the gauge settings (epoch length, vote weights, etc.)?"
- **Used by:** `tla-registry` cron (Q4)
- **Input shape:** `{ "config": {} }`
- **Inputs:** (none)
- **Output shape:** `{ owner, epoch_length_s, vote_weight_curve, ... }`
- **Powers:** Diagnostic — confirms gauge contract hasn't been reconfigured.

### Custom queries (require inputs)

#### Q-AssetGauge-Votes
- **Human label:** "How did this member vote this epoch?"
- **Used by:** Not currently in `tla-registry` cron; **wanted by future Vote Intelligence tool**
- **Input shape:** `{ "votes": { "user": "terra1...", "period": <epoch> | null } }`
- **Inputs:** `user` (member's wallet), optional `period` (epoch number, default = current)
- **Output shape:**
  ```json
  {
    "votes": [
      { "asset": {"cw20": "terra1..."}, "bps": 5000 },
      { "asset": {"native": "uluna"}, "bps": 3000 },
      { "asset": {"cw20": "terra1..."}, "bps": 2000 }
    ],
    "total_voting_power": "..."
  }
  ```
- **Powers:** Per-member voting history; coalition detection; "is this member voting in their own interest" analysis.

#### Q-AssetGauge-GaugeInfo
- **Human label:** "How much VP voted for this specific pool, and what's its current rate?"
- **Used by:** Not in cron today; **wanted by Pool Detail view**
- **Input shape:** `{ "gauge_info": { "gauge_pool_id": "<cw20:...|native:...>", "period": <epoch> | null } }`
- **Inputs:** `gauge_pool_id` (the canonical pool ID from `pools[]`), optional `period`
- **Output shape:** `{ vp_voting_for_pool, current_rate, take_rate, ... }`
- **Powers:** Per-pool VP attribution; "who's voting for this pool" panel.

---

## 3. TLA Voting Escrow (vAMP / TLA Locks)

**Address:** `terra1uqhj8agyeaz8fu6mdggfuwr3lp32jlrx5hqag4jxexde92rzkamq3l62zg`
**Role:** CW721 NFT contract — every TLA lock is one NFT. Each token has a voting power amount, a lock start period, a lock end period, and an owner. **Total locks today: ~46 named members + treasury.**

### Default queries (Chainscope)

#### Q-VotingEscrow-NumTokens
- **Human label:** "How many TLA locks exist in total?"
- **Used by:** `tla-registry` cron (Q5) for sanity check
- **Input shape:** `{ "num_tokens": {} }`
- **Inputs:** (none)
- **Output shape:** `{ count: <integer> }`
- **Powers:** Sanity check on `all_tokens` paginated walk; sizing for "X members locked" displays.

### Custom queries

#### Q-VotingEscrow-AllTokens (paginated)
- **Human label:** "List every TLA lock NFT (oldest to newest)"
- **Used by:** `tla-registry` cron (paginated walk after Q5)
- **Input shape:** `{ "all_tokens": { "limit": <int>, "start_after": "<token_id>" | omitted } }`
- **Inputs:** `limit` (max 100 per page; we use 30), `start_after` (last token_id from previous page, or omitted on first page)
- **Output shape:** `{ tokens: ["1", "2", "3", ...] }` (array of token_id strings)
- **Powers:** Member enumeration; portfolio scans; "who owns this lock" lookup.
- **Pagination:** Walk until `tokens.length === 0`. Use `start_after` = last token_id of previous response.

#### Q-VotingEscrow-LockInfo
- **Human label:** "What does this specific TLA lock contain (VP, end date, owner)?"
- **Used by:** `tla-registry` cron per-token after enumeration
- **Input shape:** `{ "lock_info": { "token_id": "<id-as-string>" } }`
- **Inputs:** `token_id` (string; even though it's numeric, pass as string)
- **Output shape:**
  ```json
  {
    "owner": "terra1...",
    "voting_power": "796000000000",
    "amp_lp": "...",
    "start": { "period": 100 },
    "end":   { "period": 200 }
  }
  ```
- **Powers:** Per-lock VP, decay schedule, lock-end notifications, "your locks expire on X" reminders.
- **Notes:** voting_power is in vAMP base units (divide by 1e6 for human display).

#### Q-VotingEscrow-Tokens (owner's locks)
- **Human label:** "What locks does this wallet own?"
- **Used by:** Not in cron today; **wanted by Portfolio Tracker**
- **Input shape:** `{ "tokens": { "owner": "terra1...", "limit": 30, "start_after": "<token_id>" | omitted } }`
- **Inputs:** `owner` (wallet address), optional pagination
- **Output shape:** `{ tokens: ["123", "456"] }`
- **Powers:** "Show me MY locks" — without scanning every NFT in existence.

#### Q-VotingEscrow-UserInfo
- **Human label:** "What's this user's total voting power and lock summary?"
- **Used by:** Not in cron today; **wanted by Member Stats page**
- **Input shape:** `{ "user_info": { "user": "terra1..." } }`
- **Inputs:** `user` (wallet)
- **Output shape:** `{ total_voting_power, lock_count, locks: [...] }`
- **Powers:** Single-call member summary; alternative to enumerate-then-aggregate.

---

## 4. TLA Asset Compounder (the amplp factory)

**Address:** `terra1zly98gvcec54m3caxlqexce7rus6rzgplz7eketsdz7nh750h2rqvu8uzx`
**Role:** Eris's compounder factory. Creates and manages every amplp wrapper. **65 vaults today.** Each vault wraps an underlying LP token (or single-asset stake) and auto-compounds rewards back into more LP.

### Default queries (Chainscope)

#### Q-AssetCompounder-AssetConfigs
- **Human label:** "What amplps exist and what LPs do they wrap?"
- **Used by:** `tla-registry` cron (Q6) — drives `amplp_mappings`
- **Input shape:** `{ "asset_configs": {} }`
- **Inputs:** (none)
- **Output shape:**
  ```json
  [
    {
      "amp_denom": "factory/.../LUNA-USDC-ampLP",
      "asset_info": { "token": { "contract_addr": "terra1..." } },
      "gauge": "stable",
      "zasset_denom": "...",
      "reward_asset_info": { "cw20": "..." } | { "native": "..." },
      "fee": null | { "override_fee": "0.05" }
    }, ...
  ]
  ```
- **Powers:** The entire amplp layer of the catalog; lp_to_amplp index; "is this LP wrapped?" lookup.
- **Notes:** `asset_configs[].gauge` is the bucket NAME (`'single'`, `'stable'`, `'project'`, `'bluechip'`), NOT a contract address — see "Schema gotchas" in PROJECT_KNOWLEDGE.md.

### Future custom queries

#### Q-AssetCompounder-State (per-amplp)
- **Human label:** "What's the current exchange rate between this amplp and its underlying LP?"
- **Used by:** Not yet; **wanted for accurate amplp USD valuation**
- **Input shape:** `{ "state": { "amp_denom": "..." } }` (exact shape TBD when implemented)
- **Powers:** Removes the need for the LST-ratio hardcoded fallbacks (`bLUNA || 1.6048` etc.) — derive directly from compounder state.

---

## 5. ASSET_STAKING contracts (one per bucket — 4 total)

These four contracts hold the *complete* list of whitelisted, below-threshold, AND dewhitelisted LPs per bucket. **Where the catalog learns about pools** — the gauge `distributions` query (§2) only returns active ones.

| Bucket | Address |
|---|---|
| `bluechip` | `terra14mmvqn0kthw6sre75vku263lafn5655mkjdejqjedjga4cw0qx2qlf4arv` |
| `stable`   | `terra1v399cx9drllm70wxfsgvfe694tdsd9x96p9ha36w7muffe4znlusqswspq` |
| `project`  | `terra1awq6t7jfakg9wfjn40fk3wzwmd57mvrqtt3a39z9rmet7wdjj3ysgw3lpa` |
| `single`   | `terra1qdz5qgafx88kp5mf6m2tah8742g4u5g2cek0m3jrgssexexk7g4qw6e23k` |

### Default queries (Chainscope)

#### Q-AssetStaking-WhitelistedAssetDetails ⭐ critical
- **Human label:** "Every LP this bucket has ever whitelisted — including inactive and dewhitelisted ones — with take-rate metadata"
- **Used by:** `tla-registry` cron (per bucket)
- **Input shape:** `{ "whitelisted_asset_details": {} }`
- **Inputs:** (none)
- **Output shape:**
  ```json
  [
    {
      "asset": {"cw20": "terra1..."} | {"native": "..."},
      "whitelisted": true | false,
      "yearly_take_rate": "0.1",
      "taken": "1234567890",
      "harvested": "1234567890",
      "last_taken_s": "1717000000",
      "stake_config": "default" | { ... custom mechanism ... }
    }, ...
  ]
  ```
- **Powers:** The full catalog scope (active + below_threshold + dewhitelisted); per-LP take-rate display; "this LP was dewhitelisted but still has X take-rate exposure" warnings.
- **Notes:** Use `whitelisted_asset_details` NOT `whitelisted_assets` — the latter only returns *currently active* entries. This was a key fix in early Phase 0.

#### Q-AssetStaking-Config
- **Human label:** "What's this bucket's staking config (threshold, take rate floor, etc.)?"
- **Input shape:** `{ "config": {} }`
- **Inputs:** (none)
- **Output shape:** `{ activation_threshold_pct, default_take_rate, ... }`
- **Powers:** "Why is this LP below threshold?" explainers — show the threshold, the LP's vote share, and the gap.

### Custom queries

#### Q-AssetStaking-ListStakers (paginated)
- **Human label:** "Who has tokens staked in this bucket?"
- **Used by:** `adao-positions` cron (different cron, but uses same shape)
- **Input shape:** `{ "list_stakers": { "limit": <int>, "start_after": "<addr>" | omitted } }`
- **Inputs:** `limit` (we use 30), `start_after` (last address from previous page)
- **Output shape:** `{ stakers: [{ address, amount, ... }, ...] }`
- **Powers:** Member-level position discovery; "who's the biggest LP in this bucket" rankings.

---

## 6. Pair contracts (LP-pool foundation)

Every TLA pool has a pair contract behind it. **Three architecture variants** appear in the catalog:

- **Astroport xyk** (`astroport-pair`) — constant product
- **Astroport stable** (`astroport-pair-stable`) — stableswap curve
- **Astroport concentrated** (`astroport-pair-concentrated`) — CL ranges
- **White Whale pool** (`white_whale-pool` v1.3.8) — operated as Skeleton Swap by Backbone Labs after WW shut down. Pool contracts inherited from WW.

Same query interface across all variants (different code, same WASM smart-query API).

### Default queries (Chainscope)

#### Q-Pair-Pair ⭐ critical
- **Human label:** "What two assets does this pool hold, and what's the LP token?"
- **Used by:** `tla-registry` cron (resolves every LP's underlyings — see §10)
- **Input shape:** `{ "pair": {} }`
- **Inputs:** (none)
- **Output shape:**
  ```json
  {
    "asset_infos": [
      { "native_token": { "denom": "ibc/2739..." } } | { "token": { "contract_addr": "terra1..." } },
      { "native_token": { "denom": "uluna" } }
    ],
    "contract_addr": "terra1...",
    "liquidity_token": { "token": { "contract_addr": "terra1..." } } | <missing for factory-LP pools>,
    "asset_decimals": [6, 6],
    "pair_type": "constant_product" | "stable" | "concentrated"
  }
  ```
- **Powers:** **Ground truth** for what a pool holds. Used to verify SS's mislabeled API denoms; used to detect cross-DEX token-identity claims (17/17 verification of TLA same-named pairs); foundation for variant-trap warnings.
- **Notes:** **TRUST THIS OVER ANY API.** SS's API JSON claims `ibc/C3988DBA...` for ATOM in some pools; `pair{}` query on those same contracts returns the standard `ibc/27394FB0...`. The contract knows what it actually holds; the API just labels it weirdly.

#### Q-Pair-Pool
- **Human label:** "How much of each token does this pool currently hold, and how many LP shares exist?"
- **Used by:** Not in `tla-registry` (would bloat the snapshot); **wanted for live valuation in future Detail View**
- **Input shape:** `{ "pool": {} }`
- **Inputs:** (none)
- **Output shape:**
  ```json
  {
    "assets": [
      { "info": {"native_token": {"denom": "..."}}, "amount": "..." },
      { "info": {"native_token": {"denom": "uluna"}}, "amount": "..." }
    ],
    "total_share": "..."
  }
  ```
- **Powers:** Live per-pool depth; per-LP token USD value; impermanent loss math; slippage estimates.

#### Q-Pair-Config
- **Human label:** "What fees does this pool charge, and is it open for swaps/deposits/withdrawals?"
- **Used by:** Not in `tla-registry` today; **wanted for pool detail view**
- **Input shape:** `{ "config": {} }`
- **Inputs:** (none)
- **Output shape (white_whale-pool):**
  ```json
  {
    "owner": "terra1...",
    "fee_collector_addr": "terra1...",
    "pool_fees": {
      "protocol_fee": { "share": "0.001" },
      "swap_fee": { "share": "0.002" },
      "burn_fee": { "share": "0" }
    },
    "feature_toggle": {
      "withdrawals_enabled": true,
      "deposits_enabled": true,
      "swaps_enabled": true
    }
  }
  ```
- **Output shape (astroport-pair):** different shape — `params` is base64-encoded; need a parse step.
- **Powers:** Surface "this pool charges N% fee", "deposits temporarily paused" notices, fee breakdown for fee-yield calcs.
- **Notes:** White Whale and Astroport return different shapes. The catalog should normalize for the page.

#### Q-Pair-Contract (the version sniffer)
- **Human label:** "What kind of pool is this — Astroport or White Whale, and what version?"
- **Used by:** Not currently in cron; **wanted for pool architecture surfacing**
- **Input shape:** `{ "contract_version": {} }` — or read the `contract_info` smart query
- **Inputs:** (none)
- **Output shape:** `{ "contract": "white_whale-pool" | "astroport-pair" | "astroport-pair-stable" | "astroport-pair-concentrated", "version": "1.3.8" | "..." }`
- **Powers:** "(S)" suffix explainability — show users "Pool architecture: White Whale v1.3.8 (operated by Backbone Labs as Skeleton Swap)" so they understand why some pools look different.

### Custom queries (Astroport-specific, future use)

#### Q-AstroportPair-LpPrice
- **Human label:** "What's one LP token worth in USD right now?"
- **Input shape:** `{ "lp_price": {} }`
- **Output shape:** decimal string in stableswap denom
- **Powers:** Direct LP USD valuation; cross-check against derived `pool / total_share × price`.

#### Q-AstroportPair-Share
- **Human label:** "If I redeem this many LP tokens, what do I get back?"
- **Input shape:** `{ "share": { "amount": "<lp-units>" } }`
- **Inputs:** `amount` (LP units)
- **Output shape:** `[ { info, amount }, { info, amount } ]`
- **Powers:** "Withdraw preview" tool; member-level position valuation.

#### Q-AstroportPair-Simulation
- **Human label:** "If I swap X of token A, how much B do I get (incl. slippage)?"
- **Input shape:** `{ "simulation": { "offer_asset": { "info": {...}, "amount": "..." } } }`
- **Inputs:** `offer_asset` (full info+amount object)
- **Output shape:** `{ return_amount, spread_amount, commission_amount }`
- **Powers:** Slippage calculator; "is this pool deep enough for your trade size" warnings.

#### Q-AstroportPair-ReverseSimulation
- **Human label:** "How much A do I need to spend to get exactly Y of B?"
- **Input shape:** `{ "reverse_simulation": { "ask_asset": { "info": {...}, "amount": "..." } } }`

### Custom queries (White Whale pool-specific)

#### Q-WhiteWhalePool-ProtocolFees
- **Human label:** "How much in fees has this pool collected (uncollected)?"
- **Input shape:** `{ "protocol_fees": {} }`
- **Output shape:** `{ fees: [ { info, amount }, { info, amount } ] }`
- **Powers:** Real fee revenue tracking — not gameable like emissions APR.

#### Q-WhiteWhalePool-BurnedFees
- **Human label:** "How much has this pool burned of each token?"
- **Input shape:** `{ "burned_fees": {} }`
- **Output shape:** same shape as protocol_fees
- **Powers:** Deflationary token accounting.

---

## 7. CW20 Token Standard (every Eris-side LP, project token, etc.)

Standard CosmWasm token interface. Applies to: cw20 LP tokens, ROAR, CAPA, SOLID, ampLUNA, ampROAR, ALLY, ADAO, and most project tokens. **Not applicable to** native tokens (`uluna`), IBC tokens (`ibc/...`), or factory denoms (`factory/.../...`).

### Default queries (Chainscope)

#### Q-CW20-TokenInfo
- **Human label:** "What's this token's name, symbol, decimals, supply?"
- **Input shape:** `{ "token_info": {} }`
- **Output shape:** `{ name, symbol, decimals, total_supply }`
- **Powers:** Token identity verification; used by chain-registry, Eris, CG indices as the canonical source.

#### Q-CW20-Minter
- **Human label:** "Who's allowed to mint this token?"
- **Used by:** `tla-registry` cron — used to resolve `cw20 LP → pair contract` (the minter of an LP token IS the pair contract that created it)
- **Input shape:** `{ "minter": {} }`
- **Output shape:** `{ minter: "terra1...", cap: null | "..." }`
- **Powers:** LP→pair resolution path. **Critical** for the catalog — without this, we couldn't get from "this cw20 is in TLA's gauge" to "what pair contract does it belong to → what underlyings does it have."

### Custom queries

#### Q-CW20-Balance
- **Human label:** "How much of this token does this wallet hold?"
- **Input shape:** `{ "balance": { "address": "terra1..." } }`
- **Inputs:** `address`
- **Output shape:** `{ balance: "..." }`
- **Powers:** Per-wallet position tracking; used by `adao-positions` cron and `aDAOLive` library.

#### Q-CW20-AllAccounts (paginated)
- **Human label:** "Who holds this token? (every holder)"
- **Input shape:** `{ "all_accounts": { "limit": 30, "start_after": "terra1..." } }`
- **Powers:** Whale-concentration analysis; Gini coefficient for LP Health Scoring.

---

## 8. Native bank queries (uluna, ibc/..., factory/.../...)

For non-cw20 tokens — LUNA itself, IBC bridged tokens, factory denoms (Eris's amplps, BBL's bWHALE, etc.). These don't have smart-query contracts; they're tracked by the chain's bank module.

### Bank balance lookups (REST, not smart query)

#### Q-Bank-AllBalances
- **Human label:** "What does this wallet hold of every native + IBC + factory token?"
- **Endpoint:** `GET /cosmos/bank/v1beta1/balances/{address}`
- **Inputs:** address
- **Output:** `{ balances: [ { denom, amount } ], pagination }`
- **Powers:** Treasury composition; "show me everything in this wallet."

#### Q-Bank-ByDenom
- **Human label:** "How much of this specific denom does this wallet hold?"
- **Endpoint:** `GET /cosmos/bank/v1beta1/balances/{address}/by_denom?denom={url-encoded denom}`
- **Inputs:** address, denom (url-encoded — `ibc/...` and `factory/.../...` need escaping)
- **Output:** `{ balance: { denom, amount } }`
- **Powers:** Targeted balance check; faster than scanning all balances when you know the denom.

#### Q-Bank-Supply
- **Human label:** "What's the total supply of this token across the chain?"
- **Endpoint:** `GET /cosmos/bank/v1beta1/supply/by_denom?denom=...`
- **Powers:** Native-token tokenomics tracking; cross-check against cw20 `token_info.total_supply` for migrated tokens.

---

## 9. TLA Incentive Manager (bribes contract)

**Address:** `terra1tuuwm8yrj54qeg0c8xu00aha9ryatyhtczq8qq2q8tntuw0auzas9037wh`
**Role:** Holds all the bribes paid to TLA voters; tracks which member can claim which bribes per epoch.

### Default queries (Chainscope)

#### Q-IncentiveManager-Config
- **Human label:** "What are the bribe-manager settings?"
- **Input shape:** `{ "config": {} }`
- **Output shape:** owner, fees, allowed bribe tokens, etc.

### Custom queries (used by `bribes-history` cron)

#### Q-IncentiveManager-UserClaimable
- **Human label:** "What bribes can this member claim across all open epochs?"
- **Input shape:** `{ "user_claimable": { "user": "terra1..." } }`
- **Inputs:** user (member's wallet)
- **Output shape:**
  ```json
  {
    "start": <epoch>,
    "end": <epoch>,
    "buckets": [
      { "bucket": "...", "claimables": [...] }, ...
    ]
  }
  ```
- **Powers:** "How much in pending bribes do I have?" tile; member-bribe summaries.
- **Notes:** Returns `{ start, end, buckets }` — NOT an array. Iterating directly throws (see `Schema gotchas` in PROJECT_KNOWLEDGE.md).

#### Q-IncentiveManager-EpochBribes
- **Human label:** "What bribes were paid into this specific epoch (across all pools)?"
- **Input shape:** `{ "epoch_bribes": { "period": <epoch> } }` (verify exact shape against bribes-history cron)
- **Powers:** Bribe history per epoch — fuels the Global Epoch Bribes tile.

---

## 10. Skeleton Swap / Backbone Labs API (off-chain reference; not a query, but indexed by cron)

**Endpoint:** `https://dex.warlock.backbonelabs.io/api/pools/phoenix-1`
**Role:** Lists SS-frontend pools (which all run on inherited White Whale pool contracts).

### Critical gotcha

SS's API JSON has **misleading "denom" labels** for some IBC tokens. Returns things like `"ATOM on Dungeon"` at `ibc/C3988DBA...` for pools that — when queried via `pair{}` on the actual contract — hold standard `ibc/27394FB0...` ATOM.

**Always verify SS's API claims with `Q-Pair-Pair` (§6) before trusting the denom field.** The catalog uses on-chain pair{} queries as ground truth.

User-verified empirically: standard-IBC ATOM deposits into both Astroport AND SS LUNA-ATOM pools. There's one ATOM on Terra, just one weird API label.

---

# Future Query Tool — UX Sketch

The current Chainscope flow is:

1. User goes to Chainscope
2. Pastes contract address
3. Picks a method from a dropdown of raw method names (`pair`, `pool`, `config`)
4. Fills in JSON inputs by hand
5. Hits Query
6. Gets back raw JSON they have to interpret

The aDAO tool replaces this with:

1. User picks a **topic** ("LPs", "My voting power", "Pool fees")
2. The tool shows a list of **human questions** that match (e.g., "What does this pool hold right now?", "How much VP do I have locked?")
3. User picks one; the tool shows the inputs as **labeled form fields** (not JSON), with placeholders, validation, and defaults
4. The tool runs the query under the hood
5. Returns a **formatted answer** with units, USD valuations, and links to related queries

Examples of human-readable wrappers:

| Chainscope flow | Tool flow |
|---|---|
| `terra1aa8nu... → pair{} → JSON response with asset_infos` | "What does the ATOM-dATOM SS pool hold? → ATOM (53M) + dATOM (12.4M), as factory denom LP" |
| `terra1uqhj... → lock_info{ token_id: "600" } → JSON` | "What's lock #600? → 796,000 vAMP owned by terra1sffd4..., ends epoch 250 (decay rate: X/epoch)" |
| `terra1tuuw... → user_claimable{ user: "terra1sffd4..." } → nested buckets JSON` | "What's pending for the aDAO treasury wallet? → $443.13 across 3 buckets (CAPA $200, LUNA $150, ROAR $93). Claim before epoch 195." |

**Build prerequisites** (in roughly this order):
1. Every contract's queries documented in this file — *done as of 2026-06-02*
2. A query runner that does retry + dual-endpoint fallback — *exists in `tla-registry` cron's `queryContract`*
3. A formatter library: takes a raw response + schema → human-readable string
4. The UI for topic → question → form-fields → answer

---

# Query Patterns Library (reusable)

### Pattern A — Pagination cursor walk

```js
let cursor = undefined;
const all = [];
while (true) {
  const msg = { all_tokens: { limit: 30, ...(cursor ? { start_after: cursor } : {}) } };
  const r = await queryContract(addr, msg);
  const batch = r?.tokens || [];
  if (batch.length === 0) break;
  all.push(...batch);
  cursor = batch[batch.length - 1];
  if (batch.length < 30) break;  // last page
}
```

Applies to: `all_tokens`, `all_accounts`, `list_stakers`.

### Pattern B — Retry with dual-LCD fallback

```js
async function queryContract(addr, query, label, retries = 2) {
  const qb = base64(JSON.stringify(query));
  const endpoints = ['https://terra-lcd.publicnode.com', 'https://terra.publicnode.com'];
  for (let attempt = 0; attempt <= retries; attempt++) {
    for (const base of endpoints) {
      try {
        const r = await fetch(`${base}/cosmwasm/wasm/v1/contract/${addr}/smart/${qb}`);
        if (r.ok) return (await r.json()).data;
      } catch {}
    }
    await sleep(200 + Math.random() * 300);  // jittered backoff
  }
  return null;  // CRITICAL: caller must distinguish null from empty
}
```

### Pattern C — LP → pair → underlyings resolution

```js
// Given an LP token address, find what it holds
async function resolveLpUnderlyings(lpAddr) {
  let pairAddr;
  if (lpAddr.startsWith('factory/')) {
    // Native factory LP: pair is the first segment of the denom path
    pairAddr = lpAddr.split('/')[1];
  } else {
    // CW20 LP: minter query
    const minterResp = await queryContract(lpAddr, { minter: {} });
    pairAddr = minterResp?.minter || minterResp?.address;
  }
  if (!pairAddr) return null;
  const pairResp = await queryContract(pairAddr, { pair: {} });
  if (!pairResp?.asset_infos) return null;
  return pairResp.asset_infos.map(info =>
    info.token?.contract_addr || info.native_token?.denom
  );
}
```

### Pattern D — Cross-DEX trust verification

When two sources disagree about what a pool holds (e.g., SS API says ATOM at `ibc/C3988DBA...`, our pair{} query says `ibc/27394FB0...`), **trust the chain**. The pair contract knows what it actually accepts — APIs can have stale or mislabeled metadata.

```js
// Run pair{} against the pool's contract directly; use this as truth
const onChainUnderlyings = await resolveLpUnderlyings(lpAddr);
// API claims become a "source coverage" claim, not a fact
const ssApiClaim = ssApiData.tokens?.map(t => t.denom);
// Surface disagreement explicitly; don't silently pick a winner
if (!arraysEqual(onChainUnderlyings, ssApiClaim)) {
  warnings.push({ pool: lpAddr, api_claim: ssApiClaim, on_chain: onChainUnderlyings });
}
```

---

# Maintenance log

| Date | Change |
|---|---|
| 2026-06-02 | Initial document created at the close of catalog audit. All current `tla-registry` cron queries inventoried, plus the wishlist queries for Vote Intelligence, Portfolio Tracker, Pool Detail View, and the future query tool. SS API gotcha documented after empirical deposit test + on-chain pair{} verification on 17 pairs. |
