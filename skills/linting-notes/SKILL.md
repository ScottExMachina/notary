---
name: linting-notes
description: |
  Lint and repair filed notes in a Notary Obsidian vault. Use when the user asks
  to lint the vault, clean up notes, fix frontmatter, check note hygiene, or do a
  vault health check. Reads the vault config and validates every note in notes/:
  fixes casing/typos silently, regenerates missing summaries and thin tags (and
  logs it), and asks before resolving genuine ambiguity. Never alters raw content
  under the Notes heading. Only acts inside a Notary vault (a folder with a
  notary.config.md at its root).
version: 0.1.0
license: MIT
---

# Vault Linter

Keep filed notes consistent and complete. Operate on `notes/` only.

## Step 0 — Vault guard (do this first)

Check for `notary.config.md` in the current working directory (the vault root).

- **If missing:** do nothing. Say: "No `notary.config.md` here — this doesn't look like a Notary vault, so there's nothing to lint. Run `/notary-setup` to initialize one." Then stop.
- **If present:** continue.

## Step 1 — Load context

Read `notary.config.md` frontmatter for the valid `types`, valid `domains`, `tag_conventions`, `summary_max_bullets`, `max_topic_tags` (default 5 when absent), and `folders.notes`. Read `index.md` for the established tag vocabulary.

## Step 2 — Check every note in `notes/`

For each `.md` file, classify each issue into one of three tiers and act accordingly.

### Auto-fix silently (no prompt; not logged individually)

- `type` or `domain` casing errors — `Meeting` → `meeting`.
- Unambiguous typos that map to exactly one valid value — `delivry` → `client-delivery`, `clientdelivery` → `client-delivery`.
- Date reformatting to `YYYY-MM-DD` when the intended date is unambiguous.
- Tag normalization to the convention casing/hyphenation (e.g. `Acme Corp` → `acme-corp`) when it clearly maps to an existing index tag.
- Legacy `#follow-up` markers — if any bullet ends with `#follow-up`, convert it to an open checkbox `- [ ]`, drop the tag, and move it into the `## Follow-ups` section (create that section directly below `## Summary` if it doesn't exist).
- Misplaced follow-ups — relocate any `- [ ]` checkbox found in `## Summary` into the `## Follow-ups` section (directly below `## Summary`). `## Summary` should hold plain `-` bullets only.

### Auto-fix and log

- **Missing or empty `## Summary`** — regenerate from the note's `## Notes` content (≤ `summary_max_bullets` plain `-` bullets; summary points only). Any todo/action/deadline goes into a `## Follow-ups` section below it as open checkbox bullets `- [ ]`; omit that section if there are none.
- **Missing or empty frontmatter `summary`** — generate a one-sentence summary.
- **Missing or thin `tags`** — add tags from the `index.md` vocabulary where you're confident they apply, following `tag_conventions`: every person and organization, and up to `max_topic_tags` topics.

### Ask before acting (one consolidated prompt at the end)

- `domain` is empty or doesn't match a valid value and content gives no clear signal.
- A person tag could match more than one person already in the index.
- `type` is genuinely unclear from the content.

Collect these into a single batch and ask the user once, rather than interrupting per note.

## Step 3 — Hard rules

- **Never** alter, reorder, or drop anything under `## Notes`. Summaries and frontmatter only.
- If a note has no `## Notes` heading but has body content, treat the existing body as raw content — do not overwrite it; add structure around it and log the change.
- Don't invent tags that aren't supported by the content.

## Step 4 — Log everything

Append (never overwrite) a single run entry to `log.md`:

```markdown
## 2025-06-02 14:45 — Lint
- Checked: 47 notes
- Auto-fixed (logged): 2
  - old-capture.md: regenerated empty ## Summary
  - may-28-note.md: added tags [acme-corp, sow] from index
- Silent fixes: 5 (casing/typos)
- Questions: 1
  - mystery-note.md: domain unclear — bd or ops?
- Unresolved: 1 (awaiting answer above)
```

Report the same summary to the user, surfacing the questions clearly so they can answer.

## Worked example

**Before — `notes/may-28-note.md`:**
```markdown
---
type: Meeting
domain: delivry
date: 5/28/2025
summary:
tags: []
---

## Notes
Quick sync with the Acme team on SOW scope.
```

**After (silent + logged fixes):**
```markdown
---
type: meeting
domain: client-delivery
date: 2025-05-28
summary: Quick sync with the Acme team on SOW scope.
tags: [acme-corp, sow]
---

## Summary
- Synced with the Acme team on SOW scope

## Notes
Quick sync with the Acme team on SOW scope.
```

Log gets: regenerated summary + added tags (logged); `type` casing and `domain` typo and date reformat (silent).
