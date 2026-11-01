# AGENTS.md

Guidelines for humans and AI agents modifying this directory. It is the Project Standalone knowledge base (UW Reality Labs standalone XR headset). It is edited in Obsidian and must also render correctly on GitHub and in plain Markdown viewers.

## Structure
| Location | Contents |
|---|---|
| Top level | Notes about the whole project: `Project Standalone.md` (hub), `Tracer bullet roadmap.md`. Plus `AGENTS.md` |
| `hardware/` | Physical components and below-kernel design: sensors, display panels and optics, sync |
| `platform/` | OS, kernel, distribution, board setup, toolchains |
| `runtime/` | Monado runtime internals we build on: sinks and the frame graph, capture, timestamping, compositor integration |
| `tracking/` | Perception: SLAM, hand and eye tracking, tracker interfaces |
| `emulation/` | Running the stack without real hardware: Monado's replay sources, simulated devices, kernel-level emulation |
| `standalone-driver/` | Documentation of the team's own code in the Monado fork (driver, builder, branches). Subfolders mirror topic folders, e.g. `standalone-driver/emulation/`; `standalone-driver/guides/` holds setup and how-to guides; `standalone-driver/task-guides/` holds task briefs for team members |
| `attachments/` | Images and other binaries |

- One concept per note; the filename is the note title (`Hardware plan.md` → `# Hardware plan`).
- Note filenames are unique across the whole tree, even in different folders.
- Folder names are lowercase, one word where possible (no spaces).
- Place a note in the folder of its primary subject. Cross-cutting notes link to each other rather than being duplicated.
- Notes about upstream Monado or third-party code go in the topic folders; notes about code the team wrote go under `standalone-driver/`, and name the branch and commit they describe.
- Add a new folder only when at least ~3 notes would go in it, or when a team member asks for it. When adding one, add a row to this table and to the hub's **Map** table.
- `Project Standalone.md` is the hub. Every note links back to it (directly or via its parent note), and every new note gets listed in the hub's **Map** table under its folder.

## Links
Use standard Markdown links with paths **relative to the file containing the link**. **Do not use Obsidian wikilinks** (`[[...]]`) or embeds (`![[...]]`); they do not render on GitHub.

| Target | Syntax (from a note in `runtime/`) |
|---|---|
| Note in same folder | `[Sensor timestamping pipeline](Sensor%20timestamping%20pipeline.md)` |
| Note in another folder | `[Hardware plan](../hardware/Hardware%20plan.md)` |
| Top-level note | `[Project Standalone](../Project%20Standalone.md)` |
| Note heading | `[Display selection](../hardware/Display.md#what-selects-the-display)` (GitHub anchor: lowercase, spaces → `-`, punctuation dropped) |
| Image | `![Data flow](../attachments/Monado%20driver%20data%20flow.png)` |
| External | `[Monado docs](https://monado.pages.freedesktop.org/monado/)` |

- Encode spaces in paths as `%20`. Do not use angle-bracket destinations or root-absolute paths (`/hardware/...`).
- Link text can differ from the note name when it reads better in the sentence.
- Obsidian settings for anyone editing in Obsidian: *Settings → Files and links* → **Use [[Wikilinks]]: off**, **New link format: Relative path to file**. Obsidian then inserts compatible links and updates them when files move inside Obsidian.

## Note format
Every note starts with YAML frontmatter:
```yaml
---
tags: [area, topic]
aliases: [other names this note is known by]
status: draft        # draft | reviewed
author: "Name <email>"  # person who wrote it, or the model, e.g. "Claude Opus 5.5"
---
```
Then:
1. `# Title` matching the filename.
2. A one-line context sentence linking the parent note (e.g. `Part of [Project Standalone](Project%20Standalone.md).`).
3. Body sections. Prefer tables for interfaces, contracts, comparisons and component lists; short prose otherwise.
4. `## Open questions` as the last section when anything is undecided.

## Content rules
- **Authorship.** `author` records who wrote the note. Keep it when editing; if someone else substantially rewrites a note, change it to a list (`author: ["A", "B"]`).
- **Decisions vs proposals.** Distinguish what the team has decided from recommendations (including AI-generated analysis). Decisions are stated plainly; proposals say "proposed", "suggested" or sit under open questions. Never promote a proposal to a decision without a team member confirming it.
- **Supersession.** When a decision changes, update the note in place and, if useful, record the old value briefly ("originally SC035HGS"). Do not leave contradictory statements across notes; search the directory for the old value.
- **No duplication.** State a fact in one note and link to it from others.
- **Status.** New or substantially AI-written notes are `draft`. Only a team member sets `reviewed`. An agent making substantive edits to a `reviewed` note sets it back to `draft`.
- **Ownership boundaries.** Hardware below the kernel (FPGA firmware, sync, hardware timestamping) is owned by the hardware lead. Record their decisions as theirs; do not specify their implementation as settled.
- **Sources.** Link upstream docs and code paths (e.g. `src/xrt/auxiliary/tracking/t_tracker_slam.cpp`) for claims about Monado or other external code. Note that external interfaces may change.
- Style: concise, direct, no filler. Code identifiers in backticks.

## Making changes
- **Moving or renaming a note:** move it inside Obsidian if possible (it rewrites links). Otherwise, move the file, then fix every link to it and every relative link inside it (its `../` depth may change):
  `grep -rn 'Old%20name.md' --include='*.md' .`
  On rename, also update the `# Title` and add the old name to `aliases`. Update the hub Map.
- **Deleting a note:** do not delete. Ask a team member; at most mark it `status: draft` and note why at the top.
- **Adding a note:** pick its folder per **Structure**, follow the format above, list it in the hub Map, and link it from at least one related note.
- **After any edit, check links** (run from this directory; both should print nothing):
  ```sh
  # wikilinks
  grep -rn '\[\[' --include='*.md' --exclude=AGENTS.md .
  # broken relative links
  python3 -c "import re,os,urllib.parse,glob
  for f in set(glob.glob('**/*.md', recursive=True)) - {'AGENTS.md'}:
      for t in re.findall(r'\]\(([^)#]+?)(?:#[^)]*)?\)', open(f).read()):
          p = os.path.join(os.path.dirname(f), urllib.parse.unquote(t))
          if '://' not in t and not os.path.exists(p): print(f, '->', t)"
  ```
- Keep commits scoped to one change, with a message describing what changed in the knowledge base.
