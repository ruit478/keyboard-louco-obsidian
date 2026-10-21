# CLAUDE.md: Vault conventions

This is the Mochicap Obsidian vault. These conventions apply to humans and AI agents (Claude Code, etc.) working in it.

## Structure

- `00 Meta/`: system stuff (dashboard, templates, attachments). Never worked in directly.
- `01 Inbox/`: quick capture; notes are filed elsewhere during weekly review.
- `10 Business/`: non-technical side (strategy, decisions, product, marketing, finance, legal, suppliers).
- `20 Technical/`: technical side (backend, frontend, infrastructure, design, runbooks).
- `90 Archive/`: completed or abandoned work.

Keep it flat: max 3 levels (`20 Technical/Backend/note.md`).

## Conventions

- **Frontmatter minimum** on every note: `type`, `status`, `date` (ISO `YYYY-MM-DD`).
- **Tags are for status only**: `#active`, `#waiting`, `#locked`, `#someday`. Never re-tag the folder tree.
- **Links are for relationships**: cross-link freely with `[[wikilinks]]`; use full paths since `_Index` filenames repeat.
- **Each folder has an `_Index.md`** map of content; `00 Meta/Home.md` links the top level.
- **Locked decisions live in `10 Business/Decisions/`**: other notes link to them; never restate them.
- **Write like a human**: natural prose with proper punctuation (commas, colons, semicolons); no em dashes.
- Dates are always `YYYY-MM-DD`.

## Working rules for AI agents

- **Full write access inside the vault**: create, edit, and move files anywhere within this folder without asking.
- **Ask before touching anything outside the vault**: no edits to paths outside this folder without explicit permission.
- **No commits or pushes**: leave all changes uncommitted in the working tree; a human reviews them first.
- **Don't delete files**: nothing is committed, so deletions are unrecoverable. Move completed or abandoned work to `90 Archive/` instead; ask if something genuinely must be removed.
