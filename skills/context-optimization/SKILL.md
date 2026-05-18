---
name: context-optimization
description: This skill should be used when the user asks to "reduce token costs", "optimize context", "implement KV-cache optimization", "apply observation masking", "design context partitioning", or discusses extending effective context capacity through caching, masking, compaction, or partitioning strategies. Provides concrete techniques for maximizing context efficiency.
---

# Context Optimization Techniques

Extend effective context capacity through four primary strategies: KV-cache optimization, observation masking, compaction, and context partitioning. Context quality matters more than quantity — optimize the signal-to-noise ratio before expanding the context window. Each technique has zero-to-low quality risk when applied correctly and measurable cost or latency benefits.

## When to Activate

Activate this skill when:
- Token costs are too high for production workloads
- Implementing KV-cache strategies to reduce inference latency
- Designing observation masking for tool-heavy agents
- Choosing between compaction and partitioning strategies
- Measuring and optimizing context efficiency in existing systems

## Core Concepts

Measure before optimizing. Applying techniques without baseline measurements produces unknown ROI and can introduce quality regressions that are hard to attribute. Establish token cost per task, cache hit rate, and task success rate as minimum baselines before any optimization work.

Context quality degrades before cost becomes visible. Token cost is a lagging indicator — quality degradation from context bloat appears in agent behavior before it appears in billing dashboards. Monitor for quality signals (task success rate, reasoning coherence) alongside cost signals.

The four strategies are ordered by risk and complexity:
1. KV-cache optimization — zero quality risk, immediate savings
2. Observation masking — low risk, high token savings for tool-heavy agents
3. Compaction — moderate risk, requires quality validation
4. Context partitioning — highest complexity, reserve for genuine context overflow

## Detailed Topics

### KV-Cache Optimization

KV-cache optimization reorders prompt structure so inference engines reuse cached Key/Value tensors across requests. This is the cheapest optimization available: zero quality risk, immediate cost and latency savings, and no changes to agent behavior.

**How it works**: Inference engines cache the KV representations of prompt prefixes. When a subsequent request begins with an identical prefix, the engine reuses the cached computation rather than reprocessing from scratch. A 70%+ cache hit rate typically achieves 50%+ cost reduction.

**Structural requirements**:
- Place stable content (system prompts, tool definitions, persistent instructions) at the beginning of every prompt
- Place dynamic content (user messages, tool outputs, conversation history) at the end
- Never modify stable prefix content between requests — even whitespace changes invalidate cached blocks

**Common cache invalidators**:
- Timestamps in system prompts force cache misses on every request
- Dynamic examples that change each call
- User-specific data injected into the system prompt rather than the user message
- Model version identifiers that vary across requests

Target 70%+ cache hit rate for production agent workloads. Below 50% indicates structural problems with prompt ordering.

### Observation Masking

Observation masking replaces verbose tool outputs with compact references once their immediate purpose is fulfilled. Tool outputs typically constitute 83.9% of total tokens in agent trajectories — masking processed outputs is the highest-leverage optimization for tool-heavy agents.

**Masking lifecycle**:
1. Tool returns verbose output (e.g., 5,000-token API response)
2. Agent processes the output and extracts the relevant result
3. Replace the full output with a compact reference: `[search_results_1: 47 results for "context engineering" — key findings: X, Y, Z]`
4. Store the full output in external storage if re-access is needed

**What to mask**:
- Search results after the agent has identified relevant items
- File contents after the agent has extracted needed information
- API responses after the agent has parsed the relevant fields
- Error outputs after the agent has determined the recovery action

**What not to mask**:
- The most recently accessed file contents (retain the last 5)
- Active task context (current goal, constraints, next steps)
- Tool outputs the agent is actively reasoning over

Masking during debugging sessions hides necessary error details — disable masking or increase the retention window when debugging.

### Compaction

Compaction summarizes accumulated context when utilization exceeds 70%, then reinitializes with the summary. Target 50-70% token reduction with less than 5% quality degradation — not maximum compression.

**Compaction trigger points**:
- 70-80% context utilization (proactive trigger)
- Phase boundary completion (end of research, start of implementation)
- Task unit completion within a multi-task session

**Compaction process**:
1. Freeze the active working set (current task context, recent tool outputs)
2. Compress historical context into a structured summary
3. Reinitialize with: system prompt + compressed summary + active working set
4. Validate that task-critical information survived compression (probe test)

**Quality validation**: After compaction, ask the agent task-specific questions that require prior history. If the agent cannot answer correctly, the compression was too aggressive.

See context-compression for detailed compression method selection and structured summary templates.

### Context Partitioning

Context partitioning splits work across sub-agents with isolated contexts. Reserve this for tasks where context genuinely exceeds 60% of the window limit — coordination overhead carries real token costs that negate savings for smaller contexts.

**When partitioning is justified**:
- The task naturally decomposes into independent subtasks
- Individual subtask contexts would exceed 60% of the window
- Parallel execution reduces wall-clock time significantly
- Different subtasks require different tool sets or system prompts

**When partitioning is not justified**:
- The task involves fewer than 3 independent subtasks
- Subtasks have significant shared context
- Coordination costs exceed the context savings
- The model can handle the full context within its effective capacity

**Coordination cost accounting**: Each sub-agent interaction requires system prompt, task handoff, result synthesis, and error handling. Budget 2,000-5,000 tokens per sub-agent boundary in the coordinating context.

## Practical Guidance

### Optimization Sequencing

Apply optimizations in risk order:

1. **Audit and baseline** — measure current token cost, cache hit rate, task success rate
2. **KV-cache optimization** — restructure prompts for maximum cache reuse (zero risk)
3. **Observation masking** — identify the top 3 tool outputs by token volume and mask them (low risk)
4. **Compaction** — implement proactive compaction at 70-80% utilization (moderate risk, requires validation)
5. **Partitioning** — split only if the above steps are insufficient and the task genuinely parallelizes (highest complexity)

### Measuring Optimization Effectiveness

For each optimization applied:
- **KV-cache**: Track cache hit rate via provider APIs; target 70%+
- **Observation masking**: Measure tokens-per-task before and after; validate task success rate
- **Compaction**: Measure compression ratio and probe-test information retention
- **Partitioning**: Measure total tokens including coordination overhead; compare to single-agent baseline

## Examples

**Example 1: Prompt Structure for KV-Cache**
```markdown
# STABLE PREFIX (cached across requests)
[System prompt]
[Tool definitions]
[Project conventions]
[Persistent instructions]

# DYNAMIC SUFFIX (not cached — different each request)
[Conversation history]
[Current tool outputs]
[User message]
```

**Example 2: Observation Masking**
```python
# Before masking: 4,800 tokens of search results
tool_result = search_web("context engineering techniques")
# Returns: [full JSON with 47 results, 4,800 tokens]

# After masking: 85 tokens
masked_result = "[web_search_1: 47 results — top 3: (1) Lost in Middle paper, (2) Anthropic context guide, (3) RULER benchmark. Full results in scratch/search_1.json]"
```

## Guidelines

1. Measure baseline before optimizing — establish token cost per task and cache hit rate
2. Apply KV-cache optimization first — zero risk, immediate savings
3. Mask tool outputs after the agent processes them, not before
4. Set compaction triggers at 70-80% utilization, not 90%+
5. Validate compaction quality through probe-based testing
6. Reserve partitioning for tasks that genuinely overflow the effective context window
7. Account for coordination overhead when evaluating partitioning ROI

## Gotchas

1. **Whitespace in stable prefix invalidates cache blocks**: Even a trailing newline change in the system prompt forces a full cache miss on every request. Treat the stable prefix as immutable once deployed — changes require cache warm-up periods.

2. **Timestamps in system prompts destroy cache efficiency**: A system prompt containing `Current time: {datetime.now()}` creates a unique prefix on every request, achieving 0% cache hit rate. Move dynamic content to the user message, never the system prompt.

3. **Over-aggressive masking during debugging**: Masking tool outputs hides error details needed to diagnose failures. Implement a debug mode that disables masking or increases the retention window, and make it easy to toggle.

4. **Partitioning coordination costs exceed savings for small tasks**: A task that decomposes into 2 subtasks rarely benefits from partitioning — coordination overhead (handoff tokens, synthesis tokens) often exceeds the context savings. Apply partitioning only when subtask count is 3+ and tasks are genuinely independent.

5. **Compaction quality not validated**: Implementing compaction without probe-based testing creates invisible information loss. Agents resume tasks with missing context, make decisions based on incorrect historical state, and produce wrong outputs. Always validate compaction quality before deploying to production.

## Integration

This skill builds on context-fundamentals and context-degradation. It connects to:

- context-compression - Detailed compression method selection
- multi-agent-patterns - Context partitioning across agents
- latent-briefing - KV-cache sharing between orchestrator and worker agents

## References

Related skills in this collection:
- context-fundamentals - Foundation for understanding context mechanics
- context-compression - Detailed compression strategies for history management
- multi-agent-patterns - Partitioning implementation patterns
- latent-briefing - Advanced KV-cache sharing between agents

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
