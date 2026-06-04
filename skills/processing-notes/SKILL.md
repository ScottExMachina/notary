---
name: processing-notes
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
   - `max_topic_tags` — max number of *topic* tags per note (default 5 when absent)
2. Read `index.md` if it exists. The tag headings in it are your **preferred tag vocabulary**. Always prefer an existing tag over coining a near-duplicate (e.g. reuse `acme-corp`; don't create `acme`).

If `inbox/` is empty, report "Inbox is empty — nothing to process" and stop.

## Step 2 — Process each note in `inbox/`

For every `.md` file in the inbox folder, in filename order:

1. **Read the raw content**, including any existing frontmatter.
2. **`type`** — if frontmatter already sets a valid `type`, honor it. Otherwise infer from content and pick the best fit from the config's `types`. Use the catch-all type (e.g. `other`) only when nothing fits.
3. **`domain`** — if already set and valid, honor it. Otherwise infer from context against the config's `domains`. If genuinely unclear, leave `domain` empty and note it in the run log rather than guessing.
4. **`date`** — if set, keep it. Otherwise use the date in the filename if present, else today's date. Format `YYYY-MM-DD`.
5. **Tags** — extract and normalize per `tag_conventions`, matching existing index vocabulary wherever possible. Flat array, no namespacing.
   - People → `firstname-lastname`. Tag **every** person mentioned.
   - Organizations / entities → hyphenated natural name. Tag **every** org mentioned.
   - Topics → descriptive, hyphenated. Tag **at most `max_topic_tags`** topics. When the note has more candidate topics than the cap, keep the most durable and reusable ones — prefer topics already in `index.md`, recurring themes, and the note's central subjects; drop one-off mentions. Full-text search covers the long tail, so don't stretch to fill the cap.
6. **`summary`** (frontmatter) — write one scannable sentence capturing the note's point.
7. **`## Summary` section** — insert at the top of the body, above everything else. Up to `summary_max_bullets` plain `-` bullets capturing the note's key points. Summary bullets only — action items do **not** go here.
8. **`## Follow-ups` section** — place it directly below `## Summary` (and above `## Notes`). List anything that reads like a todo, action, commitment, or deadline — including any `- [ ]` checkbox already in the raw content — as open checkbox bullets `- [ ]`. The open checkbox is the **only** follow-up signal: never use a `#follow-up` tag or any inline marker. If the note has no action items, omit this section entirely.
9. **Preserve raw content** unchanged under a `## Notes` heading below the follow-ups. Never edit, summarize over, or drop the original text.
10. **Move** the file from `inbox/` to `notes/` (move — do not copy then delete, and do not leave a duplicate in inbox). Keep the filename unless it collides in `notes/`; if it collides, append `-2`, `-3`, etc.

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
- Another key point

## Follow-ups
- [ ] Action or deadline to track

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
- Q3 roadmap reviewed; the August resourcing gap is the primary risk
- Offshore delivery decision deferred to next meeting

## Follow-ups
- [ ] Close SOW renewal before end of June

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
- Tag every person and organization; cap topics at `max_topic_tags` (default 5).
- Prefer existing index tags over new near-duplicates.
- Action items go in `## Follow-ups` as open `- [ ]` checkboxes — never as `#follow-up` tags or inline markers, and never in `## Summary`.
- Move notes out of `inbox/` — a successful run leaves the inbox empty.
- When uncertain about a `domain`, leave it empty and log it; don't fabricate.
