---
name: multi-agent-patterns
description: This skill should be used when the user asks to "design multi-agent systems", "choose between orchestrator and swarm", "build agent pipelines", "coordinate multiple agents", or discusses supervisor patterns, peer-to-peer agent communication, context isolation across agents, or agent handoff protocols. Provides architectural patterns for multi-agent systems with context isolation as the primary design principle.
---

# Multi-Agent Architecture Patterns

Multi-agent architectures distribute work across multiple language model instances, each with its own context window. When designed well, this distribution enables capabilities beyond single-agent limits. When designed poorly, it introduces coordination overhead that negates benefits. The critical insight is that sub-agents exist primarily to isolate context, not to anthropomorphize role division.

## When to Activate

Activate this skill when:
- Single-agent context limits constrain task complexity
- Tasks decompose naturally into parallel subtasks
- Different subtasks require different tool sets or system prompts
- Building systems that must handle multiple domains simultaneously
- Scaling agent capabilities beyond single-context limits
- Designing production agent systems with multiple specialized components

## Core Concepts

Use multi-agent patterns when a single agent's context window cannot hold all task-relevant information. Context isolation is the primary benefit — each agent operates in a clean context without accumulated noise from other subtasks, preventing the telephone game problem where information degrades through repeated summarization.

Choose among three dominant patterns based on coordination needs, not organizational metaphor:

- **Supervisor/orchestrator** — Use for centralized control when tasks have clear decomposition and human oversight matters. A single coordinator delegates to specialists and synthesizes results.
- **Peer-to-peer/swarm** — Use for flexible exploration when rigid planning is counterproductive. Any agent can transfer control to any other through explicit handoff mechanisms.
- **Hierarchical** — Use for large-scale projects with layered abstraction (strategy, planning, execution). Each layer operates at a different level of detail with its own context structure.

Design every multi-agent system around explicit coordination protocols, consensus mechanisms that resist sycophancy, and failure handling that prevents error propagation cascades.

## Detailed Topics

### Why Multi-Agent Architectures

**The Context Bottleneck**
Reach for multi-agent architectures when a single agent's context fills with accumulated history, retrieved documents, and tool outputs to the point where performance degrades. Recognize three degradation signals: the lost-in-middle effect, attention scarcity, and context poisoning.

Partition work across multiple context windows so each agent operates in a clean context focused on its subtask. Aggregate results at a coordination layer without any single context bearing the full burden.

**The Token Economics Reality**
Budget for substantially higher token costs. Production data shows multi-agent systems run at approximately 15x the token cost of a single-agent chat:

| Architecture | Token Multiplier | Use Case |
|--------------|------------------|----------|
| Single agent chat | 1x baseline | Simple queries |
| Single agent with tools | ~4x baseline | Tool-using tasks |
| Multi-agent system | ~15x baseline | Complex research/coordination |

Research on the BrowseComp evaluation found that three factors explain 95% of performance variance: token usage (80% of variance), number of tool calls, and model choice. Prioritize model selection alongside architecture design — upgrading to better models often provides larger performance gains than doubling token budgets.

**The Parallelization Argument**
Assign parallelizable subtasks to dedicated agents with fresh contexts rather than processing them sequentially in a single agent. A research task requiring searches across multiple independent sources benefits from parallel execution. Total real-world time approaches the duration of the longest subtask rather than the sum of all subtasks.

### Architectural Patterns

**Pattern 1: Supervisor/Orchestrator**
Deploy a central agent that maintains global state and trajectory, decomposes user objectives into subtasks, and routes to appropriate workers.

```
User Query -> Supervisor -> [Specialist, Specialist, Specialist] -> Aggregation -> Final Output
```

Choose this pattern when: tasks have clear decomposition, coordination across domains is needed, or human oversight is important.

Expect these trade-offs: strict workflow control and easier human-in-the-loop interventions, but the supervisor context becomes a bottleneck, supervisor failures cascade to all workers, and the "telephone game" problem emerges.

**The Telephone Game Problem and Solution**
Supervisor architectures initially perform approximately 50% worse than optimized versions due to the telephone game problem (LangGraph benchmarks). Fix this by implementing a `forward_message` tool that allows sub-agents to pass responses directly to users, bypassing supervisor synthesis when appropriate.

**Pattern 2: Peer-to-Peer/Swarm**
Remove central control and allow agents to communicate directly based on predefined protocols. Any agent transfers control to any other through explicit handoff mechanisms.

```python
def transfer_to_agent_b():
    return agent_b  # Handoff via function return
```

Choose this pattern when: tasks require flexible exploration, rigid planning is counterproductive, or requirements emerge dynamically.

Expect these trade-offs: no single point of failure and effective breadth-first scaling, but coordination complexity increases with agent count.

**Pattern 3: Hierarchical**
Organize agents into layers of abstraction: strategy (goal definition), planning (task decomposition), and execution (atomic tasks).

```
Strategy Layer -> Planning Layer -> Execution Layer
```

Choose this pattern when: projects have clear hierarchical structure or tasks require both high-level planning and detailed execution.

### Context Isolation as Design Principle

Treat context isolation as the primary purpose of multi-agent architectures. Each sub-agent should operate in a clean context window focused on its subtask without carrying accumulated context from other subtasks.

**Isolation Mechanisms**

- **Full context delegation** — Share the planner's entire context with the sub-agent. Use for complex tasks where the sub-agent needs complete understanding. Note: this partially defeats the purpose of context isolation.
- **Instruction passing** — Create instructions via function call; the sub-agent receives only what it needs. Default choice. Maintains isolation and limits sub-agent context.
- **File system memory** — Agents read and write to persistent storage. Use for complex tasks requiring shared state. Avoids context bloat from shared state passing.

### Consensus and Coordination

**Avoid simple majority voting** — it treats hallucinations from weak models as equal to reasoning from strong models.

**Weighted Voting**: Weight agent votes by confidence or expertise. Agents with higher confidence should carry more weight in final decisions.

**Debate Protocols**: Structure agents to critique each other's outputs over multiple rounds. Adversarial critique often yields higher accuracy than collaborative consensus. Guard against sycophantic convergence.

## Practical Guidance

### Failure Modes and Mitigations

**Failure: Supervisor Bottleneck**
The supervisor accumulates context from all workers, becoming susceptible to saturation.
Mitigate by constraining worker output schemas so workers return only distilled summaries.

**Failure: Coordination Overhead**
Agent communication consumes tokens and introduces latency. Complex coordination can negate parallelization benefits.
Mitigate by minimizing communication through clear handoff protocols. Measure whether multi-agent coordination actually saves time.

**Failure: Divergence**
Agents pursuing different goals without central coordination drift from intended objectives.
Mitigate by defining clear objective boundaries. Set time-to-live limits on agent execution.

**Failure: Error Propagation**
Errors in one agent's output propagate to downstream agents that consume that output.
Mitigate by validating agent outputs before passing to consumers. Add a verification agent that cross-checks critical outputs.

## Examples

**Example 1: Research Team Architecture**
```text
Supervisor
├── Researcher (web search, document retrieval)
├── Analyzer (data analysis, statistics)
├── Fact-checker (verification, validation)
└── Writer (report generation, formatting)
```

**Example 2: Handoff Protocol**
```python
def handle_customer_request(request):
    if request.type == "billing":
        return transfer_to(billing_agent)
    elif request.type == "technical":
        return transfer_to(technical_agent)
    elif request.type == "sales":
        return transfer_to(sales_agent)
    else:
        return handle_general(request)
```

## Guidelines

1. Design for context isolation as the primary benefit of multi-agent systems
2. Choose architecture pattern based on coordination needs, not organizational metaphor
3. Implement explicit handoff protocols with state passing
4. Use weighted voting or debate protocols for consensus
5. Monitor for supervisor bottlenecks and implement checkpointing
6. Validate outputs before passing between agents
7. Set time-to-live limits to prevent infinite loops
8. Test failure scenarios explicitly

## Gotchas

1. **Supervisor bottleneck scaling**: Supervisor context pressure grows non-linearly with worker count. At 5+ workers, set a hard cap (3-5) and add a second supervisor tier rather than overloading one.

2. **Token cost underestimation**: Multi-agent runs cost approximately 15x baseline. Teams consistently underbudget because they estimate per-agent costs without accounting for coordination overhead. Budget for 15x and treat anything less as a bonus.

3. **Sycophantic consensus**: Agents in debate patterns tend to converge on agreeable answers, not correct ones. Counter this by assigning explicit adversarial roles and requiring agents to state disagreements before convergence is allowed.

4. **Agent sprawl**: Adding more agents past 3-5 shows diminishing returns and increases coordination overhead. Start with the minimum viable number and add only when a clear context isolation benefit exists.

5. **Telephone game in message-passing**: Information degrades through repeated summarization. Use filesystem coordination instead of message-passing for state that multiple agents need to access faithfully.

6. **Error propagation cascades**: One agent's hallucination becomes another agent's "fact." Add validation checkpoints between agents and never trust upstream output without verification.

7. **Over-decomposition**: Splitting tasks too finely creates more coordination overhead than the task itself. Decompose only when subtasks genuinely benefit from separate contexts.

8. **Missing shared state**: Agents operating without a shared filesystem or state store duplicate work and produce inconsistent outputs. Establish shared persistent storage before building multi-agent workflows.

## Integration

This skill builds on context-fundamentals and context-degradation. It connects to:

- memory-systems - Shared state management across agents
- tool-design - Tool specialization per agent
- context-optimization - Context partitioning strategies
- latent-briefing - KV-cache trajectory handoff between orchestrator and worker

## References

Related skills in this collection:
- context-fundamentals - Context window mechanics before designing agent partitioning
- memory-systems - When agents need to share state across context boundaries
- context-optimization - When individual agent contexts are too large

External resources:
- LangGraph Documentation - For graph-based multi-agent workflows
- AutoGen Framework - For conversational GroupChat patterns
- CrewAI Documentation - For role-based hierarchical agent processes

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
