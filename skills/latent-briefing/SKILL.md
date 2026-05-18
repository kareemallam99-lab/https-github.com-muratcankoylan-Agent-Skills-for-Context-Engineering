---
name: latent-briefing
description: This skill should be used when the user asks to "share KV cache between agents", "implement latent briefing", "transfer orchestrator state to workers without replay", "debug token explosion in recursive agents", or discusses KV-cache sharing, Attention Matching compaction, orchestrator-worker state transfer, or alternatives to text summarization for agent handoff. Provides implementation guidance for representation-level state sharing between agents.
---

# Latent Briefing and KV Cache Memory Sharing

Transfer orchestrator context to worker agents at the representation level rather than the text level. Latent Briefing addresses the critical inefficiency in hierarchical multi-agent systems: orchestrators accumulate lengthy reasoning trajectories but typically hand off only narrow text summaries to workers. Full trajectory replay is expensive; text summarization loses information. Latent Briefing shares memory at the KV tensor level.

## When to Activate

Activate this skill when:
- Designing orchestrator-worker systems where text handoff is expensive or noisy
- Debugging token explosion in recursive agent systems
- Evaluating alternatives to summarization and RAG for state transfer
- Implementing systems with tight control over inference runtime
- Optimizing KV-cache utilization across agent boundaries

## Core Concepts

In recursive agent setups, orchestrators accumulate lengthy reasoning trajectories but typically hand off only narrow text summaries to workers. Passing complete trajectories inflates token costs; summarization introduces latency and information loss. Latent Briefing addresses this by sharing memory at the **representation level** rather than the text level.

Rather than serializing reasoning into language, the system compacts the orchestrator's KV cache to retain only positions most relevant to the current worker task. The worker receives a compressed KV state rather than a text reconstruction of the orchestrator's reasoning.

**Critical prerequisite**: Latent Briefing is only practical when the system controls the worker inference runtime closely enough to inspect or transform KV state. It requires direct access to internal KV tensors — it is unsuitable for API-only deployments where the KV cache is opaque.

## Detailed Topics

### The Handoff Problem

Standard orchestrator-worker handoff uses one of two approaches, each with significant costs:

**Text Summarization**
- Orchestrator generates a text summary of its reasoning and findings
- Worker receives the summary as part of its prompt
- Problems: Information loss through compression, additional latency for summary generation, summary quality degrades as reasoning complexity increases

**Full Trajectory Replay**
- Orchestrator passes its complete conversation history to the worker
- Worker processes the full history from scratch
- Problems: Massive token cost (proportional to orchestrator trajectory length), KV computation not reused across similar trajectories

**Latent Briefing Alternative**
- Orchestrator's KV cache is compacted to retain task-relevant positions
- Compacted KV state is transferred directly to the worker inference process
- Worker's attention operates on the pre-computed representations, not raw text
- Result: Lower token cost than replay, higher fidelity than summarization

### Attention Matching Compaction

Latent Briefing uses Attention Matching (AM) compaction with three modifications specific to orchestrator-worker transfer:

**1. Task-Guided Queries**
Standard AM compaction uses self-attention patterns to identify important positions. Latent Briefing uses queries derived from the current worker prompt — positions important to the worker's specific task are retained, not positions important to the orchestrator's general reasoning.

This is the key distinction: compaction is goal-directed toward the receiving agent's task, not toward the sending agent's history.

**2. Shared Global Mask**
Standard AM applies per-head compaction, allowing different attention heads to retain different positions. Latent Briefing uses a shared global mask across all attention heads. This ensures the retained positions are coherent — a position either survives compaction for all heads or is dropped from all heads.

Per-head compaction can create incoherent KV states where some heads have position X and others do not, causing attention pattern mismatches in the worker.

**3. Robust Thresholding**
Standard compaction uses fixed top-k selection (retain the top K positions by attention score). Latent Briefing uses median + tau × MAD (Median Absolute Deviation) thresholding, which adapts to the distribution of attention scores rather than using a fixed count.

Fixed top-k can retain too many positions when attention is diffuse (expensive) or too few when attention is concentrated (lossy). Adaptive thresholding maintains quality across varying attention distributions.

```python
def compute_retention_mask(attention_scores, tau=1.5):
    median = torch.median(attention_scores)
    mad = torch.median(torch.abs(attention_scores - median))
    threshold = median + tau * mad
    return attention_scores > threshold
```

### When to Use Latent Briefing vs. Alternatives

**Use Latent Briefing when:**
- Orchestrator trajectory length dominates total token costs
- Text summarization quality is insufficient for the task
- You control the inference runtime and can access KV tensors
- Orchestrator and worker use the same model architecture

**Use plain text summarization when:**
- API-only deployment (no KV access)
- Auditability requires human-readable handoff documents
- Orchestrator and worker use different model architectures
- Development speed is prioritized over optimal efficiency

**Use full trajectory replay when:**
- Worker needs complete orchestrator reasoning for correctness
- Token cost is not a constraint
- Compaction quality is uncertain for the domain

**Use RAG/retrieval when:**
- Orchestrator state is too large even for compaction
- Worker needs selective access to orchestrator findings
- Findings can be indexed semantically

### Architecture Requirements

Latent Briefing requires:
1. **Same model architecture**: Orchestrator and worker must share the same model to have compatible KV tensor formats
2. **Runtime access**: The orchestrator must be able to extract its KV cache programmatically
3. **Worker injection**: The worker inference process must accept pre-computed KV states as initialization
4. **Compaction computation**: The compaction step requires forward pass access to compute attention scores against the worker query

These requirements make Latent Briefing impractical for cloud API deployments and most managed agent frameworks. It is best suited for self-hosted inference infrastructure where the inference server is controllable.

## Practical Guidance

### Evaluation Framework

Co-design compaction with rigorous evaluation to prevent quality cliffs:

1. **Baseline**: Measure task performance with full trajectory replay (ground truth)
2. **Summarization baseline**: Measure task performance with text summarization
3. **Latent Briefing**: Measure task performance with varying compaction ratios
4. **Quality curve**: Plot performance vs. compaction ratio to find the knee point

The knee point — the compaction ratio where performance begins to drop significantly — defines the minimum viable KV state size for the task. Set tau in the thresholding formula to target compaction ratios above the knee.

### Failure Mode: Quality Cliffs

Aggressive compaction can produce quality cliffs where performance is stable until a threshold compaction ratio, then collapses. Unlike gradual degradation, quality cliffs make the safe operating range difficult to identify.

Avoid quality cliffs by:
- Testing at multiple compaction ratios, not just the target
- Setting safety margins above the knee point
- Monitoring worker output quality as a proxy for compaction quality
- Implementing fallback to text summarization when quality drops below threshold

## Examples

**Example 1: Compaction Workflow**
```python
def latent_briefing_handoff(orchestrator_kv, worker_prompt, tau=1.5):
    # Compute worker query embeddings
    worker_query = embed(worker_prompt)
    
    # Score orchestrator positions by relevance to worker query
    attention_scores = compute_cross_attention(
        query=worker_query,
        keys=orchestrator_kv.keys,
    )
    
    # Compute adaptive retention mask
    retention_mask = compute_retention_mask(attention_scores, tau)
    
    # Apply global mask across all heads
    compacted_kv = apply_global_mask(orchestrator_kv, retention_mask)
    
    return compacted_kv

def worker_inference(worker_prompt, compacted_kv):
    # Initialize worker with pre-computed KV state
    return model.generate(
        prompt=worker_prompt,
        past_key_values=compacted_kv,  # Skip recomputation
    )
```

**Example 2: Quality Monitoring**
```python
def monitored_handoff(orchestrator_kv, worker_prompt, fallback_summary):
    compacted_kv = latent_briefing_handoff(orchestrator_kv, worker_prompt)
    
    # Check compaction quality via probe task
    quality_score = evaluate_compacted_state(compacted_kv, worker_prompt)
    
    if quality_score < QUALITY_THRESHOLD:
        # Fall back to text summarization
        return worker_inference_with_summary(worker_prompt, fallback_summary)
    
    return worker_inference(worker_prompt, compacted_kv)
```

## Guidelines

1. Prefer Latent Briefing when replaying orchestrator state dominates token costs
2. Choose plain text when auditability or closed-model APIs take priority
3. Use task-guided queries for compaction — match retained positions to worker task
4. Apply global mask across all attention heads for coherent KV states
5. Use adaptive thresholding (median + tau × MAD) over fixed top-k
6. Co-design compaction ratio with evaluation — find the knee point
7. Implement quality monitoring with fallback to text summarization
8. Only apply to same-architecture model pairs

## Gotchas

1. **API-only deployments cannot access KV state**: If the orchestrator runs behind an API (OpenAI, Anthropic, etc.), KV tensors are inaccessible. Latent Briefing requires self-hosted inference infrastructure. Do not attempt workarounds — use text summarization instead.

2. **Architecture mismatch invalidates compaction**: KV tensors from one model architecture (e.g., LLaMA 3) are incompatible with another (e.g., Mistral). Orchestrator and worker must use identical model architectures, including layer count and head dimensions.

3. **Quality cliffs without monitoring**: Compaction that appears to work at low ratios can fail catastrophically at higher ratios. Always test the full compaction ratio curve and implement runtime quality monitoring with fallback.

4. **Fixed top-k breaks on diffuse attention**: When attention is spread across many positions (common in long reasoning trajectories), fixed top-k retains too many positions. Adaptive thresholding handles varying attention distributions correctly.

5. **Compaction step cost**: Computing cross-attention between the worker query and orchestrator KV requires a forward pass, which has its own token cost. For short orchestrator trajectories, this compaction cost may exceed the savings from reduced replay cost. Measure compaction ROI before deploying.

## Integration

This skill builds on context-optimization and multi-agent-patterns. It connects to:

- multi-agent-patterns - Orchestrator-worker architecture patterns
- context-optimization - KV-cache optimization as the foundation
- evaluation - Evaluating compaction quality and fallback triggers

## References

Related skills in this collection:
- context-optimization - KV-cache optimization fundamentals required before implementing Latent Briefing
- multi-agent-patterns - Orchestrator-worker architectures where Latent Briefing applies

External resources:
- Attention Matching compaction research
- KV cache compression literature
- Self-hosted LLM inference frameworks (vLLM, TGI) for KV access

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
