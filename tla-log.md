# TLA Stats Changelog

This is the change history for `tla-stats.html` (the Terra Liquidity Alliance public dashboard).
Newest revisions on top. Times are UTC.

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
