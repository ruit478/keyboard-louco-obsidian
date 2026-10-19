# CLAUDE.md — Vault conventions

This is the Mochicap Obsidian vault. These conventions apply to humans and AI agents (Claude Code, etc.) working in it.

## Structure

- `00 Meta/` — system stuff: dashboard, templates, attachments. Never worked in directly.
- `01 Inbox/` — quick capture; notes are filed elsewhere during weekly review.
- `10 Business/` — non-technical side: strategy, decisions, product, marketing, finance, legal, suppliers.
- `20 Technical/` — technical side: backend, frontend, infrastructure, design, runbooks.
- `90 Archive/` — completed or abandoned work.

Keep it flat: max 3 levels (`20 Technical/Backend/note.md`).

## Conventions

- **Frontmatter minimum** on every note: `type`, `status`, `date` (ISO `YYYY-MM-DD`).
- **Tags are for status only** — `#active`, `#waiting`, `#locked`, `#someday`. Never re-tag the folder tree.
- **Links are for relationships** — cross-link freely with `[[wikilinks]]`; use full paths since `_Index` filenames repeat.
- **Each folder has an `_Index.md`** map of content; `00 Meta/Home.md` links the top level.
- **Locked decisions live in `10 Business/Decisions/`** — other notes link to them; never restate them.
- Dates are always `YYYY-MM-DD`.
