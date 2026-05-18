# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Agent Skills for Context Engineering — an open collection of 14 Agent Skills teaching context engineering principles for production AI agent systems. Skills are platform-agnostic (Claude Code, Cursor, GitHub Copilot, any Open Plugins-conformant tool).

Context engineering is the discipline of curating everything that enters a model's context window (system prompts, tool definitions, retrieved documents, message history, tool outputs) to maximize signal within limited attention budget.

## Repository Structure

- `skills/` — 14 skill directories, each containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and optional `references/` and `scripts/` subdirectories
- `examples/` — Complete demonstration projects (digital-brain-skill, llm-as-judge-skills, book-sft-pipeline, x-to-book-system, interleaved-thinking)
- `docs/` — Research materials and reference documentation
- `researcher/` — Research output examples
- `template/SKILL.md` — Canonical skill template (use when creating new skills)
- `SKILL.md` (root) — Collection-level metadata and skill map
- `.claude-plugin/marketplace.json` — Claude Code marketplace manifest (5 bundled plugins)
- `.plugin/plugin.json` — Open Plugins format manifest (v2.0.0)

## Build & Test Commands

No top-level build system. Individual example projects have their own tooling:

### examples/llm-as-judge-skills (TypeScript, Node >= 18)
```
cd examples/llm-as-judge-skills
npm install
npm run build        # tsc
npm test             # vitest (19 tests)
npm run lint         # eslint
npm run format       # prettier
npm run typecheck    # tsc --noEmit
```

### examples/interleaved-thinking (Python >= 3.10)
```
cd examples/interleaved-thinking
pip install -e ".[dev]"
pytest               # pytest + pytest-asyncio
ruff check .         # linting (100 char line length)
```

### examples/digital-brain-skill (Node.js)
```
cd examples/digital-brain-skill
npm run setup
npm run weekly-review
npm run content-ideas
npm run stale-contacts
```

## Skill Authoring Rules

When creating or editing skills:

1. **SKILL.md must stay under 500 lines** — move detailed content to `references/` directory
2. **YAML frontmatter is required** — must include `name` and `description` fields
3. **Folder naming**: lowercase with hyphens (e.g., `context-fundamentals`)
4. **Write in third person** — descriptions are injected into system prompts; inconsistent POV causes discovery issues
5. **Platform-agnostic** — no vendor-locked examples or platform-specific tool names without abstraction
6. **Token-conscious** — challenge each paragraph: "Does Claude really need this?" Assume advanced audience
7. **Include a Gotchas section** — experience-derived failure modes are the highest-signal content in any skill
8. **Update root README.md** when adding new skills
9. **Update marketplace/plugin manifests** when adding skills (`.claude-plugin/marketplace.json`, `.plugin/plugin.json`)

## Plugin Architecture

All 14 skills are distributed as a single plugin (`context-engineering`) in the marketplace manifest. This avoids cache duplication — Claude Code caches each plugin's `source` directory separately, so multiple plugins pointing to `source: "./"` would each cache a full copy of the repo.

Progressive disclosure pattern: only skill names/descriptions load at startup; full content loads on activation.

## Key Design Principles

- **Context quality over quantity** — attention scarcity and lost-in-middle phenomenon mean more context is not always better
- **Sub-agents isolate context** — they exist to manage attention budget, not simulate org roles
- **Skills reference each other** — use plain text skill names (not links) in Integration sections to avoid cross-directory reference issues
- **Examples use Python pseudocode** — conceptual demonstrations that work across environments, not production-ready implementations

## Active Directives (auto-applied in every session in this repo)

### Context Compression
Apply context-compression in every session:
- Trigger compaction at 70-80% context utilization, not when the window fills.
- Use structured summaries with sections: Session Intent, Files Modified, Decisions Made, Current State, Next Steps.
- Maintain a separate artifact index for file paths, function names, error codes — summaries lose these first.
- Optimize for tokens-per-task, not tokens-per-request.
- For coding work, retain the last ~20 turns verbatim and compress older history.

### Excel (.xlsx) Files
- Never load full sheets into context. Read structure first (sheet names, shape, column headers), then load only what's needed.
- For inspection: report `df.shape`, `df.columns`, `df.head(5)`, `df.dtypes` instead of dumping the whole DataFrame.
- For transformations: stream row-by-row; write intermediate results to disk.
- Default tools: `pandas.read_excel(..., sheet_name=None, nrows=N)` for previewing, `openpyxl` for cell-level edits.

### Context Optimization
Apply in risk order:
1. **KV-cache**: stable content (system prompt, tools, conventions) first; dynamic content (history, user message) last. Never modify the stable prefix. No timestamps in system prompts. Target 70%+ cache hit rate.
2. **Observation masking**: replace verbose tool outputs with compact references once processed. Retain the 5 most recently accessed files in full.
3. **Compaction**: trigger at 70-80% utilization. Target 50-70% reduction with <5% quality loss. Validate via probe testing.
4. **Partitioning**: only when context exceeds 60% of window and there are 3+ truly independent subtasks. Budget 2k-5k tokens per sub-agent boundary.
