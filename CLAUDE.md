# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

BCQuality is a curated knowledge base and skills library for Business Central (BC) development. It provides structured, machine-readable guidance that LLM-driven review agents consume — establishing a consistent quality bar across teams. It contains **content** (knowledge files and skills); agents that consume BCQuality live elsewhere (AL-Go, VS Code extensions, etc.).

The admission test for any knowledge file: *"If this file did not exist, would a modern LLM reviewing or generating BC code make a mistake this file would have prevented?"* Generic software-engineering advice does not belong here regardless of how sound it is.

## Validation commands

Run the frontmatter/structure validator locally (requires Python 3.12+ and `pyyaml`):

```bash
pip install pyyaml
python .github/scripts/validate_frontmatter.py --root .
```

Rebuild the knowledge index (requires PowerShell):

```bash
pwsh ./tools/Build-KnowledgeIndex.ps1
```

Validate the knowledge index generator:

```bash
pwsh ./.github/scripts/Test-KnowledgeIndex.ps1 -Root .
```

CI runs both on every PR to `main` via `.github/workflows/validate-frontmatter.yml` and `.github/workflows/knowledge-index.yml`.

## Repository structure

```
/skills/              # Global: entry.md (entry-point) + read.md, do.md, write.md (meta-skill contracts)
/microsoft/           # Microsoft-endorsed layer
│   /knowledge/<domain>/    # One .md per concern + optional .good.al / .bad.al sibling samples
│   /skills/                # Microsoft action skills (e.g. review/al-code-review.md)
/community/           # BC community layer (same structure)
/custom/              # Empty template for forks — never populated upstream
/.github/
│   /workflows/       # CI: validate-frontmatter, knowledge-index, guard-custom-layer, flag-new-top-level
│   /scripts/         # validate_frontmatter.py, Test-KnowledgeIndex.ps1
/tools/               # Build-KnowledgeIndex.ps1 — generates knowledge-index.json at repo root
```

Layer precedence (highest to lowest): `custom` > `community` > `microsoft`.

## Knowledge file format

Every knowledge file is a markdown file with mandatory YAML frontmatter and must stay under 100 lines.

### Required frontmatter (all six fields required, schema is locked)

```yaml
---
bc-version: [all]                       # or [26..28] or [26..] for open-ended
domain: performance                     # security | performance | ux | telemetry | ...
keywords: [query, filtering, partial]   # 3–10, lowercase kebab-case
technologies: [al]                      # explicit list — no [all] sentinel
countries: [w1]                         # [w1] or ISO alpha-2 codes (mutually exclusive)
application-area: [all]                 # [all] or specific areas (mutually exclusive)
---
```

### Required sections

- `## Description` — required; states the concern and why it matters (2–5 sentences)
- `## Best Practice` — optional but recommended
- `## Anti Pattern` — optional but recommended

**No fenced code blocks** in knowledge files. Code samples live as sibling files (`<slug>.good.al`, `<slug>.bad.al`) next to the article.

### Knowledge file placement

`<layer>/knowledge/<domain>/<slug>.md` — exactly four path segments. Filename is lowercase kebab-case. One concern per file.

## Action skill format

Action skills live under `<layer>/skills/**/*.md` and follow the DO meta-skill template (`skills/do.md`).

### Required frontmatter

```yaml
---
kind: action-skill
id: al-code-review          # lowercase kebab-case, unique within kind
version: 1                  # positive integer
title: AL code review
description: ...
inputs: [pr-diff]           # pr-diff | object-list | file-path | repository | telemetry-query
outputs: [findings-report]
# Optional filters: bc-version, technologies, countries, application-area
# Optional: sub-skills: [path/to/leaf.md]  — makes this a super-skill
---
```

### Required sections (in order)

`## Source` → `## Relevance` → `## Worklist` → `## Action` → `## Output`

A super-skill (one with `sub-skills` in frontmatter) composes leaf skills rather than evaluating knowledge files directly. CI rule R26 enforces that the `sub-skills` list exactly matches the `al-*-review.md` leaf files in the same directory.

## Agent flow architecture

1. Orchestrator triggers agent with a task context and the BCQuality URL.
2. Agent invokes `skills/entry.md` (the only hardcoded convention for orchestrators).
3. Entry runs Source → Relevance → Worklist → Action over action skills and returns a **dispatch record** (JSON) naming skills to invoke.
4. Before routing, Entry rebuilds `knowledge-index.json` from the live clone via `tools/Build-KnowledgeIndex.ps1` — this is a side step that must not affect the dispatch record.
5. Agent invokes each dispatched action skill, reading `skills/read.md` (READ) and `skills/do.md` (DO) on demand.
6. Each action skill produces a structured **findings-report** (JSON) consumed by the orchestrator without skill-specific parsing.

The knowledge index (`knowledge-index.json`) accelerates the Source step — agents use it for discovery instead of opening every file, but still open each worklisted article in full for its rule body. When the index is absent, skills fall back to path-based discovery.

## CI checks and their rules

The validator (`validate_frontmatter.py`) enforces named rules:

| Rule | What it checks |
|------|---------------|
| R01 | Valid UTF-8 and parseable frontmatter |
| R02 | All required keys present, no extras, none empty |
| R03–R08 | Field-level validation (bc-version, domain, keywords, technologies, countries, application-area) |
| R09 | `## Description` section present in knowledge files |
| R10 | No fenced code blocks in knowledge files |
| R11 | Knowledge file ≤ 100 lines |
| R12 | Filename is lowercase kebab-case |
| R13 | Knowledge file at correct path depth: `<layer>/knowledge/<domain>/<slug>.md` |
| R14 | Sample files match `<slug>.<kind>.<ext>` with a sibling `<slug>.md` |
| R15–R21 | Action-skill frontmatter and section validation |
| R22 | Meta-skill frontmatter validation |
| R23 | Entry-point frontmatter validation |
| R24 | Skill IDs unique within kind |
| R25 | `kind` matches file location |
| R26 | Super-skill `sub-skills` list matches sibling `al-*-review.md` leaves on disk exactly |

## Custom layer policy

`/custom/` is empty in the upstream `microsoft/BCQuality` repository. A PR that adds content to `/custom/` will be **automatically closed** by the `guard-custom-layer.yml` workflow. Custom content belongs in a fork. Verify before authoring: `git remote get-url origin` — if it points at `github.com/microsoft/BCQuality`, use `/community/knowledge/` or fork first.
