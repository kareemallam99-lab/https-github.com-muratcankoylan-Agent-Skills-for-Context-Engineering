# User Memory — Context Engineering Directives

This file is a portable copy of the directives that should live in your local `~/.claude/CLAUDE.md`. Copy the contents below into that file on your machine so they apply in every Claude Code session.

**Why a separate file?** This repo's `CLAUDE.md` documents the project itself. User memory belongs in your home directory so it follows you across all projects.

## How to install

1. Open a terminal on your local machine.
2. Run: `mkdir -p ~/.claude && code ~/.claude/CLAUDE.md` (or use `nano`, `vim`, or any editor).
3. Paste the content below into the file and save.

If `~/.claude/CLAUDE.md` already exists, append the content below to it instead of overwriting.

---

## Context Compression

Apply the `context-compression` skill (from the context-engineering plugin) in every session:

- Trigger compaction at **70-80% context utilization**, not when the window fills.
- Use **structured summaries** with mandatory sections: Session Intent, Files Modified, Decisions Made, Current State, Next Steps. Missing sections = visible omissions = recoverable; invisible omissions are not.
- Maintain a **separate artifact index** for file modifications — summaries lose file paths, function names, and error codes first. Track these explicitly outside the summary.
- Optimize for **tokens-per-task, not tokens-per-request**. Aggressive compression that loses decision rationale forces costly re-exploration.
- For coding work, prefer **sliding-window compression** (retain last ~20 turns verbatim, compress older) over session-wide regeneration.
- Validate compression via **probe testing** — after compacting, verify the agent can answer task-specific questions about prior state without re-fetching.

## Excel (.xlsx) Files

When processing Excel files, apply context-compression principles:

- **Never load full sheets into context.** Use openpyxl/pandas to read structure first (sheet names, shape, column headers), then load only the rows/columns the task needs.
- For inspection: report `df.shape`, `df.columns.tolist()`, `df.head(5)` and `df.dtypes` instead of dumping the whole DataFrame.
- For transformations: stream row-by-row or chunk-by-chunk; write intermediate results to disk rather than holding them in context.
- For multi-sheet workbooks: treat each sheet as a separate artifact; index sheet names + shapes upfront, load sheet contents on demand.
- Preserve **artifact references** (file path, sheet name, cell ranges modified) in any summary — these are the file-tracking dimension that compression loses first.
- Default tools: `pandas.read_excel(..., sheet_name=None, nrows=N)` for previewing, `openpyxl` for cell-level edits without rewriting the whole file.

## Context Optimization

Apply the `context-optimization` skill (from the context-engineering plugin) in every session. Apply techniques in risk order — measure baseline first:

### 1. KV-Cache Optimization (zero risk, immediate savings)
- Place **stable content first** in every prompt: system prompts, tool definitions, project conventions, persistent instructions.
- Place **dynamic content last**: conversation history, tool outputs, user message.
- **Never modify the stable prefix between requests** — even whitespace changes invalidate cached blocks.
- Avoid cache invalidators: timestamps in system prompts, dynamic examples, user-specific data in the system prompt (move to user message), per-request model IDs.
- Target **70%+ cache hit rate**; below 50% indicates structural problems with prompt ordering.

### 2. Observation Masking (low risk, high savings for tool-heavy work)
- Tool outputs consume up to **83.9% of agent trajectory tokens** — mask processed outputs once their immediate purpose is fulfilled.
- Replace verbose outputs with compact references: `[search_results_1: 47 results — key findings: X, Y, Z. Full results in scratch/search_1.json]`.
- Retain the **5 most recently accessed file contents** in full; compress or evict older ones.
- Do **not** mask: the active task context, recent outputs being actively reasoned over, or anything during debugging sessions.

### 3. Compaction (moderate risk, requires validation)
- Trigger at **70-80% context utilization** (proactive), not when the window fills.
- Target **50-70% token reduction with < 5% quality degradation** — not maximum compression.
- Freeze the active working set during compaction; compress only historical turns.
- Validate via **probe testing**: ask task-specific questions that require prior history.

### 4. Context Partitioning (highest complexity, last resort)
- Reserve for tasks where context **genuinely exceeds 60% of the window**.
- Justified only when: 3+ independent subtasks, significant parallelism gains, fundamentally different tool sets or system prompts per subtask.
- Budget **2,000-5,000 tokens per sub-agent boundary** for coordination overhead — partitioning often loses money for small tasks.

### Excel-specific application
- KV-cache: keep Excel-handling instructions in the stable system prefix; put per-file paths and sheet names in the user message.
- Observation masking: after extracting needed values from a sheet, mask the full DataFrame dump and retain only the extracted result + a path reference (`[sheet_summary: 1,200 rows × 8 cols, key cols: ..., source: file.xlsx#Sheet1]`).
- Compaction: when a workbook analysis spans many turns, compress historical sheet inspections into a structured index (sheet name → shape → key columns → notable findings) and keep the active sheet's working data in full.
