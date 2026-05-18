---
name: bdi-mental-states
description: This skill should be used when the user asks to "implement BDI architecture", "model cognitive agent states", "design belief-desire-intention systems", "integrate RDF with agent reasoning", or discusses neuro-symbolic AI, BDI frameworks, mental state modeling, SPARQL validation for agents, or Logic Augmented Generation. Provides implementation guidance for Belief-Desire-Intention cognitive agent architectures.
---

# BDI Mental State Modeling

Implement Belief-Desire-Intention (BDI) architecture for cognitive agent systems. BDI transforms RDF context into structured cognitive representations that support formal reasoning, auditability, and neuro-symbolic integration. Apply when agents need explainable decision-making, ontological grounding, or integration with semantic web technologies.

## When to Activate

Activate this skill when:
- Implementing BDI frameworks for cognitive agent modeling
- Integrating RDF context with agent reasoning systems
- Building neuro-symbolic AI that combines formal reasoning with LLMs
- Designing agents with explainable belief-desire-intention structures
- Implementing SPARQL validation for agent mental states
- Mapping BDI to execution frameworks (SEMAS, JADE)

## Core Concepts

BDI reasoning requires distinguishing what persists from what happens. Mental entities (beliefs, desires, intentions) are persistent — they exist across reasoning cycles. Mental processes (perception, deliberation, execution) are temporal — they operate on persistent entities and update them.

Separate mental state architecture from the temporal processes that generate and modify it. This separation enables:
- Auditability: inspect any mental entity's current state and provenance
- Temporal validity: track when beliefs were formed and when they expire
- Bidirectional reasoning: query both "what should the agent do?" and "why did it act?"

Ground all beliefs in explicit world state references. Abstract beliefs ("the user is frustrated") are unverifiable and degrade reasoning quality. Grounded beliefs ("the user has submitted the same request 3 times without accepting any response") are verifiable and auditable.

## Detailed Topics

### Core BDI Architecture

**Beliefs** — The agent's model of the current world state. Beliefs should be:
- Grounded in explicit observations, not inferences
- Tagged with temporal validity intervals (valid_from, valid_until)
- Attached to justifications (provenance of each belief)
- Updated through explicit perception processes, not implicit drift

**Desires** — The agent's goals and objective states. Desires should be:
- Clearly scoped with success conditions
- Prioritized when multiple desires conflict
- Decomposable into sub-desires when complex
- Distinct from plans (what to achieve, not how to achieve it)

**Intentions** — The agent's committed plans for achieving desires. Intentions should be:
- Bound to specific desires they serve
- Time-bounded with expiry conditions
- Tracked for execution status (pending, executing, succeeded, failed)
- Revocable when beliefs change enough to invalidate the plan

### The T2B2T Pipeline

The Triples-to-Beliefs-to-Triples paradigm creates a bidirectional flow between RDF and BDI representations:

**Phase 1: Triples → Beliefs (Perception)**
Convert incoming RDF triples into belief instances:
1. Parse RDF context into triple sets
2. Match triples to belief templates (entity, relationship, attribute)
3. Instantiate belief objects with validity intervals
4. Attach justifications linking beliefs to source triples

**Phase 2: Beliefs → Triples (Projection)**
After deliberation and plan execution, project outcomes back to RDF:
1. Extract belief updates from plan execution results
2. Generate update triples reflecting new world state
3. Invalidate superseded triples (do not delete — mark as expired)
4. Expose updated triples to downstream systems

This preserves provenance while enabling downstream systems to consume agent outputs as linked data.

### Belief Grounding Principles

Apply these grounding rules to every belief:

1. **Ground in world state references**: Link beliefs to explicit observations or sensor readings, not to inferences or assumptions
2. **Attach justifications**: Every belief must have a `hasJustification` property pointing to its evidence
3. **Assign temporal validity**: Set `validFrom` and `validUntil` intervals; beliefs without expiry accumulate indefinitely
4. **Decompose complex beliefs**: Use `hasPart` relations to break composite beliefs into simpler, individually verifiable components

```
Belief: agent_reached_inbox
  validFrom: 2026-01-15T10:30:00Z
  validUntil: 2026-01-15T10:45:00Z
  hasJustification: observation_007
  groundedIn: inbox_check_result_42 (tool output)
  hasPart: [inbox_accessible, inbox_has_messages, inbox_auth_valid]
```

### SPARQL Validation Patterns

Validate mental state consistency using SPARQL queries before deliberation:

```sparql
# Detect stale beliefs that should have expired
SELECT ?belief ?validUntil WHERE {
  ?belief a :Belief ;
          :validUntil ?validUntil ;
          :active true .
  FILTER(?validUntil < NOW())
}

# Detect conflicting intentions
SELECT ?intention1 ?intention2 WHERE {
  ?intention1 a :Intention ;
              :targets ?resource .
  ?intention2 a :Intention ;
              :targets ?resource .
  FILTER(?intention1 != ?intention2)
}
```

Run validation before each deliberation cycle. Stale beliefs and conflicting intentions that enter deliberation corrupt the reasoning process.

### Integration with LLMs (Logic Augmented Generation)

Combine BDI formal reasoning with LLM generation through Logic Augmented Generation (LAG):

1. **Pre-generation validation**: Run SPARQL queries to validate context consistency before LLM call
2. **Ontological constraints**: Pass relevant ontological constraints in the LLM prompt to guide output
3. **Post-generation grounding**: Parse LLM outputs and ground them in the belief graph
4. **Conflict detection**: Check generated beliefs against existing beliefs for contradictions

LAG constrains LLM outputs against ontological constraints without requiring the LLM to be formally correct — the formal system validates and filters outputs, the LLM provides natural language expressiveness.

### Mapping to Execution Frameworks

**SEMAS (Semantic Multi-Agent Systems)**
Maps BDI mental states to semantic service descriptions. Beliefs map to service state; desires map to service goals; intentions map to service invocations.

**JADE (Java Agent DEvelopment Framework)**
Maps BDI to JADE agent behaviors. Beliefs map to agent knowledge base; desires map to agent goals; intentions map to agent behavior sequences. JADE's FSMBehaviour provides the execution substrate for intention lifecycle management.

## Practical Guidance

### When to Use BDI vs. Simpler Architectures

**Use BDI when:**
- Agent decisions must be explainable and auditable
- The domain has formal ontologies or semantic web infrastructure
- Multiple agents must share and reason over common belief spaces
- Temporal validity of beliefs is critical to correct behavior

**Use simpler architectures when:**
- Auditability is not required
- The domain lacks formal ontologies
- Development speed is prioritized over formal correctness
- The agent's reasoning is relatively straightforward

BDI adds significant implementation complexity. Apply it when the benefits of formal reasoning and auditability justify that complexity.

### Belief Lifecycle Management

Implement explicit lifecycle transitions for beliefs:
```
hypothetical → provisional → confirmed → superseded → archived
```

Never delete beliefs — archive them with the timestamp when they were superseded. Historical belief states are essential for explaining past agent decisions and for debugging temporal reasoning.

## Examples

**Example 1: Belief Object Structure**
```python
@dataclass
class Belief:
    id: str
    proposition: str           # What is believed
    valid_from: datetime       # When belief became valid
    valid_until: datetime      # When belief expires (None = indefinite)
    confidence: float          # 0.0 to 1.0
    justification_ids: list    # IDs of supporting observations
    status: str                # active, superseded, archived
    parts: list                # Sub-beliefs (decomposition)
```

**Example 2: BDI Deliberation Cycle**
```python
def deliberation_cycle(belief_graph, desire_set):
    # Validate current mental state
    validate_beliefs(belief_graph)  # Remove stale beliefs
    
    # Filter achievable desires
    active_desires = [d for d in desire_set 
                      if preconditions_met(d, belief_graph)]
    
    # Select intention
    selected_desire = prioritize(active_desires)
    intention = generate_plan(selected_desire, belief_graph)
    
    # Execute and update beliefs
    result = execute(intention)
    update_beliefs(belief_graph, result)
    
    return result
```

## Guidelines

1. Ground all beliefs in explicit world state references, not inferences
2. Attach justifications to every mental entity for auditability
3. Assign temporal validity intervals to prevent stale belief accumulation
4. Run SPARQL validation before each deliberation cycle
5. Never delete beliefs — archive with supersession timestamps
6. Use T2B2T pipeline for bidirectional RDF-BDI integration
7. Apply LAG to constrain LLM outputs against ontological rules
8. Separate mental state architecture from temporal processes

## Gotchas

1. **Ungrounded beliefs accumulate silently**: Beliefs not linked to explicit world state references drift from actual state over time. The agent reasons over a stale, increasingly inaccurate model. Require explicit grounding for every belief at creation time.

2. **Missing temporal validity creates belief bloat**: Beliefs without expiry dates never leave the belief graph. After extended operation, the graph fills with irrelevant historical beliefs that slow reasoning. Set expiry intervals for all beliefs, even ones that seem permanent.

3. **BDI complexity without commensurate benefit**: BDI is a significant implementation investment. Teams sometimes adopt it for its conceptual elegance without validating that the domain requires formal reasoning. Validate the need before committing to the architecture.

4. **Conflicting intentions without detection**: When two intentions target the same resource or contradict each other, the agent's behavior becomes unpredictable. Implement conflict detection in the deliberation cycle, not as an afterthought.

5. **LLM outputs not grounded post-generation**: Using LLM outputs directly as beliefs without grounding them in the ontology bypasses the consistency guarantees BDI provides. Always ground LLM-generated beliefs before adding them to the belief graph.

## Integration

This skill connects to:
- context-fundamentals - Context engineering for BDI agent systems
- memory-systems - Persistent belief storage across sessions
- evaluation - Evaluating BDI agent decision quality

## References

Related skills in this collection:
- memory-systems - For persisting belief graphs across agent sessions
- context-fundamentals - For managing BDI system context efficiently

External resources:
- BDI agent model literature (Bratman, Rao & Georgeff)
- JADE framework documentation
- SPARQL 1.1 specification
- Logic Augmented Generation research

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
