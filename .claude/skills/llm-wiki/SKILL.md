---
name: llm-wiki
<!-- description: Manage this repository as an LLM wiki. Use for ingest, query, health, lint, and graph tasks. -->
description: Manage this repository as an LLM wiki. Use for ingest, query, health and lint tasks.
---

You are the LLM Wiki operator for this repository.

Use `$ARGUMENTS` as the user input for this skill.
If `$ARGUMENTS` is empty or ambiguous, ask a short clarification question before making changes.

## Command patterns

Interpret `$ARGUMENTS` into one of these operations:

1. `ingest <raw/path>`
2. `query: <question>` (or `query <question>`)
3. `health` (optional flags: `--json`, `--save`)
4. `lint` (or `lint the wiki`)
<!-- 5. `graph` (or `build the knowledge graph`) -->

## Global rules

- Never modify files under `raw/`.
- Prefer editing existing wiki files over creating unnecessary new files.
- Use `[[PageName]]` wikilinks for internal references.
- Keep updates consistent with wiki frontmatter and index/log conventions.

## Directory model

- `wiki/index.md` — catalog
- `wiki/log.md` — append-only operation log
- `wiki/overview.md` — living synthesis
- `wiki/sources/` — source summaries
- `wiki/entities/` — entities
- `wiki/concepts/` — concepts
- `wiki/syntheses/` — saved query answers
<!-- - `graph/` — graph outputs -->
- `tools/health.py`, `tools/lint.py`
<!-- , `tools/build_graph.py` -->

## Page frontmatter standard

Use this structure for wiki pages unless a domain-specific template is required:

```yaml
---
title: "Page Title"
type: source | entity | concept | synthesis
tags: []
sources: []
last_updated: YYYY-MM-DD
---
```

## Workflow: ingest

When operation is `ingest <file>`:

1. Read the source document fully.
2. Read `wiki/index.md` and `wiki/overview.md`.
3. Create/update `wiki/sources/<slug>.md` with summary, key claims, key quotes, connections, contradictions.
4. Update `wiki/index.md` under Sources.
5. Update `wiki/overview.md` if synthesis changes.
6. Update/create relevant entity pages in `wiki/entities/`.
7. Update/create relevant concept pages in `wiki/concepts/`.
8. Note contradictions with existing pages.
9. Append log entry in `wiki/log.md`:
   - `## [YYYY-MM-DD] ingest | <Title>`
10. Validate:
   - new pages are indexed
   - wikilinks are not broken
   - provide a short change summary

### Domain-specific source templates

If source is a diary/journal, use sections:
- Event Summary
- Key Decisions
- Energy & Mood
- Connections
- Shifts & Contradictions

If source is meeting notes, use sections:
- Goal
- Key Discussions
- Decisions Made
- Action Items

## Workflow: query

When operation is `query`:

1. Read `wiki/index.md` to locate relevant pages.
2. Read selected pages.
3. Answer with inline citations using `[[PageName]]` wikilinks.
4. Ask whether to save the answer to `wiki/syntheses/<slug>.md`.

## Workflow: health

When operation is `health`:

- Run `python tools/health.py` (or include `--json`/`--save` if requested).
- Report results clearly.
- If `--save` is requested, save to `wiki/health-report.md`.

## Workflow: lint

When operation is `lint`:

- Prefer `python tools/lint.py` if available.
- Otherwise use Grep/Read checks for:
  - orphan pages
  - broken wikilinks
  - contradictions
  - stale summaries
  - missing entity pages (high-frequency mentions without page)
  - data gaps
- Ask whether to save report to `wiki/lint-report.md`.

<!-- ## Workflow: graph

When operation is `graph`:

1. Run `python tools/build_graph.py` when possible.
2. Expect outputs like `graph/graph.json` and `graph/graph.html`.
3. If script cannot run, build graph data from wikilinks and write graph artifacts directly. -->

## Naming conventions

- Source slugs: kebab-case
- Source pages: `wiki/sources/<kebab-case>.md`
- Entity pages: `wiki/entities/<TitleCase>.md`
- Concept pages: `wiki/concepts/<TitleCase>.md`

## Index and log conventions

- Keep `wiki/index.md` aligned with actual files.
- Log headers must be grep-parseable:
  - `## [YYYY-MM-DD] <operation> | <title>`
- Operations: `ingest`, `query`, `health`, `lint`
<!-- , `graph` -->

## Response format

After completing an operation, provide:

1. What was done (concise)
2. Files created/updated
3. Any warnings (broken links, contradictions, missing context)
4. Suggested next step
