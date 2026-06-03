---
name: inbox-processor
description: |
  Process raw captures in a Notary Obsidian vault. Use when the user asks to
  process the inbox, file their notes, clear the inbox, or enrich captured notes
  — i.e. when a vault has unprocessed notes in its inbox/ folder. Reads the
  vault config and tag index, enriches frontmatter (type, domain, summary, tags),
  inserts a Summary section, preserves raw content, moves notes to notes/, and
  updates index.md and log.md. Only acts inside a Notary vault (a folder with a
  notary.config.md at its root).
version: 0.1.0
license: MIT
---

# Inbox Processor

Turn raw captures in `inbox/` into enriched, filed notes in `notes/`, then update the vault's tag index and operations log.

## Step 0 — Vault guard (do this first)

Check for `notary.config.md` in the current working directory (the vault root).

- **If it is missing:** this folder is not a Notary vault. Do nothing. Say: "No `notary.config.md` here — this doesn't look like a Notary vault, so I'm not processing anything. Run `/notary-setup` to initialize one." Then stop.
- **If it is present:** continue.

## Step 1 — Load context

1. Read `notary.config.md` frontmatter. Extract:
   - `types` — the allowed `type` values
   - `domains` — the allowed `domain` values
   - `folders.inbox` and `folders.notes` — folder names (default `inbox`, `notes`)
   - `tag_conventions` — formatting rules for people / organizations / topics
   - `summary_max_bullets` — max bullets in the `## Summary` section (default 5)
   - `followup_marker` — the inline follow-up tag (default `#follow-up`)
2. Read `index.md` if it exists. The tag headings in it are your **preferred tag vocabulary**. Always prefer an existing tag over coining a near-duplicate (e.g. reuse `acme-corp`; don't create `acme`).

If `inbox/` is empty, report "Inbox is empty — nothing to process" and stop.

## Step 2 — Process each note in `inbox/`

For every `.md` file in the inbox folder, in filename order:

1. **Read the raw content**, including any existing frontmatter.
2. **`type`** — if frontmatter already sets a valid `type`, honor it. Otherwise infer from content and pick the best fit from the config's `types`. Use the catch-all type (e.g. `other`) only when nothing fits.
3. **`domain`** — if already set and valid, honor it. Otherwise infer from context against the config's `domains`. If genuinely unclear, leave `domain` empty and note it in the run log rather than guessing.
4. **`date`** — if set, keep it. Otherwise use the date in the filename if present, else today's date. Format `YYYY-MM-DD`.
5. **Tags** — extract and normalize per `tag_conventions`, matching existing index vocabulary wherever possible:
   - People → `firstname-lastname`
   - Organizations / entities → hyphenated natural name
   - Topics → descriptive, hyphenated
   - Flat array, no namespacing.
6. **`summary`** (frontmatter) — write one scannable sentence capturing the note's point.
7. **`## Summary` section** — insert at the top of the body, above everything else, with up to `summary_max_bullets` tight bullets:
   - Convert every `- [ ]` checkbox in the raw content into a bullet ending with the `followup_marker`.
   - Mark other clear action/commitment/deadline signals with the `followup_marker` inline.
8. **Preserve raw content** unchanged under a `## Notes` heading below the summary. Never edit, summarize over, or drop the original text.
9. **Move** the file from `inbox/` to `notes/` (move — do not copy then delete, and do not leave a duplicate in inbox). Keep the filename unless it collides in `notes/`; if it collides, append `-2`, `-3`, etc.

### Resulting note shape

```markdown
---
type: meeting
domain: client-delivery
date: 2025-06-02
summary: One scannable sentence about the note.
tags: [acme-corp, john-smith, roadmap]
---

## Summary
- Tight bullet capturing a key point
- Action or deadline surfaced #follow-up

## Notes
[original raw content, untouched]
```

## Step 3 — Update `index.md`

For each processed note, under every tag it carries, add a reference line. Create the tag heading if it doesn't exist. Keep entries sorted newest-first within a tag.

Line format:
```
- [[note-filename-without-extension]] — YYYY-MM-DD — frontmatter summary
```

Update the `_Last updated:_` line at the top to the current timestamp and "by Inbox Processor".

## Step 4 — Append to `log.md`

Append (never overwrite) a timestamped run entry:

```markdown
## 2025-06-02 14:32 — Inbox Processing
- Processed: 3 notes
- Filed: acme-q3-meeting.md, bd-call-john-smith.md, ops-standup.md
- Tags added: acme-corp, john-smith, roadmap
- Warnings: 1 — may-28-note.md: domain left empty (no clear signal)
```

Report the same summary to the user.

## Worked example

**Input — `inbox/acme-meeting.md`:**
```markdown
Met with John Smith and Sarah Lee at Acme about the Q3 roadmap.
August has a resourcing gap — biggest risk.
- [ ] Close SOW renewal before end of June
Offshore delivery decision pushed to next meeting.
```

**Output — `notes/acme-meeting.md`:**
```markdown
---
type: meeting
domain: client-delivery
date: 2025-06-02
summary: Acme Q3 roadmap review flagged an August resourcing gap and a June SOW renewal deadline.
tags: [acme-corp, john-smith, sarah-lee, roadmap, resourcing, sow]
---

## Summary
- Q3 roadmap reviewed; August resourcing gap is the primary risk
- Close SOW renewal before end of June #follow-up
- Offshore delivery decision deferred to next meeting #follow-up

## Notes
Met with John Smith and Sarah Lee at Acme about the Q3 roadmap.
August has a resourcing gap — biggest risk.
- [ ] Close SOW renewal before end of June
Offshore delivery decision pushed to next meeting.
```

Plus: `index.md` gains references under `acme-corp`, `john-smith`, `sarah-lee`, `roadmap`, `resourcing`, `sow`; `log.md` gains a run entry.

## Rules

- Never modify content under `## Notes`.
- Honor user-set `type`/`domain`; only infer what's missing.
- Prefer existing index tags over new near-duplicates.
- Move notes out of `inbox/` — a successful run leaves the inbox empty.
- When uncertain about a `domain`, leave it empty and log it; don't fabricate.
