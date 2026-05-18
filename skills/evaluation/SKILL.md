---
name: evaluation
description: This skill should be used when the user asks to "evaluate agent performance", "set up evaluation frameworks", "measure agent quality", "create test sets for agents", or discusses outcome-focused evaluation, multi-dimensional scoring, agent quality drift, or production monitoring for non-deterministic agent behavior. Provides evaluation frameworks suited to agent systems.
---

# Evaluation Methods for Agent Systems

Assess agent performance through frameworks that accommodate non-deterministic behavior and dynamic decision-making. Agent evaluation differs fundamentally from deterministic software testing: agents may find alternative valid routes to goals, outputs vary across runs, and quality can drift as context, models, or tools change over time.

## When to Activate

Activate this skill when:
- Setting up evaluation pipelines for agent systems
- Choosing evaluation dimensions and metrics
- Building test sets from production usage
- Establishing quality baselines before changes
- Designing production monitoring for agent quality
- Debugging agent performance regressions

## Core Concepts

Evaluate outcomes, not paths. Agents may find alternative valid routes to goals, so assessment should verify results rather than verify a specific sequence of steps. A coding agent that writes correct, tested code via a different approach than expected has succeeded — path-based evaluation would incorrectly mark it as failed.

Multi-dimensional scoring is essential. Rather than collapsing quality into one metric, separate distinct quality dimensions:
- Factual accuracy
- Completeness
- Citation accuracy (for research agents)
- Source quality
- Tool efficiency
- Reasoning coherence

Agent quality drifts. Models update, tools change, contexts evolve. Evaluation is not a launch-time gate — it is continuous monitoring. Quality that passes evaluation at launch can degrade weeks later through model updates, tool API changes, or shifts in input distribution.

## Detailed Topics

### The 95% Finding

Research identifies three performance drivers for browsing agents:
- **Token usage explains 80% of variance** — agents with more tokens available perform better
- **Tool calls account for ~10%** — number of tool calls used
- **Model choice contributes ~5%** — but model quality improvements often outperform raw token increases

This suggests prioritizing model upgrades over simply increasing token budgets when performance is insufficient. Better models use available tokens more efficiently than weaker models given more tokens.

### Evaluation Dimensions

Define quality dimensions before writing evaluation code. Dimensions should be independently assessable and non-overlapping. Common dimensions for agent systems:

**For Research/Retrieval Agents**
- Factual accuracy: Are claims supported by sources?
- Completeness: Are all relevant aspects addressed?
- Citation accuracy: Do citations match the claims they support?
- Source quality: Are high-quality, authoritative sources used?

**For Coding Agents**
- Correctness: Does the code run and produce expected outputs?
- Test coverage: Are edge cases handled?
- Code quality: Is the code idiomatic and maintainable?
- Tool efficiency: Were tools used appropriately without redundant calls?

**For Task-Completion Agents**
- Goal achievement: Was the primary objective met?
- Side-effect avoidance: Were unintended changes avoided?
- Efficiency: Was the task completed in a reasonable number of steps?

### Test Set Construction

Build test sets from real usage patterns, not synthetic examples. Synthetic test cases miss the distribution of real inputs, which includes ambiguous requests, edge cases specific to your domain, and failure modes that only appear in production.

Minimum viable test set: 50 cases. Below this threshold, evaluation results have high variance and do not reliably predict production performance. For critical systems, target 200+ cases covering the full input distribution.

Structure test cases to include:
- The input (prompt, context, prior state)
- The expected outcome (what success looks like)
- The evaluation dimension being tested
- Complexity rating (simple/medium/complex)

### Evaluation Pipeline Architecture

Implement sequence:
1. Define quality dimensions and rubrics
2. Create descriptive rubrics with clear performance levels (1-3 or 1-5 scale)
3. Build test set from real usage (minimum 50 cases)
4. Automate evaluation pipeline
5. Establish baselines before making changes
6. Supplement automated assessment with human review on a sample

### Rubric Design

Write rubrics that are descriptive, not prescriptive. Good rubrics define what each score level looks like in practice:

```markdown
## Factual Accuracy (1-5)
5 - All claims verified against sources; no unsupported assertions
4 - Most claims verified; 1-2 minor unsupported statements
3 - Core claims verified; several secondary claims unverified
2 - Mixed accuracy; several incorrect or unverifiable claims
1 - Multiple factual errors; core claims unsupported
```

Vague rubrics produce inconsistent scores across evaluators (human or LLM).

### Continuous Monitoring

Agent quality drifts through:
- **Model updates**: Provider model updates without version pinning change behavior
- **Tool API changes**: Third-party tools change response formats or availability
- **Input distribution shift**: User behavior evolves, new query types emerge
- **Prompt-environment mismatch**: System prompts optimized for one context degrade in another

Monitor quality continuously, not just at launch. Set up automated evaluation on a sample of production traces weekly. Alert when any quality dimension drops more than 10% from baseline.

## Practical Guidance

### Evaluation Anti-Patterns

**Testing for specific execution paths**: Checking whether the agent used a specific tool in a specific order rather than whether the goal was achieved. Path testing produces false negatives for valid alternative approaches.

**Ignoring complexity variations**: Using only simple test cases produces optimistic estimates. Production performance on complex inputs often differs substantially from performance on simple inputs. Include a complexity breakdown in test set statistics.

**Relying on single aggregate scores**: A single quality score can hide regressions in specific dimensions. An agent with 90% accuracy might score 70% on citation accuracy — a critical failure for a research agent that the aggregate score masks.

**Using the same model family for evaluation and production**: A Claude model evaluating Claude outputs has self-enhancement bias. Use a different model family (GPT, Gemini) for evaluation or use human raters for high-stakes quality dimensions.

**Treating evaluation as one-time**: The most common failure mode. Teams invest in evaluation at launch, then stop. Quality drifts undetected until user complaints escalate.

### Human-Automated Calibration

Use human review to calibrate automated evaluation. Periodically have humans rate a sample of agent outputs on the same rubric. Calculate inter-rater agreement between human scores and automated scores. A calibrated automated evaluator achieves >80% agreement with human raters on most dimensions.

## Examples

**Example 1: Multi-Dimensional Rubric**
```python
evaluation_dimensions = {
    "factual_accuracy": {
        "weight": 0.30,
        "scale": "1-5",
        "description": "Are claims supported by provided sources?"
    },
    "completeness": {
        "weight": 0.25,
        "scale": "1-5",
        "description": "Are all relevant aspects of the query addressed?"
    },
    "tool_efficiency": {
        "weight": 0.20,
        "scale": "1-5",
        "description": "Were tools used without redundant or unnecessary calls?"
    },
    "reasoning_coherence": {
        "weight": 0.25,
        "scale": "1-5",
        "description": "Is the reasoning chain logical and well-structured?"
    }
}
```

**Example 2: Outcome-Focused Test Case**
```python
test_case = {
    "input": "Research the top 3 memory frameworks for AI agents",
    "expected_outcome": {
        "frameworks_mentioned": ["Mem0", "Zep", "Letta"],  # minimum set
        "comparison_dimensions": ["retrieval accuracy", "infrastructure cost"],
        "accuracy_threshold": 0.85
    },
    # NOT: "agent must call web_search exactly 3 times"
    "complexity": "medium"
}
```

## Guidelines

1. Evaluate outcomes, not execution paths
2. Define quality dimensions before writing evaluation code
3. Build test sets from real usage patterns (minimum 50 cases)
4. Establish baselines before making changes
5. Monitor quality continuously — not just at launch
6. Use a different model family for evaluation than for the agent
7. Calibrate automated evaluators against human raters periodically
8. Include complexity variation in test sets

## Gotchas

1. **Path testing produces false negatives**: An agent that achieves the goal via a different path than expected is incorrectly marked as failed. Evaluate outcomes, always.

2. **Synthetic test sets miss production distribution**: Carefully crafted test cases with clean inputs do not reflect real user behavior. Sample from production traces to build test sets.

3. **Single score hides dimension regressions**: An aggregate score of 85% can mask a critical dimension at 50%. Always report per-dimension scores alongside aggregates.

4. **Self-enhancement bias in evaluation**: The same model family rates its own outputs more favorably than human raters do. Use a different provider for LLM-as-Judge evaluation of high-stakes quality dimensions.

5. **Quality drift without monitoring**: A system that passed evaluation at launch can degrade silently through model updates, tool API changes, or input distribution shifts. Continuous monitoring is not optional for production systems.

## Integration

This skill connects to:
- advanced-evaluation - LLM-as-Judge techniques and bias mitigation
- context-degradation - Detecting degradation in production
- memory-systems - Evaluating memory retrieval quality

## References

Related skills in this collection:
- advanced-evaluation - When needing LLM-as-Judge with bias mitigation for complex evaluation
- context-degradation - For detecting context-related performance degradation in production

External resources:
- LoCoMo benchmark (Snap Research) - Long-conversation memory evaluation
- MemBench evaluation framework (ACL 2025) - Memory evaluation suites

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
