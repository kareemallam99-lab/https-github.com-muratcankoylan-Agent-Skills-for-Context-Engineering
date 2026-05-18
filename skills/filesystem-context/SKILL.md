---
name: filesystem-context
description: This skill should be used when the user asks to "use filesystem as memory", "manage context overflow with files", "implement scratch pad patterns", "store agent outputs to files", or discusses filesystem-based context engineering, dynamic skill loading, sub-agent file communication, or structured file formats for agent state. Provides patterns for using the filesystem as an overflow layer for agent memory.
---

# Filesystem-Based Context Engineering

Use the filesystem as an overflow layer for agent memory and information retrieval when context windows cannot hold all relevant information. Files provide effectively unlimited storage accessible through a single interface that agents understand deeply — standard Unix utilities (grep, cat, find, ls) work without custom tooling.

## When to Activate

Activate this skill when:
- Agent faces context window limitations and needs overflow storage
- Tool outputs exceed 2,000 tokens and flood context
- Tasks span multiple conversation turns requiring state persistence
- Building dynamic skill loading systems
- Routing sub-agent findings through isolated directories
- Implementing just-in-time context loading patterns

## Core Concepts

Files let agents store, retrieve, and update an effectively unlimited amount of context through a single interface. Unlike in-memory state, file-based context survives context resets, can be shared across sub-agents without message-passing, and is inspectable by humans.

The filesystem approach addresses four distinct context failure modes:
1. **Missing context** — Information loss; solve by persisting outputs to files
2. **Under-retrieved context** — Retrieved data lacks necessary details; improve through structured, searchable formats
3. **Over-retrieved context** — Excessive data wastes tokens; offload bulk content and return references instead
4. **Buried context** — Niche information scattered across files; combine structural search (grep/glob) with semantic search

Adopt this pattern when tool outputs exceed ~2,000 tokens or tasks span multiple conversation turns. The I/O overhead is justified at that scale.

## Detailed Topics

### Key Filesystem Patterns

**Scratch Pad Pattern**
Large tool outputs redirect to files rather than flooding context. Instead of returning 8,000 tokens from a search query:
1. Write the output to a scratch file (`scratch/search_results_001.json`)
2. Extract a compact summary from the file
3. Return a file reference plus the summary to the agent

The agent can re-read the full file if needed, but only the summary enters the working context.

**Plan Persistence**
Store plans in structured YAML so agents can re-read objectives and progress, maintaining coherence across long tasks and context resets.

```yaml
# task_plan.yaml
objective: Implement authentication middleware
phases:
  - name: Design
    status: complete
    outputs: [docs/auth_design.md]
  - name: Implementation
    status: in_progress
    files_modified: [src/middleware/auth.py]
  - name: Testing
    status: pending
```

**Sub-Agent Communication via Filesystem**
Route findings through isolated filesystem directories instead of message chains. Each sub-agent writes its outputs to a designated directory; the coordinator reads from those directories rather than receiving messages. This preserves information fidelity — file contents do not degrade through repeated summarization.

```
coordination/
├── researcher/output.md
├── analyzer/findings.json
└── synthesizer/draft.md
```

**Dynamic Skill Loading**
Store skill definitions as files. Include only brief descriptions in static context to avoid token waste. Load full skill content on activation by reading the relevant file.

```
skills/
├── context-fundamentals/SKILL.md  # loaded on activation
├── memory-systems/SKILL.md        # loaded on activation
└── index.md                       # always loaded (names + descriptions only)
```

### File Format Selection

Match file format to access patterns:
- **JSON**: Structured data requiring programmatic access, easy to parse with `jq`
- **Markdown**: Human-readable documents, plans, notes — grep-friendly
- **YAML**: Configuration and state — readable and parseable
- **Plain text**: Logs, raw outputs — maximum compatibility

Use consistent naming conventions that convey content without reading: `search_results_2024-01-15.json`, `implementation_plan_v2.yaml`, `auth_module_notes.md`.

### Structural vs. Semantic Search

For filesystem context, combine structural search with semantic search:
- **Structural** (grep, glob, find): Use for known file paths, specific identifiers, function names, error codes — fast and exact
- **Semantic** (embedding-based): Use for concept retrieval when the exact location is unknown — slower but handles vagueness

Default to structural search first. Structural search is faster, more reliable, and does not require embeddings infrastructure. Escalate to semantic search only when structural search fails.

## Practical Guidance

### Measurement and Monitoring

Track two ratios to validate filesystem context effectiveness:
1. **Static/dynamic context ratio**: What fraction of context is pre-loaded vs. dynamically fetched? Target more dynamic loading.
2. **File utilization rate**: What fraction of dynamically loaded files are actually used in reasoning? Low rates indicate over-fetching.

### Retention Policies

Scratch directories grow unbounded without explicit policies. Implement:
- **Time-based expiry**: Delete scratch files older than N hours/days
- **Session scoping**: Clean scratch directory at session end
- **Size limits**: Archive or delete when directory exceeds a size threshold

### Concurrency Safety

When multiple sub-agents write to shared filesystems:
- Use file-level locking for critical state files
- Assign each sub-agent a private directory for outputs
- Use an append-only log pattern for shared event streams

## Examples

**Example 1: Scratch Pad Implementation**
```python
def search_with_scratch_pad(query: str) -> str:
    results = web_search(query)  # Returns 8,000 tokens
    
    # Write to scratch
    scratch_path = f"scratch/search_{timestamp()}.json"
    write_file(scratch_path, results)
    
    # Extract compact summary
    summary = extract_top_results(results, n=5)
    
    # Return reference + summary
    return f"[{scratch_path}: {len(results)} results — top 5: {summary}]"
```

**Example 2: Dynamic Skill Index**
```markdown
# skills/index.md (always loaded)

## Available Skills

- **context-fundamentals**: Core context mechanics, attention budgets, progressive disclosure
- **memory-systems**: Framework selection (Mem0, Zep, Letta, Cognee), retrieval strategies
- **tool-design**: Consolidation principle, description engineering, MCP naming

[Load full skill content with: read_file("skills/{skill-name}/SKILL.md")]
```

## Guidelines

1. Use filesystem patterns when tool outputs exceed ~2,000 tokens
2. Implement naming conventions that convey content without reading
3. Combine structural search (grep) with semantic search — use structural first
4. Store plans in structured formats (YAML/JSON) not free-form text
5. Route sub-agent communication through directories, not message chains
6. Implement retention policies before scratch directories grow unbounded
7. Measure static/dynamic context ratio and file utilization rate
8. Use file-level locking when multiple agents write shared state

## Gotchas

1. **Unbounded scratch directory growth**: Without retention policies, scratch directories fill storage and slow filesystem search. Implement expiry from day one.

2. **Concurrent write corruption**: Multiple sub-agents writing to the same file without locking silently corrupts data. Assign private output directories per sub-agent or implement file-level locking.

3. **Stale cached paths**: File paths stored in context become stale when files are reorganized. Cache paths only for the duration of a session, not across sessions.

4. **Large file reads without size checks**: Reading a large file without checking its size first dumps excessive tokens into context, defeating the purpose of filesystem offloading. Always check file size before reading; paginate or summarize large files.

5. **Unstructured scratch formats**: Scratch files written as unstructured text become unparseable over multiple iterations. Use consistent JSON or YAML structure from the first write.

## Integration

This skill connects to:
- context-fundamentals - Context overflow and just-in-time loading principles
- multi-agent-patterns - Filesystem coordination between agents
- memory-systems - When to escalate from filesystem to structured memory systems

## References

Related skills in this collection:
- context-fundamentals - Foundation for understanding when filesystem patterns are needed
- memory-systems - When filesystem memory is insufficient and structured memory is needed
- multi-agent-patterns - Filesystem as the coordination layer between agents

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
