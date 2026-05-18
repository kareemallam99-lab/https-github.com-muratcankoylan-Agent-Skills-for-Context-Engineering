---
name: project-development
description: This skill should be used when the user asks to "start an LLM-powered project", "evaluate if a task suits AI agents", "design agent pipelines", "estimate LLM project costs", or discusses task-model fit, pipeline architecture, batch processing, structured outputs, or choosing between single-agent and multi-agent approaches for new projects. Provides methodology for scoping and building LLM-powered applications.
---

# Project Development Methodology

Identify tasks suited to LLM processing, design effective project architectures, and use agent-assisted development for rapid iteration. Most LLM projects fail not because the technology is wrong but because the task selection, architecture, or validation process was flawed. Start with manual prototyping to validate fit before building automation.

## When to Activate

Activate this skill when:
- Starting projects that might benefit from LLM processing
- Evaluating task suitability for agents versus traditional code
- Designing LLM-powered application architecture
- Planning batch processing pipelines with structured outputs
- Choosing between single-agent and multi-agent approaches
- Estimating costs and timelines for LLM-heavy projects

## Core Concepts

Validate task-model fit before automating. Testing one representative example with your target model before building a pipeline reveals whether the model has sufficient knowledge for the task, can produce the required output format, meets quality expectations, and what failure modes to expect. This step costs less than an hour and saves weeks of wasted implementation.

Start minimal, add complexity only when production evidence requires it. Vercel reduced from 17 tools to 2 (bash and SQL), improving success from 80% to 100%. Complex architectures are harder to debug, slower to iterate, and more expensive to operate. Build the simplest architecture that achieves the goal.

## Detailed Topics

### Task-Model Fit Assessment

**Proceed when tasks involve:**
- Combining information from multiple sources
- Subjective judgment using rubrics
- Natural language outputs
- Error tolerance (a 5% error rate is acceptable)
- Batch processing over many items
- Domain knowledge within model training data

**Stop when tasks require:**
- Precise computation or exact algorithms
- Real-time sub-second responses
- Perfect accuracy (hallucination risks exist)
- Proprietary data the model cannot learn from prompts
- Sequential dependencies that compound errors
- Identical outputs for identical inputs

### Manual Prototype Step

Test one representative example with your target model before automation. This validates:
1. **Knowledge sufficiency**: Does the model know enough to attempt the task?
2. **Output format capability**: Can it produce the required format reliably?
3. **Quality expectations**: Does the quality meet the bar?
4. **Failure modes**: What breaks, and how often?

Manual prototyping with 5-10 examples reveals 80% of architectural issues before any code is written. Skipping this step consistently leads to discovering fundamental incompatibilities late in development.

### Pipeline Architecture

Structure LLM pipelines as five discrete stages:

**1. Acquire** — Fetch raw data from sources (APIs, databases, filesystems)
**2. Prepare** — Transform into prompt-ready format (chunking, formatting, filtering)
**3. Process** — Execute LLM calls (non-deterministic, expensive, parallelize here)
**4. Parse** — Extract structured data from LLM outputs (handle variation gracefully)
**5. Render** — Generate final outputs (reports, files, API responses)

Implement each stage independently so failures in one stage do not corrupt others. The Process stage is the only non-deterministic stage — keep side-effecting operations (database writes, file updates) in Acquire and Render, not in Process.

### File System State Management

Use directory structures to track completion at each pipeline stage rather than databases. This enables natural idempotency and human-readable debugging.

```
pipeline/
├── 01_acquired/     # Raw data fetched
├── 02_prepared/     # Prompt-ready format
├── 03_processed/    # LLM outputs
├── 04_parsed/       # Structured extractions
└── 05_rendered/     # Final outputs
```

Each stage reads from its input directory and writes to its output directory. Re-running a stage overwrites outputs safely. Human inspection of any stage's directory shows exactly what the pipeline produced.

### Structured Output Design

Design LLM prompts to produce structured outputs that parse reliably:
- Use section markers (`## SUMMARY\n`, `## RECOMMENDATIONS\n`)
- Provide format examples in the prompt
- Request rationale disclosure (makes errors visible)
- Constrain values to explicit options ("Choose: HIGH, MEDIUM, or LOW")

Build flexible parsers that handle LLM variation gracefully. LLMs do not always follow format instructions exactly. A parser that requires exact compliance produces brittle pipelines — expect minor deviations and handle them.

### Cost Estimation

```
Total cost = (items × tokens_per_item × price_per_token) + overhead
```

Add 20-30% buffer for retries and failures. Track actual costs during development — estimated costs diverge from actual costs when token usage per item varies more than expected.

Common cost overruns:
- Retrieval augmentation adds more tokens per item than estimated
- Retry logic doubles costs when failure rate is higher than expected
- Model upgrades mid-project change the cost model

### Agent vs. Single-Agent Decision

Default to single-agent pipelines for independent batch items — simpler, cheaper, easier to debug. Escalate to multi-agent only when:
- Parallel exploration provides clear quality improvement (validated by benchmark)
- Context window limits are genuinely hit (not just approached)
- Different subtasks require fundamentally different tool sets

## Practical Guidance

### Project Planning Template

1. **Define scope**: What are the inputs, outputs, error tolerance, and value delivered?
2. **Classify task type**: Does this match the "Proceed" criteria? Any "Stop" criteria apply?
3. **Manual prototype**: Test 5-10 representative examples with the target model
4. **Select architecture**: Single-stage vs. pipeline vs. multi-agent
5. **Design storage**: File-system state management for each pipeline stage
6. **Calculate costs**: Per-item cost × volume + buffer, compare against value
7. **Implement stage-by-stage**: Build and test each stage before proceeding

### Iteration Discipline

Plan for multiple iterations — production systems always require refactoring as models improve. Design for changeability:
- Abstract the model behind a thin wrapper (swapping models should take minutes)
- Separate prompt logic from pipeline logic (prompts change more than pipeline structure)
- Make each stage independently testable

## Examples

**Example 1: Pipeline Directory Structure**
```
report_pipeline/
├── config.yaml              # Input sources and parameters
├── 01_acquired/
│   ├── company_001.json
│   └── company_002.json
├── 02_prepared/
│   ├── company_001_prompt.md
│   └── company_002_prompt.md
├── 03_processed/
│   ├── company_001_response.txt
│   └── company_002_response.txt
├── 04_parsed/
│   ├── company_001_structured.json
│   └── company_002_structured.json
└── 05_rendered/
    └── final_report.pdf
```

**Example 2: Structured Output Prompt**
```markdown
Analyze this company and provide a structured assessment.

Company data:
{company_data}

Provide your assessment in exactly this format:

## SUMMARY
[2-3 sentence summary]

## RISK_LEVEL
[Choose exactly one: HIGH, MEDIUM, LOW]

## RISK_FACTORS
- [Factor 1]
- [Factor 2]

## RECOMMENDATION
[Specific recommended action]
```

## Guidelines

1. Validate task-model fit with manual prototyping before building automation
2. Structure pipelines as five discrete stages: Acquire, Prepare, Process, Parse, Render
3. Use filesystem directories for stage-level state management
4. Default to single-agent; escalate to multi-agent only with benchmark evidence
5. Add 20-30% buffer to cost estimates
6. Design for model swappability from the start
7. Build flexible parsers that handle output format variation
8. Track actual costs during development, not just estimates

## Gotchas

1. **Skipping manual prototype**: Discovering that the model fundamentally cannot perform the task after building a full pipeline wastes weeks. The manual prototype step catches this in hours.

2. **Monolithic pipeline stages**: A single pipeline stage that acquires, processes, and renders in one pass prevents independent debugging. If the rendered output is wrong, you cannot determine which stage failed. Separate stages make failures attributable.

3. **Over-constraining with scaffolding**: Too many instructions, format requirements, and constraints reduce model performance. Models perform better with moderate guidance than with exhaustive prescriptions. Start with less scaffolding and add only when testing reveals specific failures.

4. **Ignoring costs until production**: Token costs that seem acceptable in development become prohibitive at production volume. Calculate production cost projections before building the pipeline, not after.

5. **Requiring perfect parsing**: LLMs produce output format variation. Parsers that require exact compliance break in production when the model deviates slightly. Build parsers that handle expected variations (leading/trailing whitespace, minor phrasing differences in headers).

6. **Model version lock-in**: Hardcoding model identifiers throughout the codebase makes upgrades expensive. Abstract behind a thin model wrapper that can be changed in one place.

7. **Deploying without quality measurement**: Shipping without evaluation means quality regressions from model updates go undetected until user complaints escalate. Establish evaluation before launch.

## Integration

This skill connects to:
- context-fundamentals - Token budget management for pipeline design
- tool-design - Tool design for agent-based pipelines
- multi-agent-patterns - When to escalate from single-agent
- evaluation - Quality measurement for LLM pipelines
- context-compression - Managing context in long-running pipeline sessions

## References

Related skills in this collection:
- tool-design - For designing tools used in agent pipelines
- multi-agent-patterns - For escalating from single-agent when justified
- evaluation - For measuring and maintaining pipeline quality

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
