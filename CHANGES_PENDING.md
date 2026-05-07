# Changes Pending — aDAO-links-site

> Rolling list of identified changes for the next batch upload. Add to this as we identify more, check off as completed.

---

## ✅ Completed (already in updated index.html — needs commit/push)

- [x] Removed `test-page.html` link from index footer (line 2227 in original)
- [x] Removed `Logos` modal trigger link from index footer (line 2224 in original)
- [x] Confirmed all deleted page references are clean — no broken internal links

## ✅ Already done at the repo level

- [x] Deleted from repo: `graphs.html`, `news.html`, `rampt.html`, `on-ramp.html`, `off-ramp.html`, `alliance-dao-docs.html`
- [x] (User pending) Delete `test-page.html` from repo

---

## 🔧 Ready to do — index.html SEO additions

Add to `<head>` of index.html (high priority — directly improves Google indexing):

- [ ] Add canonical URL: `<link rel="canonical" href="https://www.thealliancedao.com/">`
- [ ] Add `og:url`, `og:type`, `og:site_name`
- [ ] Add `og:image:width` / `og:image:height` (helps preview rendering)
- [ ] Add Twitter Card meta tags (4 tags)
- [ ] Add explicit `<meta name="robots" content="index, follow">`
- [ ] Add `<meta name="theme-color" content="#0a0b0f">`
- [ ] Improve meta description copy (be more specific, ~150 chars)

## 🗺️ Ready to do — sitemap.xml cleanup

- [ ] **Remove dead entries** (causing 404s for crawlers):
  - graphs.html
  - news.html
  - nft-explorer.html (old single-page version, replaced by nft-explorer-index.html)
  - on-ramp.html
  - off-ramp.html
  - rampt.html
- [ ] **Add missing active pages** (currently not indexed):
  - tools.html (major hub!)
  - dao_governance.html
  - dao_governance_tool.html
  - dao_tla_deposits.html
  - dao_treasury.html
  - capa_lp_converter.html
  - fuel_tracker.html
  - tla-docs.html
- [ ] (Decide) Remove `alliance-dao-docs.html` entry (file was deleted)

## 🐛 Ready to do — bug fixes

- [ ] **rawgit.hack issue:** In `tla_tool.html` line 142, change ext-tab link from `https://raw.githack.com/defipatriot/aDAO-links-site/main/tla-tool_ext.html` to local `tla-tool_ext.html`. Eliminates the "One more step" interstitial and serves from Vercel CDN.

---

## 🧹 Cleanup — dead code (low priority, safe to defer)

- [ ] Remove dead Logos modal HTML in index.html (line 1745+)
- [ ] Remove `'logo-modal-trigger': 'logoModal'` mapping in JS (line 5396 area)

---

## 🌐 SEO — site-wide pattern (after index.html template proven)

Once we nail the index.html SEO setup, apply the same pattern to every other active page:
- [ ] Apply meta tag template to: ally, alliances, tla-stats, tla-docs, tla_tool, dao_governance, dao_treasury, dao_tla_deposits, nft-explorer-index, planet-map, rarity-explained, release-history, tutorials, links, tools, capa_lp_converter, fuel_tracker, dao_governance_tool
- [ ] Each page needs unique title, unique description, own canonical URL
- [ ] og:image can stay shared (the 512x512 logo) or per-page if specific imagery exists

---

## ⚙️ Infrastructure — Vercel side (verify, don't change in repo)

- [ ] Verify `theadao.com` → `thealliancedao.com` is a **301** (permanent), not 302
- [ ] Verify both domains serve over HTTPS with valid certs
- [ ] Confirm Vercel Analytics is recording (already wired in code)

---

## 🚀 Future projects — separate threads

These are too big to roll into this batch — flagged for their own focused work later:

### TLA data collection automation
- Build GitHub Actions workflow on `0 59 23 * * 0` cron (Sunday 23:59 UTC)
- Port browser-based collection logic to Node.js
- Auto-commit JSON snapshots to the 3 storage repos via PAT
- Replace manual Sunday-night data capture

### Capa Protocol integration prep
- Once partnership solidifies — likely new pages/sections for Capa marketplace
- Possible new lore integration (per the framework Lion DAO established)

### Other pages SEO sweep
- Apply meta tag pattern to all active pages (after index.html template proven)

---

## 📝 Open questions / decisions needed

- [ ] Want to migrate to a static site generator (Eleventy, Astro) eventually so meta tags / nav are templated, not duplicated across files? Big refactor, not urgent.
- [ ] Should the dead Logos modal be removed entirely, or rebuilt with actual logos? (Currently just removed the broken trigger.)
