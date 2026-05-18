---
name: context-degradation
description: This skill should be used when the user asks to "diagnose context problems", "fix lost-in-middle issues", "debug agent failures", "understand context poisoning", or mentions context degradation, attention patterns, context clash, context confusion, or agent performance degradation. Provides patterns for recognizing and mitigating context failures.
---

# Context Degradation Patterns

Diagnose and fix context failures before they cascade. Context degradation is not binary — it is a continuum that manifests through five distinct, predictable patterns: lost-in-middle, poisoning, distraction, confusion, and clash. Each pattern has specific detection signals and mitigation strategies. Treat degradation as an engineering problem with measurable thresholds, not an unpredictable failure mode.

## When to Activate

Activate this skill when:
- Agent performance degrades unexpectedly during long conversations
- Debugging cases where agents produce incorrect or irrelevant outputs
- Designing systems that must handle large contexts reliably
- Evaluating context engineering choices for production systems
- Investigating "lost in middle" phenomena in agent outputs
- Analyzing context-related failures in agent behavior

## Core Concepts

Structure context placement around the attention U-curve: beginning and end positions receive reliable attention, while middle positions suffer 10-40% reduced recall accuracy (Liu et al., 2023). This is not a model bug but a consequence of attention mechanics — the first token often acts as an "attention sink" that absorbs disproportionate attention budget, leaving middle tokens under-attended as context grows.

Treat context poisoning as a circuit breaker problem. Once a hallucination, tool error, or incorrect retrieved fact enters context, it compounds through repeated self-reference. A poisoned goals section causes every downstream decision to reinforce incorrect assumptions. Detection requires tracking claim provenance; recovery requires truncating to before the poisoning point or restarting with verified-only context.

Filter aggressively before loading context — even a single irrelevant document measurably degrades performance on relevant tasks. Models cannot "skip" irrelevant context; they must attend to everything provided, creating attention competition between relevant and irrelevant content. Move information that might be needed but is not immediately relevant behind tool calls instead of pre-loading it.

Isolate task contexts to prevent confusion. When context contains multiple task types or switches between objectives, models incorporate constraints from the wrong task, call tools appropriate for a different context, or blend requirements from multiple sources. Explicit task segmentation with separate context windows eliminates cross-contamination.

Resolve context clash through priority rules, not accumulation. When multiple correct-but-contradictory sources appear in context (version conflicts, perspective conflicts, multi-source retrieval with divergent facts), models cannot determine which applies. Mark contradictions explicitly, establish source precedence, and filter outdated versions before they enter context.

## Detailed Topics

### Lost-in-Middle: Detection and Placement Strategy

Place critical information at the beginning and end of context, never in the middle. The U-shaped attention curve means middle-positioned information suffers 10-40% reduced recall accuracy. For contexts over 4K tokens, this effect becomes significant.

Use summary structures that surface key findings at attention-favored positions. Add explicit section headers and structural markers — these help models navigate long contexts by creating attention anchors. When a document must be included in full, prepend a summary of its key points and append the critical conclusions.

Monitor for lost-in-middle symptoms: correct information exists in context but the model ignores it, responses contradict provided data, or the model "forgets" instructions given earlier in a long prompt.

### Context Poisoning: Prevention and Recovery

Validate all external inputs before they enter context. Tool outputs, retrieved documents, and model-generated summaries are the three primary poisoning vectors. Each introduces unverified claims that subsequent reasoning treats as ground truth.

Detect poisoning through these signals: degraded output quality on previously-successful tasks, tool misalignment (wrong tools or parameters), and hallucinations that persist despite explicit correction. When these cluster, suspect poisoning rather than model capability issues.

Recover by removing poisoned content, not by adding corrections on top. Truncate to before the poisoning point, restart with clean context preserving only verified information, or explicitly mark the poisoned section and request re-evaluation from scratch. Layering corrections over poisoned context rarely works — the original errors retain attention weight.

### Context Distraction: Curation Over Accumulation

Curate what enters context rather than relying on models to ignore irrelevant content. Research shows even a single distractor document triggers measurable performance degradation — the effect follows a step function, not a linear curve. Multiple distractors compound the problem.

Apply relevance filtering before loading retrieved documents. Use namespacing and structural organization to make section boundaries clear. Prefer tool-call-based access over pre-loading: store reference material behind retrieval tools so it enters context only when directly relevant to the current reasoning step.

### Context Confusion: Task Isolation

Segment different tasks into separate context windows. Context confusion is distinct from distraction — it concerns the model applying wrong-context constraints to the current task, not just attention dilution. Signs include responses addressing the wrong aspect of a query, tool calls appropriate for a different task, and outputs mixing requirements from multiple sources.

Implement clear transitions between task contexts. Use state management that isolates objectives, constraints, and tool definitions per task. When task-switching within a single session is unavoidable, use explicit "context reset" markers that signal which constraints apply to the current segment.

### Context Clash: Conflict Resolution Protocols

Establish source priority rules before conflicts arise. Context clash differs from poisoning — multiple pieces of information are individually correct but mutually contradictory (version conflicts, perspective differences, multi-source retrieval with divergent facts).

Implement version filtering to exclude outdated information before it enters context. When contradictions are unavoidable, mark them explicitly with structured conflict annotations: state what conflicts, which source each claim comes from, and which source takes precedence. Without explicit priority rules, models resolve contradictions unpredictably.

### Empirical Benchmarks and Thresholds

The RULER benchmark found only 50% of models claiming 32K+ context maintain satisfactory performance at that length. Near-perfect needle-in-haystack scores do not predict real-world long-context performance.

As a general rule, expect degradation to begin at 60-70% of the advertised context window for complex retrieval tasks. Always benchmark degradation thresholds with your specific workload rather than relying on published benchmarks. Model-specific thresholds go stale with each model update.

### Counterintuitive Findings

**Shuffled context can outperform coherent context.** Studies found incoherent (shuffled) haystacks produce better retrieval performance than logically ordered ones. Coherent context creates false associations that confuse retrieval; incoherent context forces exact matching.

**Single distractors have outsized impact.** The performance hit from one irrelevant document is disproportionately large. Treat distractor prevention as binary: either keep context clean or accept significant degradation.

**Low needle-question similarity accelerates degradation.** Tasks requiring inference across dissimilar content degrade faster with context length. Design retrieval to maximize semantic overlap between queries and retrieved content.

## Practical Guidance

### The Four-Bucket Mitigation Framework

Apply these four strategies based on which degradation pattern is active:

**Write** — Save context outside the window using scratchpads, file systems, or external storage. Use when context utilization exceeds 70% of the window.

**Select** — Pull only relevant context into the window through retrieval, filtering, and prioritization. Use when distraction or confusion symptoms appear.

**Compress** — Reduce tokens while preserving information through summarization, abstraction, and observation masking. Use when context is growing but all content is relevant.

**Isolate** — Split context across sub-agents or sessions to prevent any single context from growing past its degradation threshold. Use when confusion or clash symptoms appear.

### Architectural Patterns for Resilience

Implement just-in-time context loading. Use observation masking to replace verbose tool outputs with compact references after processing. Deploy sub-agent architectures where each agent holds only task-relevant context. Trigger compaction before context exceeds the model-specific degradation onset threshold.

## Examples

**Example 1: Detecting Degradation**
```yaml
# Context grows during long conversation
turn_1: 1000 tokens
turn_5: 8000 tokens
turn_10: 25000 tokens
turn_20: 60000 tokens (degradation begins)
turn_30: 90000 tokens (significant degradation)
```

**Example 2: Mitigating Lost-in-Middle**
```markdown
[CURRENT TASK]                      # At start - high attention
- Goal: Generate quarterly report
- Deadline: End of week

[DETAILED CONTEXT]                  # Middle - lower attention
- 50 pages of data
- Multiple analysis sections

[KEY FINDINGS]                     # At end - high attention
- Revenue up 15%
- Costs down 8%
```

## Guidelines

1. Monitor context length and performance correlation during development
2. Place critical information at beginning or end of context
3. Implement compaction triggers before degradation becomes severe
4. Validate retrieved documents for accuracy before adding to context
5. Use versioning to prevent outdated information from causing clash
6. Segment tasks to prevent context confusion across different objectives
7. Design for graceful degradation rather than assuming perfect conditions
8. Test with progressively larger contexts to find degradation thresholds

## Gotchas

1. **Normal variance looks like degradation**: Model output quality fluctuates naturally across runs. Establish a baseline over multiple runs and look for sustained, correlated decline tied to context growth. A 5-10% quality dip on one run is noise; the same dip consistently appearing after 40K tokens is signal.

2. **Model-specific thresholds go stale**: The degradation onset values in benchmark tables reflect specific model versions. Provider updates can move thresholds by 20-50% in either direction. Re-benchmark quarterly and after any major model update.

3. **Needle-in-haystack scores create false confidence**: A model scoring 99% on needle-in-haystack does not mean it handles 128K tokens well in production. Use task-specific benchmarks that mirror actual workload patterns.

4. **Contradictory retrieved documents poison silently**: When a RAG pipeline retrieves two documents that disagree on a fact, the model may silently pick one without signaling the conflict. Implement contradiction detection in the retrieval layer before documents enter context.

5. **Prompt quality problems masquerade as degradation**: Poor prompt structure produces symptoms identical to context degradation. Before diagnosing degradation, verify the same prompt works correctly at low context lengths.

6. **Degradation is non-linear with a cliff edge**: Performance does not degrade gradually — it holds steady until a model-specific threshold, then drops sharply. Set compaction triggers well before the cliff (at 70% of known onset), not at the onset itself.

7. **Over-organizing context can backfire**: Research shows shuffled haystacks sometimes outperform coherent ones for retrieval tasks. Test whether heavy structural formatting actually helps for the specific task.

## Integration

This skill builds on context-fundamentals and should be studied after understanding basic context concepts. It connects to:

- context-optimization - Techniques for mitigating degradation
- multi-agent-patterns - Using isolation to prevent degradation
- evaluation - Measuring and detecting degradation in production

## References

Related skills in this collection:
- context-fundamentals - Read when: lacking foundational understanding of context windows, token budgets, or placement mechanics
- context-optimization - Read when: degradation is diagnosed and specific mitigation techniques are needed
- evaluation - Read when: setting up production monitoring to detect degradation before it impacts users

External resources:
- Liu et al., 2023 "Lost in the Middle" - Primary research on U-shaped attention claims
- RULER benchmark documentation - For evaluating model long-context claims

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
