---
name: context-engineering
description: This skill collection should be used when the user asks to build, optimize, debug, or design AI agent systems — especially when context management, multi-agent coordination, memory architectures, tool design, or agent evaluation is involved. Provides a comprehensive framework covering context fundamentals through production-grade evaluation methodologies. Version 1.3.0 (Updated April 14, 2026).
---

# Agent Skills for Context Engineering

This collection addresses context engineering across three core themes: **context fundamentals** (understanding attention mechanisms and information quality), **architectural patterns** (multi-agent coordination, memory systems, tool design), and **operational excellence** (compression, optimization, evaluation).

## Skill Map

### Foundational Skills

| Skill | Activate When |
|-------|--------------|
| `context-fundamentals` | Designing agent systems, debugging context issues, optimizing context usage, onboarding to context engineering |
| `context-degradation` | Agent performance degrades, debugging failures, investigating lost-in-middle, analyzing context failures |
| `context-compression` | Long sessions need history management, applying summarization strategies |

### Architectural Skills

| Skill | Activate When |
|-------|--------------|
| `multi-agent-patterns` | Single-agent context limits constrain complexity, designing parallel or specialized agent systems |
| `memory-systems` | Building agents with cross-session persistence, choosing memory frameworks, evaluating Mem0/Zep/Letta/Cognee |
| `tool-design` | Creating agent tools, debugging tool misuse, optimizing tool sets |
| `filesystem-context` | Agent faces context overflow, using filesystem as memory overflow layer |
| `hosted-agents` | Building remote sandboxed agent infrastructure |

### Operational Skills

| Skill | Activate When |
|-------|--------------|
| `context-optimization` | Reducing token costs, implementing KV-cache optimization, applying compaction |
| `latent-briefing` | Designing orchestrator-worker systems, debugging token explosion in recursive agents |
| `evaluation` | Setting up agent evaluation frameworks, measuring agent quality |
| `advanced-evaluation` | Implementing LLM-as-Judge systems, mitigating evaluation bias |

### Development & Cognitive Skills

| Skill | Activate When |
|-------|--------------|
| `project-development` | Starting LLM-powered projects, evaluating task-model fit, designing pipelines |
| `bdi-mental-states` | Implementing BDI cognitive architectures, neuro-symbolic AI integration |

## Core Principle

Context is not just prompt text — it is the complete state available to the language model at inference time: system instructions, tool definitions, retrieved documents, message history, and tool outputs. The engineering challenge is maximizing utility per token against three constraints:

1. The hard token limit
2. The softer effective-capacity ceiling (typically 60-70% of the advertised window)
3. The U-shaped attention curve that penalizes information placed in the middle of context

## Collection Version

**Version**: 1.3.0
**Updated**: April 14, 2026
**Skills**: 14
