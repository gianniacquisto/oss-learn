# Research Notes: pi.dev Architecture Analysis

## Source Commit: 3911d6f5 (main branch)

## Key Architectural Insights

### The Agent Loop Pattern
pi.dev's agent loop (`packages/agent/src/agent-loop.ts`) has two layers:
- **Outer loop:** handles follow-up messages that arrive after the agent would normally stop. This supports "queue mode" where users can type while the agent is working.
- **Inner loop:** processes tool calls from a single assistant response, either sequentially or in parallel (configurable per-tool).

The key insight: the loop doesn't just call the LLM once — it's a state machine that transitions between "prompting," "executing tools," and "deciding what to do next."

### Message Abstraction
`AgentMessage` is a union of standard LLM messages (`user`, `assistant`, `toolResult`) plus custom application-specific message types. This allows pi to add UI-only notifications, status updates, etc., without polluting the LLM's context. The `convertToLlm` function filters out non-LLM messages before each provider call.

### Tool System Design
Tools in pi.dev use TypeBox schemas (`TSchema`) for parameter validation. Each tool has:
- A name and description (for the LLM to decide when to call it)
- A Zod/TypeBox schema (for runtime argument validation)
- An `execute` function that takes `(toolCallId, args, signal)` — the signal enables cancellation

Tools can be marked as "sequential" (must execute one at a time) or "parallel" (can execute concurrently). This is per-tool, not global.

### Event System
The agent emits events for every lifecycle stage: `agent_start`, `turn_start`, `message_update` (during streaming), `tool_execution_end`, etc. UIs subscribe to these and render independently. The event system uses async-await listeners — each listener is awaited before the next event fires, ensuring ordering.

### State Management
The Agent class owns mutable state internally but exposes it through readonly getters. Arrays are copied on assignment (set) to prevent external mutation. This is a common pattern in reactive systems: controlled mutability with immutable access.

## What Was Cut for the MVP

1. **Compaction/summarization** — pi.dev compresses long conversations to stay within context windows. Not needed for MVP; learners can add it as an extension.
2. **Session persistence** — JSONL file storage, memory repo, UUID generation. The MVP uses in-memory messages only.
3. **Multi-provider support** — the MVP implements one provider (OpenAI or Anthropic). Adding the other follows the same pattern.
4. **MCP integration** — pi.dev's tool system is designed to be MCP-compatible, but that's an advanced topic for a later course.
5. **TUI/Web UI** — the event system is built for this, but rendering is out of scope for the MVP.

## Design Decisions Worth Highlighting in the Course

- Why `convertToLlm` exists: it decouples the loop from provider-specific message formats
- Why tools use Zod schemas: runtime validation + TypeScript types from a single source of truth
- Why events are async-await (not fire-and-forget): ordering guarantees matter for correctness
- Why state is mutable internally but readonly externally: performance vs. safety trade-off
