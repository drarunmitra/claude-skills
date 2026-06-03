# claude-skills

Personal [Claude Code](https://claude.com/claude-code) skills for academic research
writing in public health and medical research. Each subfolder is one self-contained
skill (a single `SKILL.md`).

| Skill | What it does |
|-------|--------------|
| [`research-writer`](research-writer/) | Drafts, revises, and structures research writing (theses, manuscripts, grants, reports). Aggressively removes AI-generated bloat: em-dashes (hard rule), significance language, filler. |
| [`zotero-cite`](zotero-cite/) | Inserts and verifies citations in Quarto/markdown using Zotero + Better BibTeX as the source of truth. Converts manual citations to citekeys, adds papers by DOI/PDF. |

These complement each other but stay separate skills: Claude invokes `research-writer`
while drafting prose and `zotero-cite` when handling references. (For literature search
and synthesis, install the `claude-scientific-writer` plugin's `literature-review` skill
from its marketplace — it is third-party and not bundled here.)

## Install on a new machine

Clone the repo, then symlink each skill into `~/.claude/skills/` so a single `git pull`
keeps every machine in sync:

```bash
git clone https://github.com/drarunmitra/claude-skills ~/claude-skills
mkdir -p ~/.claude/skills
ln -s ~/claude-skills/research-writer ~/.claude/skills/research-writer
ln -s ~/claude-skills/zotero-cite     ~/.claude/skills/zotero-cite
```

Claude Code auto-discovers anything under `~/.claude/skills/` (symlinks included).

## Update workflow

Edit any `SKILL.md`, then from `~/claude-skills`:

```bash
git commit -am "describe change" && git push
```

On other machines: `git -C ~/claude-skills pull` updates all skills at once.

## Runtime prerequisites

- **zotero-cite** needs Zotero running with the Better BibTeX extension (JSON-RPC on
  `localhost:23119`). Optionally the `zotero-mcp` server for richer search. These are
  per-machine runtime dependencies, not part of this repo.
