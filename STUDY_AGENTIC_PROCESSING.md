# OpenClaw Agentic Processing — Deep Code Study & Improvement Plan

## Executive Summary

OpenClaw (ai-prentice) is a sophisticated personal AI assistant platform with a rich agentic processing pipeline. This document provides a detailed architectural analysis of how agent intelligence is orchestrated, and proposes concrete enhancements to push the system towards more autonomous, planning-aware, and adaptive agent behavior.

---

## 1. Architecture Overview

### 1.1 High-Level Structure

```
┌──────────────────────────────────────────────────────────┐
│                     Gateway (Control Plane)               │
│   RPC server, session routing, channel multiplexing       │
├──────────────────────────────────────────────────────────┤
│                        Channels                           │
│   WhatsApp, Telegram, Discord, Slack, iMessage, ...       │
├──────────────────────────────────────────────────────────┤
│                    Agent Command Layer                     │
│   Session mgmt, model selection, delivery orchestration   │
├──────────────────────────────────────────────────────────┤
│                 Embedded Agent Runner                      │
│   Run loop, retries, failover, compaction, context engine │
├──────────────────────────────────────────────────────────┤
│            Tool Surface + ACP (Sub-agents)                 │
│   60+ built-in tools, plugin tools, MCP tools             │
├──────────────────────────────────────────────────────────┤
│            Provider Layer (LLM Transport)                  │
│   OpenAI, Anthropic, Google, local models, 80+ providers  │
├──────────────────────────────────────────────────────────┤
│               Plugin SDK + Extensions                      │
│   ~140 extension directories, skills, hooks               │
└──────────────────────────────────────────────────────────┘
```

### 1.2 Key Source Directories

| Directory | Purpose |
|-----------|---------|
| `src/agents/` | Core agent orchestration (2,400+ files) |
| `src/agents/embedded-agent-runner/` | The actual LLM run loop |
| `src/agents/tools/` | 70+ built-in tool implementations |
| `src/acp/` | Agent Collaboration Protocol (sub-agent spawning) |
| `src/context-engine/` | Pluggable context management lifecycle |
| `src/skills/` | Skill discovery, loading, and runtime |
| `src/gateway/` | Gateway server, RPC methods, auth |
| `src/channels/` | Channel plugins and transport layer |
| `src/plugin-sdk/` | SDK for external plugins |
| `extensions/` | ~140 provider/channel/tool plugins |
| `packages/` | Shared libraries (acp-core, llm-core, etc.) |

---

## 2. Agent Run Lifecycle (Current)

### 2.1 Entry Point

The main orchestration lives in `src/agents/embedded-agent-runner/run.ts` (4,064 lines). The flow:

```
runEmbeddedAgent(params)
  → runEmbeddedAgentInternal(params)
    → resolve workspace, plugins, model
    → run hooks (before_agent_reply, before_model_resolve)
    → select agent harness (builtin "openclaw" or plugin harness)
    → resolve model + auth profiles
    → enter retry/failover loop:
        → runEmbeddedAttemptWithBackend()
          → build system prompt
          → assemble context (via context-engine)
          → stream LLM response
          → execute tool calls
          → handle tool results
          → loop until model returns final text
        → handle compaction if overflow
        → handle failover if provider error
        → handle rate limits, auth rotation
    → emit events, update session state
    → return payloads
```

### 2.2 Context Engine (Pluggable)

The Context Engine (`src/context-engine/`) is a plugin-owned lifecycle that controls what the LLM "sees":

- **`bootstrap()`** — Initialize context for a new session
- **`maintain()`** — Background maintenance (prune, reindex)
- **`ingest()` / `ingestBatch()`** — Absorb new messages into the store
- **`assemble()`** — Build the final message array sent to the LLM
- **`compact()`** — Summarize and compress when context overflows
- **`afterTurn()`** — Post-turn housekeeping

The default engine is simple message-history replay. Plugin engines (like `memory-lancedb`) provide RAG/vector-search augmentation.

### 2.3 Tool Execution Pipeline

Tool execution flows through:

1. **Tool Surface Assembly** (`agent-tools.ts`) — Builds available tools from: core shell/read/write/edit tools, channel-specific tools, OpenClaw platform tools, plugin-contributed tools, MCP server tools, skill-workshop tools.

2. **Before-Tool-Call Hook** (`agent-tools.before-tool-call.ts`) — Plugin hooks, trusted tool policies, approval gates, loop detection, diagnostics.

3. **Tool Loop Detection** (`tool-loop-detection.ts`) — Monitors for:
   - Generic repeated identical tool calls
   - Unknown (hallucinated) tool name loops
   - Ping-pong patterns between two tools
   - Poll-without-progress patterns
   - Global circuit breaker (30 consecutive)

4. **Sandbox Enforcement** — Docker/SSH/OpenShell sandbox for non-main sessions.

### 2.4 Sub-Agent / ACP System

The Agent Collaboration Protocol (`src/acp/`) enables hierarchical multi-agent work:

- **`sessions_spawn` tool** — The model can spawn child sessions with isolated tool scopes
- **Subagent Registry** (`subagent-registry.ts`) — Tracks active children per session
- **Steering Queue** (`agent-steering-queue.ts`) — Delivers completed child results back to the parent
- **ACP Control Plane** (`src/acp/control-plane/`) — Session lifecycle, identity reconciliation, backend failover
- **Depth Limits** — Configurable max spawn depth (default 3) and max children per agent
- **Context Modes** — `"fork"` (inherit parent transcript) or isolated

### 2.5 System Prompt Construction

The system prompt (`src/agents/system-prompt.ts`, ~1200 lines) assembles:

- Identity and personality (SOUL.md, AGENTS.md)
- Runtime info (OS, model, capabilities)
- Tool descriptions and guidance
- Workspace/skills context
- Sub-agent delegation guidance (mode: "suggest" or "prefer")
- Memory section (if active-memory plugin is active)
- Channel-specific behavior guidance
- Owner identification and security posture

### 2.6 Model Failover & Auth Rotation

The runner has sophisticated error handling:
- **Cross-provider failover** — Falls back to configured fallback models
- **Auth profile rotation** — Cycles through API keys on rate limits
- **Compaction on overflow** — Summarizes history when context exceeds budget
- **Idle timeout breaker** — Retries when model hangs
- **Incomplete-turn recovery** — Retries empty responses, reasoning-only turns

---

## 3. Current Agentic Capabilities Assessment

### 3.1 Strengths

| Capability | Status | Notes |
|-----------|--------|-------|
| Multi-turn conversation | Mature | Session persistence, compaction |
| Tool use | Mature | 60+ tools, approval gates, loop detection |
| Multi-agent orchestration | Mature | ACP + sessions_spawn + steering queue |
| Provider diversity | Mature | 80+ LLM providers supported |
| Channel multiplexing | Mature | 25+ messaging platforms |
| Memory/RAG | Available | Via `memory-core` + `memory-lancedb` plugins |
| Planning | Basic | `update_plan` tool (structured step list) |
| Goal tracking | Basic | `create_goal` / `get_goal` / `update_goal` tools |
| Execution contracts | Emerging | GPT-5 "strict agentic" execution mode |
| Skills | Mature | Workspace/managed/bundled skill system |

### 3.2 Gaps & Limitations

| Area | Current State | Impact |
|------|--------------|--------|
| **Autonomous planning** | Passive (model decides ad-hoc) | Agent doesn't proactively decompose complex tasks |
| **Task dependency graphs** | None | No DAG-based execution or parallel planning |
| **Self-reflection** | None built-in | No "did I succeed?" verification loop |
| **Adaptive strategy** | Provider-level failover only | No task-level strategy adjustment |
| **Long-horizon state** | Compaction summarizes away | Loses detail on multi-session projects |
| **Tool composition** | Sequential only | No declarative workflows or pipelines |
| **Learning from failure** | Per-session only | No cross-session pattern learning |
| **Proactive behavior** | Cron + heartbeat only | No goal-driven autonomous scheduling |
| **Budget-aware execution** | Goal token budget only | No cost-aware routing for sub-tasks |
| **Observation-action loop** | Single-turn reactive | No continuous environment monitoring |

---

## 4. Improvement Plan: Enhanced Agentic Processing

### Phase 1: Structured Planning & Decomposition (Foundation)

#### 4.1.1 Hierarchical Task Planner

**What:** Replace the flat `update_plan` tool with a proper task planner that decomposes goals into dependency-aware sub-tasks.

**Where to implement:**
- New module: `src/agents/planning/` (planner core)
- Extend: `src/agents/tools/update-plan-tool.ts` → `plan-tool.ts`
- Integrate with: `src/agents/embedded-agent-runner/run.ts` (run decisions)

**Design:**
```typescript
type TaskNode = {
  id: string;
  objective: string;
  status: "pending" | "in_progress" | "completed" | "blocked" | "failed";
  dependencies: string[];  // task ids that must complete first
  strategy: "direct" | "delegate" | "parallel";
  estimatedComplexity: "trivial" | "simple" | "moderate" | "complex";
  tools: string[];  // likely tools needed
  verificationCriteria: string;  // how to know it succeeded
};

type ExecutionPlan = {
  goal: string;
  tasks: TaskNode[];
  criticalPath: string[];  // ordered task ids
  parallelGroups: string[][];  // groups that can run concurrently
};
```

**Integration points:**
- System prompt gets a "planning mode" section when plan exists
- The runner checks plan state before each attempt to determine next action
- Sub-agent spawns are auto-tagged with plan task IDs for tracking

#### 4.1.2 Pre-Execution Analysis

**What:** Before the first LLM call for a complex prompt, run a lightweight "analysis pass" that classifies the task and optionally generates a plan.

**Where:**
- New: `src/agents/embedded-agent-runner/run/pre-analysis.ts`
- Hook into: `runEmbeddedAgentInternal()` before the main attempt loop

**Mechanism:**
- Use a smaller/faster model (or the same model with constrained output) to produce a structured task analysis
- Cache the analysis for the session so it doesn't repeat on retries
- Feed the plan into the system prompt for the main execution

---

### Phase 2: Self-Verification & Reflection (Quality)

#### 4.2.1 Post-Action Verification Loop

**What:** After tool execution, optionally verify the outcome matches intent before continuing.

**Where:**
- Extend: `src/agents/agent-tools.before-tool-call.ts` (add an "after-tool-result" verification hook)
- New: `src/agents/verification/` module

**Design:**
```typescript
type VerificationPolicy = {
  // Which tools trigger verification
  triggerTools: string[];
  // Verification strategy
  mode: "self_check" | "schema_validate" | "assertion_eval";
  // Max retries on verification failure
  maxRetries: number;
};
```

**Key rules:**
- Destructive tools (write, exec with side effects) get mandatory post-verification
- Non-destructive tools (read, search) skip verification
- Verification failures insert a "correction prompt" before the next LLM call

#### 4.2.2 Execution Outcome Scoring

**What:** At end-of-turn, the agent scores its own success and records it for learning.

**Where:**
- Extend: `src/agents/embedded-agent-runner/run.ts` (post-attempt)
- Store in: `src/agents/trajectory/` (trajectory already exists for recording)

**Mechanism:**
- After final assistant text is produced, inject a brief self-assessment prompt
- Record: task classification, tools used, outcome score, failure patterns
- Feed patterns into future planning (e.g., "last time tool X failed on task type Y")

---

### Phase 3: Adaptive Multi-Agent Execution (Scale)

#### 4.3.1 Intelligent Work Distribution

**What:** Enhance the sub-agent spawning to be plan-aware and strategy-driven rather than ad-hoc.

**Where:**
- Extend: `src/agents/subagent-spawn.ts`
- Extend: `src/agents/acp-spawn.ts`
- New: `src/agents/planning/distributor.ts`

**Enhancements:**
- Auto-detect parallelizable sub-tasks from the plan DAG
- Spawn sub-agents with explicit task contexts and verification criteria
- Implement a "coordinator" pattern where the parent monitors progress and intervenes
- Add budget allocation per sub-agent (token budget, time budget)

#### 4.3.2 Cross-Session Knowledge Transfer

**What:** Enable learning across sessions so repeated task patterns improve over time.

**Where:**
- Extend: `extensions/memory-core/`
- New: `src/agents/pattern-memory/`

**Design:**
- Track: successful tool sequences for task types, common failure modes, effective decomposition patterns
- Retrieve: at planning time, query pattern memory for similar past tasks
- Feed into system prompt as "past experience" context

---

### Phase 4: Proactive & Goal-Driven Behavior (Autonomy)

#### 4.4.1 Goal Persistence & Monitoring

**What:** Extend the basic goal system to support multi-session goals with progress tracking.

**Where:**
- Extend: `src/agents/tools/goal-tools.ts`
- Extend: `src/cron/` (existing cron infrastructure)
- New: `src/agents/goals/` persistent goal store

**Design:**
```typescript
type PersistentGoal = {
  id: string;
  objective: string;
  status: "active" | "paused" | "completed" | "abandoned";
  milestones: GoalMilestone[];
  schedule?: CronExpression;  // proactive check-ins
  triggers?: GoalTrigger[];   // events that should resume work
  successCriteria: string;
  progressHistory: GoalProgressEntry[];
};
```

#### 4.4.2 Environment-Aware Agent Loop

**What:** Add a continuous observation layer that can detect state changes and trigger agent actions.

**Where:**
- Extend: `src/hooks/` (existing hook infrastructure)
- Integrate with: `src/cron/` for periodic polling
- New: `src/agents/observers/`

**Use cases:**
- File system watcher → detect when watched files change
- Webhook ingestion → react to external events
- Scheduled environment probes → check service health, repo state

---

### Phase 5: Enhanced Reasoning & Tool Composition (Intelligence)

#### 4.5.1 Tool Pipelines / Workflows

**What:** Allow the agent to define and execute multi-step tool workflows as atomic units.

**Where:**
- New: `src/agents/workflows/`
- Integrate with tool surface in `src/agents/agent-tools.ts`

**Design:**
```typescript
type ToolWorkflow = {
  name: string;
  steps: WorkflowStep[];
  errorHandler: "abort" | "retry" | "fallback";
};

type WorkflowStep = {
  tool: string;
  params: Record<string, string | WorkflowRef>;
  outputKey: string;  // captured for downstream steps
  condition?: string;  // skip if false
};
```

#### 4.5.2 Reasoning Trace Enhancement

**What:** Make the thinking/reasoning trace actionable — extract structured insights from CoT that inform subsequent actions.

**Where:**
- Extend: `src/agents/embedded-agent-runner/thinking.ts`
- New: `src/agents/reasoning/trace-parser.ts`

**Mechanism:**
- Parse reasoning output for:
  - Identified uncertainties → trigger verification
  - Alternative approaches → record for fallback
  - Resource needs → pre-allocate (e.g., spawn sub-agents early)
  - Risk flags → adjust execution policy (be more cautious)

---

## 5. Implementation Priority & Effort

| Phase | Feature | Effort | Impact | Priority |
|-------|---------|--------|--------|----------|
| 1 | Hierarchical Task Planner | Medium | High | **P0** |
| 1 | Pre-Execution Analysis | Low | Medium | **P0** |
| 2 | Post-Action Verification | Medium | High | **P1** |
| 2 | Outcome Scoring | Low | Medium | **P1** |
| 3 | Intelligent Work Distribution | High | High | **P1** |
| 3 | Cross-Session Knowledge | High | High | **P2** |
| 4 | Goal Persistence | Medium | Medium | **P2** |
| 4 | Environment Observers | Medium | Medium | **P2** |
| 5 | Tool Pipelines | Medium | Medium | **P3** |
| 5 | Reasoning Trace Enhancement | Low | Medium | **P3** |

---

## 6. Recommended First Steps

### Immediate (Week 1-2)

1. **Create `src/agents/planning/` module** with:
   - `task-graph.ts` — DAG data structure for task dependencies
   - `decomposer.ts` — LLM-powered task decomposition
   - `scheduler.ts` — Critical path + parallel group resolution
   - `plan-store.ts` — SQLite-backed persistence per session

2. **Extend `update_plan` → `plan` tool** to accept richer structured plans with dependencies and verification criteria.

3. **Add pre-analysis gate** in `runEmbeddedAgentInternal()`:
   ```typescript
   // After workspace resolution, before main loop
   if (shouldRunPreAnalysis(params)) {
     const analysis = await runPreAnalysisPass(params);
     params = { ...params, preAnalysis: analysis };
   }
   ```

### Short-term (Week 3-4)

4. **Implement verification hooks** for destructive tools (write, exec, apply_patch).

5. **Wire plan-aware sub-agent spawning** — When the plan has parallel groups, auto-suggest `sessions_spawn` with proper task context.

### Medium-term (Month 2)

6. **Cross-session pattern memory** using the existing `memory-core` vector store.
7. **Goal persistence** leveraging the existing `state/openclaw.sqlite` store.

---

## 7. Integration Notes

### Respecting Existing Architecture

All proposed changes should follow these existing patterns:

- **Plugin-first:** New capabilities should be implementable as plugins where possible. The context-engine pattern is a good model — define a typed interface, provide a default implementation, allow plugins to override.

- **SQLite-backed state:** Use Kysely helpers, not raw SQL. Prefer the shared state DB or agent DB over new file stores.

- **Config-driven:** New features get `openclaw.json` configuration knobs following existing patterns. Keep the config surface minimal — prefer smart defaults.

- **Hook integration:** Use the existing hook lifecycle (`before_agent_run`, `llm_output`, `before_tool_call`, etc.) rather than inventing new extension points.

- **Sandbox-safe:** All new agent capabilities must work within the existing sandbox model. Sub-agents inherit tool policy from parent.

### Key Files to Modify

| File | Change |
|------|--------|
| `src/agents/embedded-agent-runner/run.ts` | Add pre-analysis gate, plan-aware retry |
| `src/agents/tools/update-plan-tool.ts` | Extend schema for DAG plans |
| `src/agents/subagent-spawn.ts` | Plan-aware child spawning |
| `src/agents/system-prompt.ts` | Planning context sections |
| `src/agents/agent-tools.before-tool-call.ts` | Post-execution verification |
| `src/context-engine/types.ts` | Plan-aware assembly hints |
| `src/agents/embedded-agent-runner/system-prompt.ts` | Dynamic plan injection |

---

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Increased latency from pre-analysis | Make it opt-in via config; use cheaper model; cache aggressively |
| Token overhead from plan context | Compact plan representation; only inject active/next steps |
| Over-delegation to sub-agents | Enforce complexity threshold; "suggest" mode by default |
| Verification loops | Hard cap on retries; timeout-based escape hatch |
| Cross-session memory noise | Relevance scoring; decay factor; manual curation tools |
| Config complexity | Smart defaults; `openclaw doctor` validates config |

---

## 9. Conclusion

OpenClaw already has excellent foundations for agentic processing — the tool system is rich, sub-agent orchestration is production-grade, and the provider layer is highly resilient. The main gaps are in **proactive planning**, **self-verification**, and **cross-session learning**. 

The proposed improvements build incrementally on the existing architecture without requiring fundamental rewrites. Phase 1 (planning + pre-analysis) alone would significantly improve the agent's ability to handle complex multi-step tasks, and it integrates naturally with the existing sub-agent and tool systems.

The `execution-contract.ts` pattern (GPT-5 strict agentic mode) suggests the project is already moving towards richer execution semantics — the planning framework proposed here extends that direction into task-level intelligence.
