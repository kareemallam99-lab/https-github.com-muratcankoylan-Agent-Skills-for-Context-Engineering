---
name: advanced-evaluation
description: This skill should be used when the user asks to "implement LLM-as-Judge", "reduce evaluation bias", "design pairwise comparison evaluation", "calibrate evaluation rubrics", or discusses position bias, length bias, self-enhancement bias, direct scoring vs pairwise comparison, or production-grade LLM evaluation systems. Provides techniques for building reliable LLM-based evaluation pipelines.
---

# Advanced Evaluation: LLM-as-Judge

Implement LLM-as-Judge evaluation systems with systematic bias mitigation. LLM-based evaluation scales to volumes where human review is impractical, but naive implementations embed systematic biases that invalidate scores. Treat LLM-as-Judge as a designed system addressing known failure modes, not a single technique.

## When to Activate

Activate this skill when:
- Implementing automated evaluation at scale using LLMs
- Designing direct scoring or pairwise comparison systems
- Mitigating known LLM evaluation biases
- Calibrating LLM evaluators against human raters
- Building evaluation pipelines for subjective quality dimensions
- Choosing between scoring approaches for specific quality dimensions

## Core Concepts

LLM-as-Judge evaluation has six systematic biases that, if unaddressed, invalidate scores:
1. **Position bias** — First responses get unfair preference in pairwise comparisons
2. **Length bias** — Longer outputs score higher regardless of quality
3. **Self-enhancement bias** — Models rate their own outputs more favorably
4. **Verbosity bias** — Unnecessary detail inflates scores
5. **Authority bias** — Confident tone masks accuracy issues
6. **Rubric drift** — Evaluation standards shift as models improve over time

These biases are not random noise — they are systematic, directional errors that compound across large evaluation runs. An evaluation pipeline with unmitigated length bias will consistently overrate verbose agents and underrate concise ones.

## Detailed Topics

### Two Primary Evaluation Methods

**Direct Scoring**
A single LLM rates responses on a defined scale (1-3, 1-5, or 1-10) against a rubric. Best for objective criteria like factual accuracy, where "correct" has a verifiable definition.

When to use:
- Objective quality dimensions (factual accuracy, code correctness)
- Cases where a clear rubric exists
- High-volume evaluation where pairwise comparison cost is prohibitive

Limitation: Score calibration requires attention. The same response can receive different absolute scores from different models or across different evaluation sessions if rubrics drift.

**Pairwise Comparison**
Two responses are compared and the evaluator picks the better one (or declares a tie). Achieves higher human-judge agreement than direct scoring for preference tasks.

When to use:
- Subjective quality dimensions (tone, style, persuasiveness, helpfulness)
- Preference tasks where relative quality matters more than absolute quality
- Cases where defining a rubric for direct scoring is difficult

Limitation: O(n²) cost for ranking n responses. Swap positions twice to detect position bias.

### Bias Mitigation Techniques

**Position Bias Mitigation**
Always swap the order of responses in pairwise comparisons and check for consistency. If the evaluator prefers A over B in one order but B over A in the reversed order, mark as inconsistent rather than averaging.

```python
def pairwise_with_debiasing(response_a, response_b, rubric):
    result_1 = evaluate(response_a, response_b, rubric)  # A first
    result_2 = evaluate(response_b, response_a, rubric)  # B first
    
    if result_1 == result_2:
        return result_1  # Consistent — use this result
    else:
        return "tie"     # Inconsistent — mark as tie
```

**Length Bias Mitigation**
Explicitly instruct the evaluator to ignore length when it is not a quality dimension. Include in the evaluation prompt: "Quality, not length, determines the score. A concise, accurate response outscores a verbose, inaccurate one."

Alternatively, normalize by truncating long responses to a maximum before evaluation, but only for dimensions where length is not relevant.

**Self-Enhancement Bias Mitigation**
Never use the same model family to evaluate outputs it produced. Use a different provider for evaluation:
- Claude outputs → evaluated by GPT or Gemini
- GPT outputs → evaluated by Claude or Gemini

For high-stakes evaluation, use human raters for a calibration sample rather than relying exclusively on LLM evaluation.

**Rubric Drift Mitigation**
Anchor rubrics with concrete examples at each score level. Abstract descriptors drift as models improve — a response that scored 4/5 a year ago may score 3/5 today because the model's reference point has shifted upward. Concrete examples do not drift.

```markdown
## Factual Accuracy — Score 4 Example
"The lost-in-middle effect reduces recall accuracy in middle-context positions 
by 10-40% compared to positions at the beginning or end of context."
↑ This specific, verifiable claim with quantified range = Score 4
```

### Justification Requirement

Always require justification before scores. Evaluation pipelines that request scores without justification show 15-25% lower reliability than those requiring justification first. Justification forces the evaluator to articulate reasoning before committing to a score, reducing superficial pattern-matching.

```markdown
# Evaluation prompt template
Evaluate the following response on [dimension].

Response: {response}

First, explain what the response does well and what it lacks on [dimension].
Then, assign a score from 1-5.

Justification:
Score:
```

### Scale Granularity Selection

Match scale granularity to rubric specificity:
- **1-3 scale**: Use when rubric distinguishes only coarse quality levels (poor/acceptable/excellent). Fewer options reduce disagreement.
- **1-5 scale**: Use for most quality dimensions. Sufficient granularity for meaningful differentiation without over-precision.
- **1-10 scale**: Use only when the rubric can meaningfully distinguish 10 levels and human annotators agree on fine-grained distinctions.

Finer scales do not produce more reliable scores — they produce less reliable scores when rubrics cannot justify the granularity.

### Confidence Calibration

Include confidence calibration aligned with consistency checks. High-confidence low-consistency evaluations indicate the evaluator is confidently wrong — a calibration failure that regular metrics miss.

```python
def calibrated_evaluation(response, rubric, n_runs=3):
    scores = [evaluate(response, rubric) for _ in range(n_runs)]
    score_variance = variance(scores)
    
    if score_variance > 0.5:  # High variance = low consistency
        return {
            "score": mean(scores),
            "confidence": "low",
            "note": "High variance across evaluation runs"
        }
    return {
        "score": mean(scores),
        "confidence": "high"
    }
```

## Practical Guidance

### Building a Production Evaluation Pipeline

1. **Define dimensions with concrete rubrics** — avoid abstract descriptors
2. **Choose method per dimension** — direct scoring for objective, pairwise for subjective
3. **Require justification before scores** — non-negotiable for reliability
4. **Implement position swap** — for all pairwise comparisons
5. **Use different model family for evaluation** — avoid self-enhancement bias
6. **Calibrate against human raters** — run human calibration on 5% of evaluation volume monthly
7. **Monitor evaluator consistency** — track variance across repeated evaluations of the same content

### Domain-Specific Rubrics

Do not use generic rubrics. Domain-specific rubrics that reference actual quality criteria for the task produce 20-30% better agreement with human raters than generic quality rubrics.

Example: For a coding agent, a factual accuracy rubric should reference code correctness criteria specific to the language and domain, not a generic "accuracy" descriptor.

## Examples

**Example 1: Direct Scoring Prompt**
```markdown
You are evaluating an AI agent's response for factual accuracy.

Response to evaluate:
{response}

Rubric:
5 - All claims verified against sources; no unsupported assertions
4 - Most claims verified; 1-2 minor unsupported statements  
3 - Core claims verified; several secondary claims unverified
2 - Mixed accuracy; several incorrect or unverifiable claims
1 - Multiple factual errors; core claims unsupported

First, identify each factual claim and whether it is supported.
Then, assign a score from 1-5.

Factual claims analysis:
Score:
```

**Example 2: Pairwise Comparison Prompt**
```markdown
Compare these two responses for helpfulness. Ignore length — a shorter, 
more helpful response is better than a longer, less helpful one.

Response A:
{response_a}

Response B:
{response_b}

Which response is more helpful? Explain why, then choose A, B, or Tie.

Reasoning:
Verdict:
```

## Guidelines

1. Always require justification before scores to improve reliability 15-25%
2. Swap positions twice in pairwise evaluations to detect position bias
3. Match scale granularity (1-3, 1-5, 1-10) to rubric specificity
4. Use domain-specific rubrics rather than generic templates
5. Never use the same model family to evaluate its own outputs
6. Include confidence calibration aligned with consistency checks
7. Calibrate automated evaluators against human raters monthly
8. Anchor rubrics with concrete examples to prevent drift

## Gotchas

1. **Requesting scores without justification**: Evaluation reliability drops 15-25% without justification. The LLM produces superficial pattern-matched scores rather than reasoned assessments. Always require justification first.

2. **Single pairwise run**: Without position swapping, position bias systematically favors the first response. Always run both orderings and mark inconsistent pairs as ties.

3. **Using the same model for evaluation and production**: Self-enhancement bias means the model rates its own outputs more favorably than external raters do. The bias is directional and consistent, not random noise — it will skew all evaluation results upward.

4. **Generic rubrics across domains**: A generic "accuracy" rubric applied to both research and coding tasks produces inconsistent scores because "accuracy" means different things in each domain. Write domain-specific rubrics with domain-specific examples.

5. **Ignoring rubric drift**: Evaluation scores from six months ago are not comparable to scores today if the rubric is abstract. Concrete examples prevent drift; abstract descriptors do not.

6. **High confidence + high variance = calibration failure**: An evaluator that gives confident scores but produces different scores on repeated runs of the same content is confidently unreliable. Monitor consistency and reduce confidence weight for high-variance evaluators.

## Integration

This skill builds on evaluation. It connects to:
- evaluation - Foundation evaluation principles and test set construction
- context-fundamentals - Understanding token costs of evaluation pipelines
- multi-agent-patterns - Using dedicated evaluation agents

## References

Related skills in this collection:
- evaluation - Foundation for evaluation principles; read first
- multi-agent-patterns - For evaluation architectures using dedicated evaluator agents

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
