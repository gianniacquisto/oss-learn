# Key Concepts

These are the architectural patterns and design decisions that make an agent harness interesting to study. You'll encounter each of these as you build through the milestones.

## 1. The Agent Loop Pattern

**What it is:** An agent isn't a single function call — it's a loop. The model generates text, which may include tool calls. Those tools execute and produce results. The results go back to the model, which decides what to do next. This repeats until the model signals completion (no more tool calls).

**Why it matters:** This is the fundamental pattern behind every agentic system — from pi.dev to AutoGPT to custom enterprise agents. Understanding this loop is understanding how AI agents actually work at runtime.

### How It Shows Up in This Course

You'll build this in Milestone 3 (core loop) and refine it through Milestone 4 (tool execution). The key insight: the loop has two layers — an outer layer that handles multi-turn follow-ups, and an inner layer that processes tool calls from a single assistant response.

## 2. Stateful Agent vs. Stateless API Calls

**What it is:** A chat completion API call is stateless — you send messages, get a response. An *agent* is stateful — it owns the conversation context, manages tools, tracks what's happening, and makes decisions about when to stop. The `Agent` class is the bridge between these two worlds.

**Why it matters:** Most tutorials teach you to call an LLM API. Fewer teach you how to *manage* the state around that call — which messages are in context, which tools are available, what events have fired, when to abort, etc. This is where real engineering happens.

### How It Shows Up in This Course

Milestone 2 introduces the Agent class with its internal state (messages, tools, streaming status). You'll see how state transitions work — from idle → processing → idle again — and why that matters for correctness.

## 3. Event-Driven Architecture

**What it is:** Instead of returning a single result, the agent emits events as it progresses: `agent_start`, `turn_start`, `message_update`, `tool_execution_end`, etc. Consumers (UIs, loggers, monitors) subscribe to these events and react independently.

**Why it matters:** This decouples the core engine from whatever consumes its output. A CLI can render events as text, a TUI can render them as rich components, a web UI can stream them via WebSocket — all without changing the agent code.

### How It Shows Up in This Course

You'll implement an event system starting in Milestone 3 and use it throughout. The key design question: should events be synchronous (blocking) or async (non-blocking)? Pi uses async-await listeners, which means each listener is awaited before the next event fires — this ensures ordering but can slow things down if a listener does heavy work.

## 4. Tool Registration with Type-Safe Schemas

**What it is:** Tools are registered with a name, description, parameter schema (usually JSON Schema or equivalent), and an execute function. The agent validates arguments against the schema before calling `execute()`. This gives you type safety from the LLM's output all the way through to your tool implementation.

**Why it matters:** Without schema validation, you're passing raw strings from an LLM directly into your code — a recipe for bugs and security issues. With schemas, you get automatic argument parsing, validation, and TypeScript types that flow from registration to execution.

### How It Shows Up in This Course

Milestone 4 is entirely about this pattern. You'll define tools with parameter schemas, register them on the Agent, and see how the loop validates and executes them. The design choice here: do you validate eagerly (before calling execute) or lazily (inside execute)? Pi validates eagerly — it's safer and gives better error messages.

## 5. Streaming vs. Non-Streaming Responses

**What it is:** LLM APIs can return responses all at once (non-streaming) or token-by-token (streaming). Streaming lets you show partial results to the user immediately, but it complicates your code because you're dealing with partial data that may change as more tokens arrive.

**Why it matters:** Streaming is what makes agents feel "alive" — users see text appearing in real time rather than waiting for a complete response. But streaming requires careful state management: you need to accumulate partial messages, handle updates correctly, and know when the stream is truly done.

### How It Shows Up in This Course

Milestone 5 introduces streaming. You'll implement both a simple non-streaming path (for debugging) and a streaming path (for production). The key insight: even with streaming, your agent's *logic* doesn't change — only how it receives the assistant message changes.
