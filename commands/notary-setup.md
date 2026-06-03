---
description: Initialize or sync a Notary Obsidian vault in the current folder (idempotent — never overwrites your notes or config)
---

You are setting up or updating a **Notary vault** in the current working directory. This command is idempotent: run it on an empty folder to scaffold a new vault, or on an existing vault to sync the latest plugin-owned templates. It must never destroy user content.

## Source of truth

The bundled vault template lives at `${CLAUDE_PLUGIN_ROOT}/vault-template/`. This mirror is exactly what an installed vault should look like. Read from it; copy into the current working directory.

If `${CLAUDE_PLUGIN_ROOT}` is not set in this environment, locate the `notary` plugin's `vault-template/` directory under the Claude plugins cache and use that. Tell the user which source path you resolved.

## What to do

1. **Confirm the target.** State the current working directory and that you'll set up a Notary vault there. Proceed once confirmed (the folder may already contain `.obsidian/` — that's fine, vaults and Claude projects share the folder).

2. **Create-if-absent (never overwrite):**
   - `inbox/` — create the folder (with a `.gitkeep` if empty).
   - `notes/` — create the folder.
   - `templates/` — create the folder.
   - `index.md` — copy from the template only if it does not already exist.
   - `log.md` — copy from the template only if it does not already exist.
   - `notary.config.md` — copy from the template **only if it does not already exist**. This file is user-owned customization; an existing one is never touched.

3. **Sync plugin-owned files (create or refresh):**
   - Everything under `templates/` in the mirror is plugin-owned. Copy each template file in; if it already exists, refresh it to the bundled version (these are not user content). Report any that changed.

4. **Sample content (first init only):**
   - The mirror's `notes/` contains a clearly-labeled sample note, plus matching entries already present in the template `index.md`/`log.md`. Copy the sample note in **only when initializing a fresh vault** (i.e. `notes/` did not already exist). Do not re-add it on a sync of an existing vault. Mention that the sample is safe to delete.

5. **Report a summary** of what was created, what already existed (skipped), and what was refreshed. Example:
   ```
   Notary vault ready in /Users/scott/vault
   Created:   inbox/, notes/, templates/, index.md, log.md, notary.config.md, notes/2025-06-02-acme-q3-roadmap-meeting.md
   Refreshed: templates/inbox-capture.md, templates/meeting.md
   Skipped:   (none)
   Next: edit notary.config.md to set your domains and types, then drop captures into inbox/ and run the inbox-processor skill.
   ```

## Hard rules

- Never overwrite `notary.config.md` if it exists.
- Never modify, move, or delete anything under `notes/` or `inbox/` that the user created.
- Only `templates/` (and future plugin-owned views) get refreshed on a re-run.
- If anything is ambiguous, ask before writing.

## Optional: wire up Obsidian's Templates plugin

Offer (do not do automatically) to point Obsidian's core **Templates** plugin at the `templates/` folder by setting the template folder path in `.obsidian/`. Only do this if the user explicitly agrees, since it modifies their Obsidian config. If they decline, tell them they can set it manually in Obsidian: Settings → Templates → Template folder location → `templates`.
