# Notary

Turn an Obsidian vault into a low-friction **work ledger**. Capture raw notes with
zero ceremony; let Claude Code enrich and file them; retrieve with frontmatter and
search instead of folder gymnastics.

Notary is a Claude Code **plugin**. The skill logic is shared and versioned in the
plugin; each vault's taxonomy (your domains, types, and tag conventions) lives in a
per-vault config you customize. Same engine everywhere, different config on, say,
your work machine vs. your personal machine.

See [docs/idea.md](docs/idea.md) for the design philosophy this implements.

## What you get

- **`inbox-processor`** skill — reads raw captures in `inbox/`, enriches frontmatter
  (type, domain, summary, tags), adds a `## Summary`, preserves raw content under
  `## Notes`, files the note to `notes/`, and updates `index.md` + `log.md`.
- **`vault-linter`** skill — keeps filed notes consistent: fixes casing/typos
  silently, regenerates missing summaries and thin tags (logged), and asks before
  resolving genuine ambiguity. Never touches your raw content.
- **`/notary-setup`** command — idempotently scaffolds a new vault or syncs the
  latest templates into an existing one. Never overwrites your notes or config.

## Two update channels (read this once)

Notary has two independent things to update, and they're easy to confuse:

| You want to… | Do this | What it touches |
|---|---|---|
| Get improved skill/command **logic** | `/plugin update notary` | The engine, machine-wide. No vault action needed. |
| Get new vault **files/templates** | re-run `/notary-setup` in the vault | That vault's `templates/` and any missing scaffolding. |

Updating the plugin does **not** modify your vaults. Running `/notary-setup` does
**not** change skill logic. They're separate on purpose.

## Install

From this repo (local):

```
/plugin marketplace add /Users/scott/Code/notary
/plugin install notary
```

Later, from GitHub, point `marketplace add` at the repo URL instead.

## Set up a vault

1. Open your Obsidian vault folder in Claude Code (the vault folder **is** the
   Claude project — `.obsidian/` and `.claude/` coexist fine).
2. Run `/notary-setup`. It creates `inbox/`, `notes/`, `templates/`, `index.md`,
   `log.md`, and `notary.config.md`, plus a deletable sample note.
3. Edit `notary.config.md` to set your `domains` and `types`.

## Daily use

1. **Capture** — drop a raw note into `inbox/`. Optionally start from a template in
   `templates/` (see below). Fill `type`/`domain` if you know them; otherwise leave
   them for the processor.
2. **Process** — ask Claude to "process my inbox." The `inbox-processor` skill
   enriches each note, files it to `notes/`, and updates the index and log.
3. **Lint** — periodically ask Claude to "lint the vault." The `vault-linter` skill
   cleans up frontmatter and flags anything ambiguous.

## Customizing (the whole point)

Everything specific to your work lives in **`notary.config.md`** at the vault root:

```yaml
types: [meeting, note, transcript, other]
domains: [bd, client-delivery, thought-leadership, ops]
tag_conventions:
  people: firstname-lastname
  organizations: hyphenated-natural-name
  topics: descriptive-hyphenated
summary_max_bullets: 5
followup_marker: "#follow-up"
```

A personal vault might use `domains: [creative, home, learning, finance]`. The
skills read whatever you put here — no need to edit the skills themselves.

## Templates

`templates/` holds Obsidian note templates that pre-fill the frontmatter scaffold
and a `## Notes` body, so captures start in the right shape. They use the core
**Templates** plugin syntax (`{{date:YYYY-MM-DD}}`); no extra plugin required. If
you prefer [Templater](https://github.com/SilentVoid13/Templater), swap in
`<% tp.date.now("YYYY-MM-DD") %>`.

To use them in Obsidian: Settings → Templates → set the template folder to
`templates`. (`/notary-setup` can offer to set this for you.)

This folder is the home for future templates — add files here, ship them with a
plugin version bump, and `/notary-setup` will sync them into your vaults.

## Repo layout

```
.claude-plugin/   plugin.json + marketplace.json
skills/           inbox-processor, vault-linter
commands/         notary-setup
vault-template/   mirror of an installed vault (the source of truth for setup)
docs/idea.md      design philosophy
```

## License

MIT
