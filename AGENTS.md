# AGENTS.md

Guidelines for humans and AI agents modifying this directory. It is the Project Standalone knowledge base (UW Reality Labs standalone XR headset). It is edited in Obsidian and must also render correctly on GitHub and in plain Markdown viewers.

## Structure
- Flat directory. One concept per note; the filename is the note title (`Hardware plan.md` → `# Hardware plan`).
- `Project Standalone.md` is the hub. Every note links back to it (directly or via its parent note), and every new note gets a row in the hub's **Map** table.
- Images and other binaries go in `attachments/`.
- `AGENTS.md` is the only non-note file at the top level.

## Links
Use standard Markdown links with relative paths. **Do not use Obsidian wikilinks** (`[[...]]`) or embeds (`![[...]]`); they do not render on GitHub.

| Target | Syntax |
|---|---|
| Another note | `[Hardware plan](Hardware%20plan.md)` |
| Note heading | `[Display selection](Display.md#what-selects-the-display)` (GitHub anchor: lowercase, spaces → `-`, punctuation dropped) |
| Image | `![Data flow](attachments/Monado%20driver%20data%20flow.png)` |
| External | `[Monado docs](https://monado.pages.freedesktop.org/monado/)` |

- Encode spaces in paths as `%20`. Do not use angle-bracket destinations.
- Link text can differ from the note name when it reads better in the sentence.
- Inside tables, links need no escaping (there is no `|` in the syntax).
- Obsidian settings for anyone editing in Obsidian: *Settings → Files and links* → **Use [[Wikilinks]]: off**, **New link format: Relative path to file**. Obsidian then inserts compatible links automatically.

## Note format
Every note starts with YAML frontmatter:
```yaml
---
tags: [area, topic]
aliases: [other names this note is known by]
status: draft        # draft | reviewed
---
```
Then:
1. `# Title` matching the filename.
2. A one-line context sentence linking the parent note (e.g. `Part of [Project Standalone](Project%20Standalone.md).`).
3. Body sections. Prefer tables for interfaces, contracts, comparisons and component lists; short prose otherwise.
4. `## Open questions` as the last section when anything is undecided.

## Content rules
- **Decisions vs proposals.** Distinguish what the team has decided from recommendations (including AI-generated analysis). Decisions are stated plainly; proposals say "proposed", "suggested" or sit under open questions. Never promote a proposal to a decision without a team member confirming it.
- **Supersession.** When a decision changes, update the note in place and, if useful, record the old value briefly ("originally SC035HGS"). Do not leave contradictory statements across notes; search the directory for the old value.
- **No duplication.** State a fact in one note and link to it from others.
- **Status.** New or substantially AI-written notes are `draft`. Only a team member sets `reviewed`. An agent making substantive edits to a `reviewed` note sets it back to `draft`.
- **Ownership boundaries.** Hardware below the kernel (FPGA firmware, sync, hardware timestamping) is owned by the hardware lead. Record their decisions as theirs; do not specify their implementation as settled.
- **Sources.** Link upstream docs and code paths (e.g. `src/xrt/auxiliary/tracking/t_tracker_slam.cpp`) for claims about Monado or other external code. Note that external interfaces may change.
- Style: concise, direct, no filler. Code identifiers in backticks.

## Making changes
- **Renaming a note:** rename the file, update the `# Title`, add the old name to `aliases`, and update every link to it:
  `grep -rn 'Old%20name.md' *.md`
- **Deleting a note:** do not delete. Ask a team member; at most mark it `status: draft` and note why at the top.
- **Adding a note:** follow the format above, link it from the hub Map and from at least one related note.
- **After any edit, check links:**
  ```sh
  # wikilinks (should print nothing)
  grep -rn '\[\[' --include='*.md' --exclude=AGENTS.md .
  # broken relative links (should print nothing)
  python3 -c "import re,os,urllib.parse,glob
  for f in set(glob.glob('*.md')) - {'AGENTS.md'}:
      for t in re.findall(r'\]\(([^)#]+?)(?:#[^)]*)?\)', open(f).read()):
          if '://' not in t and not os.path.exists(urllib.parse.unquote(t)): print(f, '->', t)"
  ```
- Keep commits scoped to one change, with a message describing what changed in the knowledge base.
