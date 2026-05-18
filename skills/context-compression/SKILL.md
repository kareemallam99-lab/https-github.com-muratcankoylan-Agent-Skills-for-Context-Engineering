---
name: context-compression
description: This skill should be used when the user asks to "compress conversation history", "summarize context", "manage long sessions", "apply compaction", or discusses strategies for reducing token usage while preserving information in long-running agent sessions. Provides production compression techniques for managing context growth.
---

# Context Compression Strategies

Compress conversation history to extend effective session length without losing task-critical information. Context compression addresses the fundamental tension in long-running agents: history grows unboundedly while context windows are fixed. The right strategy depends on session length, re-fetch cost, and how much interpretability the system requires.

## When to Activate

Activate this skill when:
- Managing long agent sessions where history dominates context
- Implementing compaction strategies for production systems
- Choosing between summarization approaches
- Designing context lifecycle management for agentic loops
- Debugging cases where historical context crowds out active instructions

## Core Concepts

Optimize for tokens-per-task, not tokens-per-request. Aggressive compression that loses file paths or decision rationale forces costly re-exploration, potentially wasting more tokens than the compression saved. Every compression decision trades information fidelity against token budget — make that trade explicitly.

Trigger compression at 70-80% context utilization, not when the window fills. Waiting until the window is full means the model is already degraded when compaction fires. Proactive compression preserves headroom for the active task.

File tracking is the weakest dimension of all compression methods. Modified files, created artifacts, and specific identifiers (function names, error codes) disappear in summarization unless explicitly preserved. Implement separate artifact indexing rather than relying on summarization alone.

## Detailed Topics

### Three Production Compression Methods

**Anchored Iterative Summarization**
Best for long sessions where file tracking and decision continuity matter. Maintains a structured summary document with dedicated sections (session intent, files modified, decisions made, current state, next steps). Updates only the newly-truncated portion of history rather than regenerating the full summary on each compaction cycle.

Use when: Sessions span many tool calls, the agent must reference prior decisions, or file modification history is critical to task completion.

Trade-offs: Higher per-compaction cost than opaque methods; summary sections can drift if not updated consistently.

**Opaque Compression**
Maximizes token savings (approaching 99% compression ratios) for short sessions with low re-fetching costs. Replaces conversation history with a dense summary blob without preserving structure or intermediate reasoning steps.

Use when: Sessions are short enough that re-fetching context is cheap, interpretability of intermediate states is not required, or storage cost of history outweighs re-fetch cost.

Trade-offs: Sacrifices interpretability; historical reasoning chains are unrecoverable from summaries; any missed information requires full context replay.

**Regenerative Full Summary**
Prioritizes readability with detailed structured summaries generated at clear phase boundaries (end of research phase, end of implementation phase). Produces high-quality summaries but accumulates cumulative detail loss across multiple cycles.

Use when: Phase boundaries are well-defined, humans need to inspect agent reasoning, or audit trails are required.

Trade-offs: Cumulative quality loss across compression cycles; expensive to generate complete summaries repeatedly.

### Structured Summary Templates

Structured summaries outperform free-form summaries because they function as checklists the summarizer must populate, making omissions visible. Every summary should include mandatory sections:

```markdown
## Session Intent
[What task is this session accomplishing?]

## Files Modified
[List of files touched, with what changed]

## Decisions Made
[Key architectural or implementation choices and their rationale]

## Current State
[Where the session is right now — what's done, what's in progress]

## Next Steps
[What the agent should do next]
```

Missing sections indicate information the compactor failed to capture — visible gaps are recoverable; invisible omissions are not.

### Artifact Trail Problem

File tracking is consistently the weakest dimension across all compression methods. Standard summarization loses:
- Exact file paths (compressed to vague references like "the config file")
- Function names and identifiers modified
- Error codes encountered and resolved
- Specific line numbers or locations

Implement a parallel artifact index — a separate, append-only log of file operations that compaction cannot overwrite. Maintain this alongside the compressed summary rather than depending on the summary to preserve it.

### Sliding-Window Strategies for Coding Agents

For coding agents that operate on a single codebase across many turns, sliding-window compression outperforms session-based compression. Retain the N most recent turns in full; compress older turns into a structured summary. This preserves recency bias (recent tool outputs are most relevant) while preventing unbounded growth.

Set N based on the average tool call count per logical task unit. If a typical implementation subtask takes 8-12 tool calls, retain the last 15-20 turns before compressing older history.

## Practical Guidance

### Compression Quality Evaluation

Evaluate compression quality through probe-based testing rather than traditional metrics:

1. After compression, ask the agent to answer specific questions about prior history
2. Check whether file modifications, decisions, and current state are accurately recalled
3. Verify that the agent can resume the task from the compressed state without re-fetching

Standard metrics (ROUGE, BLEU) do not predict task resumption quality. Probe testing does.

### Compaction Trigger Design

Implement compaction as a two-phase operation:
1. **Detection**: Monitor context utilization and trigger at 70-80% of the window
2. **Execution**: Compress history while preserving the active task context intact

Never compress the current active instruction set or the most recent tool outputs — these are the highest-signal content at the time of compaction. Compress historical turns, not the active working set.

## Examples

**Example 1: Structured Summary**
```markdown
## Session Intent
Implementing authentication middleware for the API gateway

## Files Modified
- src/middleware/auth.py — Added JWT validation logic
- src/config/settings.py — Added JWT_SECRET_KEY config
- tests/test_auth.py — Added 3 test cases for token validation

## Decisions Made
- Using PyJWT library (not python-jose) — better maintenance status
- Token expiry set to 24h for user tokens, 1h for service tokens
- Refresh token stored in HttpOnly cookie, not localStorage

## Current State
Core validation working. Rate limiting middleware not yet integrated.

## Next Steps
1. Integrate with rate_limiter.py
2. Add refresh token endpoint
3. Update API documentation
```

**Example 2: Sliding-Window Configuration**
```python
COMPRESSION_CONFIG = {
    "trigger_threshold": 0.75,  # 75% of context window
    "retain_recent_turns": 20,   # Keep last 20 turns verbatim
    "summary_target_tokens": 2000,  # Compressed summary budget
    "artifact_tracking": True,   # Separate file modification log
}
```

## Guidelines

1. Trigger compaction at 70-80% utilization, not when the window fills
2. Use structured templates with mandatory sections to prevent invisible omissions
3. Maintain a separate artifact index for file modifications
4. Evaluate compression quality through probe-based testing
5. Match compression method to session characteristics (length, re-fetch cost, interpretability needs)
6. Retain the active working set (current task context, recent outputs) verbatim during compaction
7. For coding agents, use sliding-window compression over session-based approaches

## Gotchas

1. **Over-compressing loses task-critical context**: Summaries that reduce 50K tokens to 500 tokens cannot preserve file paths, function names, or error codes. Losing this information forces the agent to rediscover it, spending more tokens than the compression saved. Target 50-70% reduction, not maximum compression.

2. **Compacting too late causes quality cliffs**: Waiting until context is 90%+ full means the agent is already degraded. The compaction itself then runs in a degraded state, producing lower-quality summaries. Set triggers at 70-80% utilization.

3. **Summary drift across cycles**: Each compaction cycle introduces errors that compound in subsequent cycles. A summary of a summary of a summary diverges progressively from original events. Limit compaction cycles for a single session or rebuild from raw history periodically.

4. **Re-compacting stable content wastes tokens**: If the session intent and early decisions haven't changed, regenerating summaries of that content wastes tokens on unchanged information. Use anchored iterative summarization that only reprocesses newly-truncated content.

5. **Missing the artifact trail**: File modifications are the first thing lost in summarization and the most expensive to recover. Teams consistently discover this failure mode in production when agents start re-implementing already-completed work. Implement separate artifact tracking before this happens.

## Integration

This skill builds on context-fundamentals and context-degradation. It connects to:

- context-optimization - Broader context management strategies including KV-cache optimization
- memory-systems - Persistent memory as an alternative to in-context compression
- evaluation - Evaluating compression quality in production

## References

Related skills in this collection:
- context-fundamentals - Foundation for understanding why compression is necessary
- context-degradation - Degradation patterns that compression mitigates
- context-optimization - Complementary optimization strategies

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
