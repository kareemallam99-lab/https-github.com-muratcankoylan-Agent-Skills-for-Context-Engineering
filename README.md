# Agent Skills for Context Engineering

A comprehensive collection of Agent Skills focused on context engineering principles for building production-grade AI agent systems.

## What is Context Engineering?

Context engineering is the discipline of curating everything that enters a model's context window — system prompts, tool definitions, retrieved documents, message history, and tool outputs — to maximize signal within a limited attention budget. Unlike prompt engineering, which focuses on crafting effective instructions, context engineering addresses the holistic curation of all information entering the model's limited attention budget.

## Skill Categories

### Foundational Skills
- **context-fundamentals** — Core concepts: context anatomy, attention mechanics, progressive disclosure, and budget allocation
- **context-degradation** — Five degradation patterns (lost-in-middle, poisoning, distraction, confusion, clash) with detection and mitigation
- **context-compression** — Strategies for compressing conversation history in long-running sessions

### Architectural Skills
- **multi-agent-patterns** — Supervisor, peer-to-peer, and hierarchical patterns with context isolation as the primary design principle
- **memory-systems** — Memory layer selection (Mem0, Zep/Graphiti, Letta, Cognee, LangMem) with benchmark comparisons
- **tool-design** — Designing unambiguous tool contracts for agents; the consolidation principle
- **filesystem-context** — Using the filesystem as an overflow layer for agent memory
- **hosted-agents** — Building background agents in remote sandboxed environments

### Operational Skills
- **context-optimization** — KV-cache optimization, observation masking, compaction, and context partitioning
- **latent-briefing** — Sharing orchestrator KV-cache state with workers at the representation level
- **evaluation** — Outcome-focused evaluation frameworks for non-deterministic agent behavior
- **advanced-evaluation** — LLM-as-Judge techniques with bias mitigation

### Development & Cognitive Skills
- **project-development** — Task-model fit assessment, pipeline architecture, and agent development methodology
- **bdi-mental-states** — Belief-Desire-Intention architecture for cognitive agent modeling

## Installation & Usage

### Claude Code
```
/plugin install context-engineering@context-engineering-marketplace
```

Or register via:
```
/plugin marketplace add muratcankoylan/Agent-Skills-for-Context-Engineering
```

### Individual Skills
Copy specific skill markdown files to `.claude/skills/` directory.

### Cursor
Listed on the Cursor Plugin Directory following Open Plugins standards.

## Examples

- **digital-brain-skill** — Personal OS for founders with 6 modules and 4 automation scripts
- **x-to-book-system** — Multi-agent monitoring and synthesis platform
- **llm-as-judge-skills** — Production TypeScript evaluation tools with 19 passing tests
- **book-sft-pipeline** — Style transfer training achieving 70% human score at $2 cost

## Design Principles

- **Context quality over quantity** — attention scarcity and the lost-in-middle phenomenon mean more context is not always better
- **Sub-agents isolate context** — they exist to manage attention budget, not simulate org roles
- **Progressive disclosure** — only skill names/descriptions load at startup; full content loads on activation
- **Platform-agnostic** — transferable principles that work across Claude Code, Cursor, and any agent framework

## License

MIT License
