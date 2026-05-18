---
name: tool-design
description: This skill should be used when the user asks to "design agent tools", "debug tool failures", "optimize tool sets", "create tool APIs", "fix tool misuse", or discusses tool description engineering, the consolidation principle, MCP tool naming, or agent tool architecture. Provides principles for designing unambiguous tool contracts for agents.
---

# Tool Design for Agents

Design every tool as a contract between a deterministic system and a non-deterministic agent. Unlike human-facing APIs, agent-facing tools must make the contract unambiguous through the description alone — agents infer intent from descriptions and generate calls that must match expected formats. Every ambiguity becomes a potential failure mode that no amount of prompt engineering can fix.

## When to Activate

Activate this skill when:
- Creating new tools for agent systems
- Debugging tool-related failures or misuse
- Optimizing existing tool sets for better agent performance
- Designing tool APIs from scratch
- Evaluating third-party tools for agent integration
- Standardizing tool conventions across a codebase

## Core Concepts

Design tools around the consolidation principle: if a human engineer cannot definitively say which tool should be used in a given situation, an agent cannot be expected to do better. Reduce the tool set until each tool has one unambiguous purpose, because agents select tools by comparing descriptions and any overlap introduces selection errors.

Treat every tool description as prompt engineering that shapes agent behavior. The description is not documentation for humans — it is injected into the agent's context and directly steers reasoning. Write descriptions that answer what the tool does, when to use it, and what it returns, because these three questions are exactly what agents evaluate during tool selection.

## Detailed Topics

### The Tool-Agent Interface

**Tools as Contracts**
Design each tool as a self-contained contract. When humans call APIs, they read docs, understand conventions, and make appropriate requests. Agents must infer the entire contract from a single description block. Make the contract unambiguous by including format examples, expected patterns, and explicit constraints. Omit nothing that a caller needs to know, because agents cannot ask clarifying questions before making a call.

**Tool Description as Prompt**
Write tool descriptions knowing they load directly into agent context and collectively steer behavior. A vague description like "Search the database" with cryptic parameter names forces the agent to guess — and guessing produces incorrect calls. Instead, include usage context, parameter format examples, and sensible defaults. Every word in the description either helps or hurts tool selection accuracy.

**Namespacing and Organization**
Namespace tools under common prefixes as the collection grows, because agents benefit from hierarchical grouping. When an agent needs database operations, it routes to the `db_*` namespace; when it needs web interactions, it routes to `web_*`. Without namespacing, agents must evaluate every tool in a flat list, which degrades selection accuracy as the count grows.

### The Consolidation Principle

**Single Comprehensive Tools**
Build single comprehensive tools instead of multiple narrow tools that overlap. Rather than implementing `list_users`, `list_events`, and `create_event` separately, implement `schedule_event` that finds availability and schedules in one call. The comprehensive tool handles the full workflow internally, removing the agent's burden of chaining calls in the correct order.

**Why Consolidation Works**
Each tool in the collection competes for attention during tool selection, each description consumes context budget tokens, and overlapping functionality creates ambiguity. Consolidation eliminates redundant descriptions, removes selection ambiguity, and shrinks the effective tool set. Vercel demonstrated this principle by reducing their agent from 17 specialized tools to 2 general-purpose tools and achieving better performance.

**When Not to Consolidate**
Keep tools separate when they have fundamentally different behaviors, serve different contexts, or must be callable independently. Over-consolidation creates a different problem: a single tool with too many parameters and modes becomes hard for agents to parameterize correctly.

### Architectural Reduction

Push the consolidation principle to its logical extreme by removing most specialized tools in favor of primitive, general-purpose capabilities. Production evidence shows this approach can outperform sophisticated multi-tool architectures.

**The File System Agent Pattern**
Provide direct file system access through a single command execution tool instead of building custom tools for data exploration, schema lookup, and query validation. The agent uses standard Unix utilities (grep, cat, find, ls) to explore and operate on the system. This works because file systems are a proven abstraction that models understand deeply, standard tools have predictable behavior, and agents can chain primitives flexibly.

**When Reduction Outperforms Complexity**
Choose reduction when the data layer is well-documented and consistently structured, the model has sufficient reasoning capability, specialized tools were constraining rather than enabling the model, or more time is spent maintaining scaffolding than improving outcomes.

### Tool Description Engineering

Structure every tool description to answer four questions:

1. What does the tool do? State exactly what the tool accomplishes — avoid vague language like "helps with" or "can be used for."
2. When should it be used? Specify direct triggers and indirect signals.
3. What inputs does it accept? Describe each parameter with types, constraints, defaults, and format examples.
4. What does it return? Document the output format, successful response examples, and error conditions.

**Default Parameter Selection**
Set defaults to reflect common use cases. Defaults reduce agent burden by eliminating unnecessary parameter specification and prevent errors from omitted parameters.

### Response Format Optimization

Offer response format options (concise vs. detailed) because tool response size significantly impacts context usage. Concise format returns essential fields only; detailed format returns complete objects. Document when to use each format so agents learn to select appropriately.

### Error Message Design

Design error messages for two audiences: developers debugging issues and agents recovering from failures. For agents, every error message must be actionable — it must state what went wrong and how to correct it. Include retry guidance for retryable errors, corrected format examples for input errors, and specific missing fields for incomplete requests.

### MCP Tool Naming Requirements

Always use fully qualified tool names with MCP (Model Context Protocol) to avoid "tool not found" errors.

Format: `ServerName:tool_name`

```python
# Correct: Fully qualified names
"Use the BigQuery:bigquery_schema tool to retrieve table schemas."
"Use the GitHub:create_issue tool to create issues."

# Incorrect: Unqualified names
"Use the bigquery_schema tool..."  # May fail with multiple servers
```

### Tool Collection Design

Limit tool collections to 10-20 tools for most applications. Research shows description overlap causes model confusion and more tools do not always lead to better outcomes. When more tools are genuinely needed, use namespacing to create logical groupings.

## Practical Guidance

### Tool Selection Framework

When designing tool collections:
1. Identify distinct workflows agents must accomplish
2. Group related actions into comprehensive tools
3. Ensure each tool has a clear, unambiguous purpose
4. Document error cases and recovery paths
5. Test with actual agent interactions

### Using Agents to Optimize Tools

Feed observed tool failures back to an agent to diagnose issues and improve descriptions. Production testing shows this approach achieves 40% reduction in task completion time.

```python
def optimize_tool_description(tool_spec, failure_examples):
    prompt = f"""
    Analyze this tool specification and the observed failures.

    Tool: {tool_spec}

    Failures observed:
    {failure_examples}

    Identify:
    1. Why agents are failing with this tool
    2. What information is missing from the description
    3. What ambiguities cause incorrect usage

    Propose an improved tool description that addresses these issues.
    """
    return get_agent_response(prompt)
```

## Examples

**Example 1: Well-Designed Tool**
```python
def get_customer(customer_id: str, format: str = "concise"):
    """
    Retrieve customer information by ID.

    Use when:
    - User asks about specific customer details
    - Need customer context for decision-making
    - Verifying customer identity

    Args:
        customer_id: Format "CUST-######" (e.g., "CUST-000001")
        format: "concise" for key fields, "detailed" for complete record

    Returns:
        Customer object with requested fields

    Errors:
        NOT_FOUND: Customer ID not found
        INVALID_FORMAT: ID must match CUST-###### pattern
    """
```

**Example 2: Poor Tool Design**
```python
def search(query):
    """Search the database."""
    pass
```
Problems: vague name, no parameter guidance, no return description, no usage context, no error handling.

## Guidelines

1. Write descriptions that answer what, when, and what returns
2. Use consolidation to reduce ambiguity
3. Implement response format options for token efficiency
4. Design error messages for agent recovery
5. Establish and follow consistent naming conventions
6. Limit tool count and use namespacing for organization
7. Test tool designs with actual agent interactions
8. Iterate based on observed failure modes
9. Prefer primitive, general-purpose tools over specialized wrappers
10. Always use fully qualified names with MCP servers

## Gotchas

1. **Vague descriptions**: Descriptions like "Search the database for customer information" leave too many questions unanswered. State the exact database, query format, and return shape.

2. **Cryptic parameter names**: Parameters named `x`, `val`, or `param1` force agents to guess meaning. Use descriptive names that convey purpose.

3. **Missing error recovery guidance**: Tools that fail with generic messages like "Error occurred" provide no recovery signal. Every error response must tell the agent what went wrong and what to try next.

4. **Inconsistent naming across tools**: Using `id` in one tool, `identifier` in another, and `customer_id` in a third creates confusion. Standardize parameter names across the entire tool collection.

5. **MCP namespace collisions**: When multiple MCP tool providers register tools with similar names, agents cannot disambiguate. Always use fully qualified `ServerName:tool_name` format.

6. **Tool description rot**: Descriptions become inaccurate as underlying APIs evolve. Treat descriptions as code: version them, review during API changes, and test against current behavior.

7. **Over-consolidation**: Making a single tool handle too many workflows produces parameter lists so large that agents struggle. If a tool requires more than 8-10 parameters or serves fundamentally different use cases, split it.

8. **Parameter explosion**: Too many optional parameters overwhelm agent decision-making. Provide sensible defaults and group related options into format presets.

## Integration

This skill connects to:
- context-fundamentals - How tools interact with context
- multi-agent-patterns - Specialized tools per agent
- evaluation - Evaluating tool effectiveness

## References

Related skills in this collection:
- context-fundamentals - Tool context interactions
- evaluation - Tool testing patterns

External resources:
- MCP (Model Context Protocol) documentation — When implementing tools for multi-server agent environments

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
