---
comments: Milestone 2 of 5
---

# ② Messages & Agent State

## What You'll Build

A complete message system and an `Agent` class that manages conversation context, tools, and lifecycle state. By the end you will have:

- **Message type definitions** — TypeScript interfaces for user messages, assistant responses, tool results, and system prompts
- **A union type (`AgentMessage`)** — a single type that represents any message in the conversation
- **A `Tool` interface** — defining what every tool must implement (name, description, execute function)
- **An `Agent` class** — with private mutable state, a readonly public getter, `reset()`, and tool registration

When you run `npm start`, it will print your agent's current state as JSON so you can inspect it.

## Why This Matters

The `Agent` class is the heart of your entire harness. It owns the conversation transcript, knows which tools are available, tracks whether a run is in progress, and surfaces errors to callers. Without this layer, you're just making raw API calls — not building an agent.

Think of it this way: the LLM is the *brain*, but the Agent class is the *body* — it holds memory (messages), can act (tools), and has a lifecycle (running/idle). You're building the body that gives the brain something to work with.

### Key Concepts Introduced Here

??? example "Discriminated Unions (TypeScript)"
    TypeScript can distinguish between types by checking a common field — called a "discriminant." When every message type has a `role` field (`"user"`, `"assistant"`, `"toolResult"`, `"system"`), TypeScript can narrow the union automatically: `if (msg.role === "user")` means `msg` is now typed as `UserMessage` inside that block. This eliminates runtime type-checking bugs.

??? example "Encapsulation (Private State + Readonly Access)"
    The Agent class stores its state in a private `_state` field but exposes it through a readonly `state` getter. Callers can *read* the agent's state but can't accidentally mutate it — only the agent's own methods change its internal data. This prevents bugs where one part of the code silently corrupts the agent's memory.

??? example "Union Types as Message Contracts"
    Instead of one big object with optional fields, each message type is its own interface with exactly the fields it needs. The union type `AgentMessage = UserMessage | AssistantMessage | ToolResultMessage | SystemMessage` guarantees that every message has a `role` and the correct shape for that role. This catches typos and missing fields at compile time.

## Prerequisites for This Milestone

- **Milestone 1 complete** — you have a working TypeScript project with config loading
- **Basic understanding of TypeScript interfaces** — if fuzzy, review the [Reading List](../../reading-list.md) TypeScript links
- **The source files from `src/` are ready** — `types.ts` and `agent.ts` are the main targets

---

## Steps

| Step | Task | What You'll Learn |
|------|------|-------------------|
| 2.1 | Define message types | Discriminated unions, type safety for conversation data |
| 2.2 | Define the Tool interface | How tools declare their shape before any LLM understands them |
| 2.3 | Implement the Agent class | Private state, readonly accessors, encapsulation |
| 2.4 | Wire CLI to create and inspect an Agent | Putting it all together — verifying the Agent works end-to-end |

---

[:material-arrow-left: Back to Course](../../README.md){: .md-button }&nbsp;&nbsp;[:material-book-open-variant: Read Full Context](context.md){: .md-button }&nbsp;&nbsp;[:material-flag: Done Checklist](done.md){: .md-button }&nbsp;&nbsp;[:material-arrow-right: Start Building →](steps.md){: .md-button .md-button--primary }
