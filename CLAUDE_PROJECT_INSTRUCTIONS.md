This project is for working on The Alliance DAO (aDAO) website at thealliancedao.com. The user is defipatriot, a council member of the DAO.

## At the start of every conversation

Fetch these three documents (in order) to load full project context. Don't ask permission — just do it:

1. https://raw.githubusercontent.com/defipatriot/website-adao-core/main/PROJECT_KNOWLEDGE.md — canonical project context: design principles, architecture, conventions, tracking responsibilities
2. https://raw.githubusercontent.com/defipatriot/website-adao-core/main/CHANGES_PENDING.md — active to-do list and open questions
3. https://raw.githubusercontent.com/defipatriot/website-adao-core/main/logs/index-log.md — changelog of shipped revisions

PROJECT_KNOWLEDGE.md has a "Tracking responsibilities" section near the top — read that carefully. You are expected to maintain these docs proactively as we work, not only when the user explicitly asks. The user has stated they will forget things, so logging decisions, future ideas, tech debt, and open questions falls on Claude.

## Repo setup

Main site repo: `github.com/defipatriot/aDAO-links-site`
Docs repo: `github.com/defipatriot/website-adao-core`
Image hosting repo: `github.com/defipatriot/aDAO-Image-Files`

To work on the site, clone the main repo:
```
git clone https://github.com/defipatriot/aDAO-links-site /home/claude/aDAO-links-site
```

The user pushes commits between chats — always sync with `git fetch && git reset --hard origin/main` before making changes.

## Workflow

Iterative, screenshot-driven UI work. Pattern: user pushes → user screenshots an issue → Claude diagnoses + edits → user pushes the fix → repeat.

The user does NOT have me push directly. I produce updated files; the user downloads them and uploads via GitHub web UI. When generating multiple iterations of the same file in a session, mention which is the final version to push (browsers rename duplicates with `(1)`, `(2)`).

## Deployment

The site is on Vercel (auto-deploys on push to `main`). Cron jobs also run on Vercel — when building scheduled tasks, default to Vercel cron rather than GitHub Actions.

## Style

The user prefers concrete, focused responses. They push back on over-engineering and busy UIs. Don't add CSS/features that weren't asked for. When something's broken, find the actual root cause (often a CSS specificity conflict from layered prior fixes) rather than piling on `!important` flags.
