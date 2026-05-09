# Changes Pending — aDAO-links-site

> Rolling list of identified changes for the next batch upload. Newest items at the top of the active section. Add as we identify, check off as completed.

---

## 🔥 Active / next round

### Tune `fetchDaoTlaVp` attribute names (first-deploy verification)
The Rev 3.31 live VP fetcher tries multiple attribute names since the CW721 schema isn't documented (`amount`/`locked_amount`/`balance`, `multiplier`/`power_factor`/`factor`, `asset`/`token`/`lock_asset`, `duration`/`lock_duration`). On first deploy the console will log the raw attribute list if extraction fails — adjust the helper to the real format if needed.

### Optional: fully-live DAO Broken/Held NFTs
Rev 3.31 hardcodes `DAO_BROKEN_HELD_COUNT = 1000` (the documented governance count from Props 64-69), matching the admin tool's stated approach. Could be replaced with `nfts.filter(n => n.broken && DAO_WALLETS.includes(n.owner)).length` if the user provides the two missing liquidity wallet addresses (suffixes `...8ywv` and `...417v`; the third Enterprise wallet `terra1nn7yrgjzj6zvle7ms9vlpg4cj3kaxjls4g6ugw` is already in the code).

### Dashboard live queries — additional resilience (follow-up to Rev 3.27 / 3.29 / 3.30 / 3.31)
Rev 3.27 stopped the cascade crash. Rev 3.29 → 3.30 → 3.31 progressively replaced snapshot-dependent tiles with live queries where possible. Remaining resilience items (relevant only for the tiles that still legitimately need on-chain data):

- [ ] **Per-fetch try/catch isolation in `fetchLiveOnChainData`.** Right now all treasury balance fetches share one outer `try`. One LCD 500 still produces null for downstream calcs (which now display `—` instead of crashing — better, but not great). Each fetch should be its own isolated promise so a single asset failure only blanks the affected tile.
- [ ] **Per-fetch timeout in `fetchLiveOnChainData`.** Today's load took 14.67s — almost all of it in the Terra LCD `Promise.all`. A 5–8 second timeout per request via AbortController would cap the worst case. Pattern already used in Rev 3.29 / 3.31.
- [ ] **Fallback LCD endpoint on 5xx.** Both `terra.publicnode.com` and `terra-lcd.publicnode.com` returned 500s during the May 8 outage. On 5xx, retry against a different public Terra LCD. Need to research which alternates are reliable and CORS-permitted in-browser.
- [ ] **Apply Rev 3.29 parallel pattern to `fetchTlaData()` (TLA Stats page).** Currently walks epochs sequentially — slow on stale fallback. Easy port of the same `Promise.allSettled` + `AbortController` pattern. Cache could be shared between the two functions for further speedup.
- [ ] **Deduplicate `fmt` and `safeLocale` helpers.** Rev 3.26 added `fmt` inside the try block, Rev 3.27 added `safeLocale` outside. They overlap. Fold `fmt` callsites onto `safeLocale` and drop `fmt`.

### Mobile layout polish — non-index pages in PWA
Tested-environments table in `PROJECT_KNOWLEDGE.md` confirms only `index.html` is verified working on iPhone 16 PWA. Every other page is unverified for PWA mobile. This is iterative screenshot-driven work; one page per pass.

- [ ] NFT Explorer (`nft-explorer-index.html`) — PWA mobile review
- [ ] aDAO Lore (`adao-lore.html`) — PWA mobile review
- [ ] TLA Stats (`tla-stats.html`) — PWA mobile review
- [ ] DAO (`dao.html`) — PWA mobile review
- [ ] ALLY Rewards (`ally.html`) — PWA mobile review
- [ ] Tutorials, Tools, Rarity Info, NFT Releases, Official Links, Alliances — PWA mobile review (batch)
- [ ] DAO Treasury, DAO TLA Deposits, Fuel Tool, ampCapa Tool, TLA Docs — PWA mobile review (batch)

### NFT ranking system rework — align with BBL
The current aDAO ranking system diverged from Backbone Labs' approach. User wants to revisit and align — needs design discussion before any code changes.

- [ ] **Investigate BBL's ranking methodology** — what algorithm/weighting do they use? Is it documented or do we need to reverse-engineer from displayed ranks?
- [ ] **Document current aDAO ranking method** — what's actually live in `nft-explorer-index.html` / `nft-explorer-app.js` / `adao_json_storage` snapshots?
- [ ] **Decide on convergence approach** — full BBL parity, hybrid, or stay independent with documented rationale?
- [ ] **Implement chosen approach** — code changes to whatever produces the rank values in the snapshot data
- [ ] **Migration plan** — if rank values change, communicate to holders (rank changes are visible and people care about them)

### Webmanifest / favicon 404s in production
Today's console log showed `/site.webmanifest` and `/favicon.ico` 404'ing on `www.thealliancedao.com` despite both files being present in the repo. Either Vercel routing issue, deploy timing, or CSP. Investigate.

- [ ] Verify the deployed Vercel build actually includes both files (check Vercel deployment file listing)
- [ ] Check if there's a `vercel.json` rewrite or header config that's intercepting these paths
- [ ] Check if response is genuinely 404 vs. 200 with wrong MIME type

---

## 🚀 Future projects — separate threads

These are too big to roll into a normal patch — flagged for their own focused work later:

### Fully-live TLA Deposits tile (replace snapshot dependency)
**See `DESIGN_live_tla_deposits.md` for the full design.** Two-stage plan: Stage 1 makes the dashboard tile live (DAO's own positions only); Stage 2 extends to a full live TLA Stats page with auto-detected pool whitelist. Stage 1 depends on the resilience prereqs above (per-fetch timeout, fallback LCD, per-fetch isolation) being in place first.

Key model correction recorded in the design doc: each TLA pool has TWO positions (non-amplified white deposit + amplified orange deposit). The 5 ampLP denoms the user pasted are only half the picture — non-amp LPs need to be enumerated too. Success metric is ampLP token count growth per epoch, not USD.

### TLA data automation — replacing `tla_tool.html`

**See `DESIGN_tla_data_automation.md`.** The earlier design draft in this file (and earlier in this conversation) severely underscoped the problem. After actually reading `tla-stats.html` (7849 lines) and `tla_tool.html` (13,320 lines), the real scope:

- `tla-stats.html` is a sophisticated analytical dashboard with 4 computed scores per pool (Performance, Support, Opportunity, Access), historical 4-epoch averages, dual APR (amp + non-amp), bribe attribution (PD vs Other), Votion+aDAO+Other vote breakdowns.
- `tla_tool.html` is the manual capture tool the user runs every Sunday — it's NOT a single fetch, it's a multi-step interactive pipeline including manual paste-from-UI steps (Phoenix Directive bribes, Votion lockup data) plus computed scores and curator judgment (access score, included_in_grade flag).
- **Goal: replace `tla_tool.html` with a Vercel cron producing the same v3 JSON.** This is a 6-10 session project, not a single session.

**The user's original spec to deving.zone was reasonable** given what `tla-stats.html` actually consumes. Earlier dismissal of that spec as "over-spec" was wrong.

**Don't pay deving.zone IF we commit to building it in-house.** But the in-house path is real work spanning months. If user wants this fast, paying deving.zone may still be the right call — that decision needs revisiting with this honest sizing.

**Phase 0 investigations the next session must do BEFORE any code:**

1. Eris Amp Compounder farm registry pattern
2. Bucket-level VP query format from the four `TLA_CONTRACTS`
3. Astroport TRPC stability + what other `charts.*` endpoints exist
4. Phoenix Directive — does it have a programmatic API?
5. Votion full API surface beyond what `tla-tool_ext.html` already uses
6. Astroport pool-discovery (full list, not hardcoded)
7. Read the remaining ~10k lines of `tla_tool.html` not yet traced

Each is 10-30 min depending on the answer. Do them ALL before estimating effort or writing code.

### Static site generator migration (Astro / Eleventy)
The "every page must look like index.html" cross-page consistency requirement currently means duplicating the same shared HTML/CSS/JS across 18+ files by hand. A static site generator would template the shared chrome and dramatically reduce the maintenance burden. Big refactor though — entire repo restructure. Not urgent, but worth keeping in mind whenever cross-page work feels painful.

### TLA data collection automation
- Build cron-driven workflow on `0 59 23 * * 0` (Sunday 23:59 UTC, end of TLA epoch)
- Default to **Vercel cron** (per project convention), not GitHub Actions
- Auto-commit JSON snapshots to the 3 storage repos via PAT
- Replace manual Sunday-night data capture

### Capa Protocol integration prep
- Once partnership solidifies — likely new pages/sections for Capa marketplace
- Possible new lore integration (per the framework Lion DAO established)

---

## 🧹 Low priority / cleanup

- [ ] Remove dead Logos modal HTML in `index.html` (line ~1745+) — trigger was removed but the modal markup is still there
- [ ] Remove `'logo-modal-trigger': 'logoModal'` mapping in JS (line ~5396 area)
- [ ] Delete leftover stale files from main repo if still present: `news.html`, `graphs.html`, `rampt.html`, `on-ramp.html`, `off-ramp.html`, `alliance-dao-docs.html`, `test-page.html` (PROJECT_KNOWLEDGE marks these as removed but the latest GitHub file listing still shows them — verify and clean up)
- [ ] Delete duplicate Vercel project `a-dao-links-site` (keep `a-dao-links-site-t6nu`)

---

## 📝 Open questions / decisions needed

- [ ] **LST ratio fallbacks** (`bLUNA || 1.6048`, `ampLUNA || 1.9015`, `arbLUNA || 2.6873`) — Design Principle #1 violation; kept because ratios drift slowly. Keep or remove?
- [ ] Dead Logos modal — remove entirely, or rebuild with actual logos?
