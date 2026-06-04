# Obsidian Work Vault — System Design & Build Spec

## Philosophy

Obsidian is the **capture point and ledger** for work — not a project management system, not a second brain, not a wiki. It sits on top of existing file systems and captures the intellectual residue of work: meetings, notes, transcripts, insights, decisions. The design prioritizes low friction on the way in and reliable retrieval on the way out.

---

## Design Principles

- **Minimal structure** — two content folders plus two root-level system files
- **Frontmatter does the organizational work** — not file location
- **Inbox as processing state** — unprocessed notes live in `/inbox`, filed notes live in `/notes`
- **Claude handles enrichment** — tags, domain, summary generated automatically
- **Dataview handles retrieval** — queries replace folders and manual organization

---

## Vault Structure

```
/inbox          ← unprocessed captures, zero friction
/notes          ← all filed notes, flat
index.md        ← auto-maintained index of tags → notes
log.md          ← log of all Claude skill operations
```

---

## Work Domains

| Domain | Description |
|--------|-------------|
| `bd` | Business development, pursuits, relationships |
| `client-delivery` | Client engagements, meetings, project work |
| `thought-leadership` | Frameworks, POVs, IP development |
| `ops` | Firm operations, internal meetings, admin |

---

## Note Types

| Type | Description |
|------|-------------|
| `meeting` | Notes from a meeting, processed or summarized |
| `note` | Original thinking, reference, insights |
| `transcript` | Raw transcript, source material |
| `other` | Anything that doesn't fit above |

---

## Frontmatter Schema

```yaml
---
type: meeting
domain: client-delivery
date: 2025-06-02
summary: One-sentence summary of the note for scanning
tags: [acme-corp, john-smith, sarah-lee, roadmap, resourcing]
---
```

### Field Rules

- `type` — lowercase, one of the four types above
- `domain` — lowercase, one of the four domains above
- `date` — YYYY-MM-DD format
- `summary` — single sentence, Claude-generated, scannable in Dataview
- `tags` — flat array, no namespacing, follows conventions below

---

## Tagging Conventions

**People** — `firstname-lastname` (e.g., `john-smith`, `sarah-lee`). Tag everyone.

**Organizations/Entities** — natural name, hyphenated (e.g., `acme-corp`, `deloitte`). Tag every org.

**Topics** — descriptive, hyphenated (e.g., `roadmap`, `sow`, `operating-model`). Capped per note at `max_topic_tags` (default 5); keep the most durable, reusable topics and let full-text search cover the rest.

Follow-ups are not tags. Action items live as open `- [ ]` checkboxes in a dedicated `## Follow-ups` section — no `#follow-up` marker.

---

## Note Anatomy

```markdown
---
type: meeting
domain: client-delivery
date: 2025-06-02
summary: Acme Q3 roadmap review flagged resourcing risk and SOW renewal deadline
tags: [acme-corp, john-smith, sarah-lee, roadmap, resourcing, sow]
---

## Summary
- Acme reviewing Q3 roadmap; August resourcing gap is primary risk
- John Smith flagged budget sensitivity around change orders

## Follow-ups
- [ ] SOW renewal must close before end of June
- [ ] Offshore delivery model decision deferred to next meeting

## Notes
[raw capture, transcript, or unprocessed content below this line]
```

**Transcripts** are kept as `type: transcript`. A processed `type: meeting` note is generated from them, linked via shared tags.

---

## Capture Flow

1. Create note in `/inbox` with raw content — add `type` and `domain` if known, otherwise leave for Claude
2. Run **Inbox Processing skill** in Claude Code
3. Claude reads existing `index.md` for tag vocabulary
4. Claude enriches frontmatter, inserts `## Summary` bullets at top of body, populates tags
5. Note is moved from `/inbox` to `/notes`
6. Claude updates `index.md` and appends to `log.md`
7. Done

---

## Skills

### 1. Inbox Processing Skill

**Trigger:** Run manually when inbox has unprocessed notes

**Claude's job for each note in `/inbox`:**
- Read `index.md` first — use existing tags as the preferred vocabulary before creating new ones
- Read raw content
- Infer `type` from content only if not already set in frontmatter
- Infer `domain` from context only if not already set in frontmatter
- Extract all people mentioned → `firstname-lastname` tags, matching existing index tags where possible
- Extract all organizations/clients mentioned → entity tags, matching existing index tags where possible
- Extract key topics → topic tags (at most `max_topic_tags`, default 5), matching existing index tags where possible
- Write one-sentence `summary` for frontmatter
- Insert `## Summary` at top of note body with up to 5 tight bullets — summary points only
- Add a `## Follow-ups` section directly below it listing every todo/action/deadline (including raw `- [ ]` items) as open `- [ ]` checkboxes; the checkbox is the only signal (no `#follow-up`). Omit the section when there are no action items
- Preserve all raw content under `## Notes` below the follow-ups
- Move note from `/inbox` to `/notes` (move, not copy/delete)
- Update `index.md` — add new note reference under each relevant tag, create tag entry if it doesn't exist
- Append run summary to `log.md` — timestamp, notes processed, files moved, tags added, any warnings

---

### 2. Linting Skill

**Trigger:** Run periodically (weekly or on demand)

**Claude auto-fixes silently:**
- `type` or `domain` casing errors or unambiguous typos (e.g. `Meeting` → `meeting`, `delivry` → `client-delivery`)

**Claude auto-fixes and logs the change:**
- Missing or empty `## Summary` — regenerates from note content
- Missing or thin `tags` — applies matching vocabulary from `index.md` where confident

**Claude asks before acting:**
- Ambiguous `domain` with no clear signal in content
- Person name that could match multiple people in the index
- `type` that is genuinely unclear from content

**Always:**
- Appends all changes, questions, and unresolved issues to `log.md`
- Never silently drops or alters raw content under `## Notes`

---

## Tag Index Format

`index.md` — maintained by Claude at vault root, updated after every inbox processing run.

```markdown
# Vault Tag Index
_Last updated: 2025-06-02 14:32 by Inbox Processor_

## acme-corp
- [[acme-q3-roadmap-meeting]] — 2025-06-02 — Acme Q3 roadmap review flagged resourcing risk
- [[acme-sow-kickoff]] — 2025-05-28 — SOW scope and timeline alignment

## john-smith
- [[acme-q3-roadmap-meeting]] — 2025-06-02 — Acme Q3 roadmap review flagged resourcing risk
- [[bd-intro-call-june]] — 2025-06-01 — Intro call with John Smith at Acme
```

---

## Log Format

`log.md` — maintained by Claude at vault root, append-only.

```markdown
# Vault Operations Log

## 2025-06-02 14:32 — Inbox Processing
- Processed: 3 notes
- Filed: acme-q3-meeting.md, bd-call-john-smith.md, ops-standup.md
- Tags added: acme-corp, john-smith, roadmap, resourcing
- Warnings: none

## 2025-06-02 14:45 — Lint
- Checked: 47 notes
- Issues: 2
  - old-capture.md: missing domain
  - may-28-note.md: empty tags array
```

---

## Claude Code Build Prompt

> I'm building a personal knowledge management system in Obsidian for my consulting work. I need you to build two Claude Code skills as natural language instruction files. Here is the full system design:
>
> **Vault structure:**
> ```
> /inbox          ← unprocessed captures
> /notes          ← all filed notes, flat
> index.md        ← auto-maintained tag index at vault root
> log.md          ← operation log at vault root
> ```
>
> **Frontmatter schema:**
> ```yaml
> ---
> type: meeting | note | transcript | other
> domain: bd | client-delivery | thought-leadership | ops
> date: YYYY-MM-DD
> summary: one sentence summary
> tags: [tag1, tag2, tag3]
> ---
> ```
>
> **Tagging conventions:**
> - People: `firstname-lastname` (e.g., `john-smith`)
> - Organizations: hyphenated natural name (e.g., `acme-corp`)
> - Topics: descriptive hyphenated (e.g., `operating-model`)
> - Inline follow-ups: `#follow-up` in bullet text, not frontmatter
>
> **Note body structure:**
> ```
> ## Summary
> [3-5 Claude-generated bullets, inline #follow-up tags where relevant]
>
> ## Notes
> [raw original content preserved here]
> ```
>
> **Skill 1 — Inbox Processor**
> Read `index.md` first to load existing tag vocabulary. Then read every `.md` file in `/inbox`. For each note: infer `type` and `domain` only if not already present in frontmatter — honor values the user has set; extract all people as `firstname-lastname` tags, preferring existing index tags; extract organizations and topics as tags, preferring existing index tags before creating new ones; write a one-sentence frontmatter summary; insert `## Summary` at top of note body with up to 5 tight bullets — surface `- [ ]` checkboxes as bullets with `#follow-up`, mark other action signals with `#follow-up` inline; preserve all original content under `## Notes`; move note from `/inbox` to `/notes` (move, not copy and delete). After all notes: update `index.md` adding new note references under relevant tags, update `_Last updated` line. Append timestamped run summary to `log.md`. `index.md` header: `# Vault Tag Index`. `log.md` header: `# Vault Operations Log`, append-only.
>
> **Skill 2 — Vault Linter**
> Read all `.md` files in `/notes`. Auto-fix silently: casing errors or unambiguous typos on `type`/`domain`. Auto-fix and log: missing/empty `## Summary` — regenerate from content; missing/thin tags — apply `index.md` vocabulary where confident. Ask before acting: ambiguous domain, ambiguous person name, unclear type. Never alter raw content under `## Notes`. Append all changes, questions, and unresolved issues to `log.md`.
>
> Build each skill as a separate markdown instruction file, precise enough for Claude Code to execute without additional clarification. Include example inputs and expected outputs.

---