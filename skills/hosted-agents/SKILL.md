---
name: hosted-agents
description: This skill should be used when the user asks to "host agents remotely", "build background agents", "design sandboxed agent infrastructure", "optimize agent startup latency", or discusses remote execution environments, pre-built sandbox images, per-session state isolation, or production agent infrastructure. Provides architecture patterns for deploying agents in remote sandboxed environments.
---

# Hosted Agent Infrastructure

Build agents that run in remote sandboxed environments rather than locally. Remote execution eliminates the fundamental limits of local execution: resource contention, environment inconsistency, and single-user constraints. Well-designed hosted agent infrastructure achieves fast perceived startup through predictive warm-up and pre-built environment images.

## When to Activate

Activate this skill when:
- Building background agents that run without user supervision
- Designing multi-user agent platforms requiring isolation
- Optimizing agent startup latency for production workloads
- Architecting agent infrastructure that scales across users
- Implementing state management for persistent agent sessions

## Core Concepts

Move agent execution to remote sandboxed environments to eliminate the fundamental limits of local execution. Each session gets an isolated sandbox: its own filesystem, dependencies, and state. This prevents cross-session interference and enables consistent reproducible environments regardless of the user's local machine.

The fundamental performance challenge is startup latency. Sandboxes that take 30-60 seconds to initialize create unacceptable user experience. Solve this through predictive warm-up and pre-built environment images, not faster hardware.

## Detailed Topics

### Three Independent Scaling Layers

Design hosted agent infrastructure as three independently scalable layers:

**1. Sandbox Infrastructure**
Isolated execution environments for each agent session. Containers or VMs with:
- Clean filesystem per session
- Pre-installed dependencies
- Resource limits (CPU, memory, network)
- Automatic cleanup on session end

**2. API Layer**
State management and client coordination:
- Session lifecycle management (create, poll, terminate)
- Authentication and authorization
- Result streaming to client interfaces
- Queue management for concurrent sessions

**3. Client Interfaces**
Platform-specific frontends that communicate with the API layer:
- Web application
- Slack integration
- IDE extensions (VS Code, JetBrains)
- CLI client

Scaling each layer independently allows infrastructure to grow at different rates — sandbox capacity scales with usage, API layer scales with request volume, client interfaces scale with platform adoption.

### Predictive Warm-Up

Initiate sandbox preparation when users begin typing their prompt, not upon submission. The 5-30 second typing interval provides sufficient time to complete most setup work, dramatically reducing perceived latency.

**Warm-up sequence** (executes during user typing):
1. Allocate sandbox from warm pool
2. Clone repository at HEAD
3. Install dependencies
4. Run build steps
5. Start background services

By the time the user submits, the sandbox is ready. The agent starts immediately rather than waiting for setup.

### Pre-Build Strategy

Create environment images on a regular cadence (30-minute intervals recommended). Each image includes:
- Cloned repository with dependencies installed
- Completed build steps
- Warmed caches

Pre-built images mean users start from a ready state rather than waiting for setup. The warm pool maintains pre-initialized sandboxes available for immediate allocation.

**Image lifecycle**:
```
Build trigger (30 min interval)
  → Clone repo + install deps + build
  → Push to image registry
  → Warm pool pulls new image
  → Old images expire after N sessions or T hours
```

### Per-Session State Isolation

Isolate state per session. Cross-session interference is a subtle and hard-to-debug failure mode where one session's writes corrupt another session's reads.

**State isolation patterns**:
- SQLite database per session (not shared database)
- Session-scoped temporary directories
- No global mutable state in the agent runtime
- Explicit session ID in all state keys if sharing infrastructure is unavoidable

### Production Success Metrics

Track outcomes, not process metrics. Vanity metrics (sessions created, messages sent) do not correlate with value delivered. Track instead:
- **Merged PRs**: Primary metric for coding agents — did the agent produce shippable work?
- **Time-to-first-response**: Measures startup latency from user perspective
- **Approval rate**: What fraction of agent outputs are accepted without modification?
- **Agent-written code percentage**: Growing this metric indicates increasing trust and delegation

## Practical Guidance

### Sandbox Pool Management

Maintain a warm pool of pre-initialized sandboxes to eliminate allocation latency. Pool sizing depends on:
- Peak concurrent usage
- Sandbox initialization time
- Acceptable queue wait time

Over-provisioning warm pools wastes resources. Under-provisioning creates latency spikes. Monitor pool utilization and adjust size based on observed demand patterns.

### Session Lifecycle Management

Design explicit lifecycle states:
```
created → initializing → ready → running → completing → terminated
```

Handle each transition explicitly. Sessions that get stuck in `initializing` require timeout and cleanup logic. Sessions that crash in `running` require cleanup of allocated resources and notification to the client.

### Network and Resource Limits

Apply resource limits per sandbox to prevent runaway sessions from affecting others:
- CPU: Set a per-session CPU quota
- Memory: Hard limit with OOM termination, not swap
- Network: Allowlist required endpoints; block outbound by default
- Disk: Quota per session with enforcement

## Examples

**Example 1: Warm-Up Trigger**
```python
def on_user_typing_start(session_id: str):
    """Called when user begins typing a prompt."""
    if not warm_pool.has_available():
        # Can't warm-up synchronously, fall back to on-demand
        return
    
    sandbox = warm_pool.allocate()
    sandbox.associate_with_session(session_id)
    sandbox.start_background_setup()

def on_user_submit(session_id: str, prompt: str):
    """Called when user submits the prompt."""
    sandbox = get_sandbox_for_session(session_id)
    if sandbox.is_ready():
        sandbox.start_agent(prompt)  # Immediate start
    else:
        sandbox.wait_for_ready(timeout=30)  # Fallback
        sandbox.start_agent(prompt)
```

**Example 2: Per-Session State**
```python
class SessionState:
    def __init__(self, session_id: str):
        self.db_path = f"/tmp/sessions/{session_id}/state.db"
        self.workspace = f"/tmp/sessions/{session_id}/workspace"
        os.makedirs(self.workspace, exist_ok=True)
        self.db = sqlite3.connect(self.db_path)
```

## Guidelines

1. Design three independent scaling layers: sandbox, API, client interfaces
2. Implement predictive warm-up triggered by user typing, not submission
3. Pre-build environment images on a regular cadence
4. Isolate state per session — never use shared mutable state
5. Track outcome metrics (merged PRs, approval rate) not process metrics
6. Apply resource limits per sandbox to prevent cross-session interference
7. Design explicit session lifecycle states with timeout handling

## Gotchas

1. **Startup latency perception gap**: A 10-second wait feels unacceptable even if technically fast. Predictive warm-up is not optional for production systems — it is the difference between acceptable and unusable UX.

2. **Cross-session state bleeding**: Shared global state that seemed convenient in development causes intermittent, hard-to-reproduce bugs in production where sessions overlap. Per-session SQLite or per-session directories eliminate this class of bug entirely.

3. **Warm pool undersizing**: A warm pool that runs out at peak load forces on-demand initialization precisely when latency matters most. Monitor pool exhaustion rate and provision for peak + 20% headroom.

4. **Resource limit enforcement gaps**: Setting memory limits without enforcing them (relying on OS to swap rather than terminate) allows OOM conditions to degrade neighboring sandboxes. Use hard limits with OOM kill, not soft limits.

5. **Missing cleanup on crash**: Sessions that crash without cleanup leave allocated sandboxes in `running` state indefinitely, exhausting pool capacity. Implement heartbeat monitoring with automatic cleanup of unresponsive sessions.

## Integration

This skill connects to:
- multi-agent-patterns - Remote sub-agents as hosted infrastructure
- filesystem-context - Per-session filesystem state management
- evaluation - Measuring hosted agent performance in production

## References

Related skills in this collection:
- multi-agent-patterns - Coordination between hosted agents
- filesystem-context - Filesystem patterns within sandboxed environments

---

## Skill Metadata

**Created**: 2025-12-20
**Last Updated**: 2026-03-17
**Author**: Agent Skills for Context Engineering Contributors
**Version**: 2.0.0
