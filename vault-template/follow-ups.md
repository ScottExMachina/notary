# Open Follow-ups

Every unchecked `- [ ]` item from the `## Follow-ups` section of your filed notes,
grouped by note. Check one off in its source note and it drops from this list.

```dataview
TASK
FROM "notes"
WHERE !completed AND meta(section).subpath = "Follow-ups"
GROUP BY file.link
```

---

_Powered by the [Dataview](https://github.com/blacksmithgu/obsidian-dataview)
community plugin. If the block above shows as code instead of a checklist, enable
Dataview in Obsidian: Settings → Community plugins. The query scopes to the
`## Follow-ups` section of notes in `notes/`; if you renamed that folder in
`notary.config.md`, update the `FROM "notes"` line to match._
