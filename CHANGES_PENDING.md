# Changes Pending — aDAO-links-site

> Rolling list of identified changes for the next batch upload. Newest items at the top of the active section. Add as we identify, check off as completed.

---

## 🔥 Active / next round

### Dashboard tiles — DAO Broken/Held NFTs missing implementation
The `#dao-broken-total` tile has been spinning since launch. Discovered while debugging Rev 3.30 — turns out it has **zero** live-data writes anywhere in the codebase (only the chart icon attaches to it). Probably should be derived from the live `nfts` array via `nfts.filter(n => n.broken && n.owner === DAO_MAIN_WALLET).length` — but PROJECT_KNOWLEDGE notes the DAO holds 1,000 broken NFTs across **three** wallets (`...8ywv`, `...417v`, `...6ugw`), so the filter probably needs to match any of those three owners. Design call needed before implementing.

### Cleanup — Rev 3.29 dead code in `fetchTlaFromGitHub`
`fetchTlaFromGitHub()` still runs the parallel 6-epoch fetch against `adao_json_storage` on every page load, even though that repo has zero matching files. It harmlessly returns null (downstream builders already fall back to `fetchTlaData`), but it's wasted network work. Either delete the function and remove its caller in `fetchLiveOnChainData`'s finally block, or repoint it to `tla_json_storage` and consolidate the two TLA fetch paths.

### Dashboard live queries — additional resilience (follow-up to Rev 3.27 / 3.29 / 3.30)
Rev 3.27 stopped the cascade crash. Rev 3.29 fixed the TLA snapshot fetch with parallel epoch-fallback. Rev 3.30 fixed the stale-data UX. The remaining resilience items:

- [ ] **Per-fetch try/catch isolation in `fetchLiveOnChainData`.** Right now all treasury balance fetches share one outer `try`. One LCD 500 still produces null for downstream calcs (which now display `—` instead of crashing — better, but not great). Each fetch should be its own isolated promise so a single asset failure only blanks the affected tile.
- [ ] **Per-fetch timeout in `fetchLiveOnChainData`.** Today's load took 14.67s — almost all of it in the Terra LCD `Promise.all`. A 5–8 second timeout per request via AbortController would cap the worst case. Reuse the pattern Rev 3.29 introduced for the TLA snapshot fetch.
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
