---
notary_version: 0.1.0
types: [meeting, note, transcript, other]
domains: [bd, client-delivery, thought-leadership, ops]
folders:
  inbox: inbox
  notes: notes
tag_conventions:
  people: firstname-lastname
  organizations: hyphenated-natural-name
  topics: descriptive-hyphenated
summary_max_bullets: 5
---

# Notary Config

This file defines the taxonomy for **this** vault. The Notary skills read the
frontmatter above; edit those lists to customize the vault for your work. This is
the one file that differs between, say, a work machine and a personal machine —
the skills themselves stay the same.

## Fields

- **types** — the allowed values for a note's `type` frontmatter. The processor
  picks from this list; use the catch-all (`other`) only when nothing fits.
- **domains** — the allowed values for a note's `domain`. These are your top-level
  areas of work. Personal example: `[creative, home, learning, finance]`.
- **folders** — where captures land (`inbox`) and where filed notes live (`notes`).
  Change only if you want different folder names.
- **tag_conventions** — how the processor formats extracted tags:
  - *people* → `firstname-lastname` (e.g. `john-smith`)
  - *organizations* → hyphenated natural name (e.g. `acme-corp`)
  - *topics* → descriptive and hyphenated (e.g. `operating-model`)
- **summary_max_bullets** — how many bullets the generated `## Summary` may have.

Follow-ups aren't a tag. When the processor writes the `## Summary`, the note's
points come first as plain `-` bullets, then anything that reads like a todo or
deadline is listed as an open checkbox `- [ ]` at the end. See `follow-ups.md` for
the running list of open items across the vault.

## How it's used

1. Drop a raw note into `inbox/` (optionally via a template in `templates/`).
2. Run the **processing-notes** skill — it reads this config, enriches the note,
   files it to `notes/`, and updates `index.md` and `log.md`.
3. Run the **linting-notes** skill periodically to keep everything consistent.
4. Open `follow-ups.md` to see every open `- [ ]` item across your notes.
