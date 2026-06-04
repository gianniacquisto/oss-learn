---
description: Terms and concepts you'll encounter throughout this course.
---

# :material-book-alphabet: Glossary

Terms and concepts you'll encounter throughout this course. Click any term to expand for details and references.

---

## A

### Agent Loop

??? detail "Agent Loop"
    **Definition:** The core control flow of an AI agent: prompt the model → receive a response → execute any tool calls → feed results back to the model → repeat until completion.

    **Why it matters here:** This is the central pattern you'll build in Milestone 3. Every agentic system, from simple chatbots to complex autonomous agents, implements some variation of this loop. Understanding it means understanding how agents actually work at runtime.

    [:fontawesome-solid-link: Lilian Weng's AI Agent Survey](https://lilianweng.github.io/posts/2023-06-23-agent/) — the definitive overview

### AbortSignal

??? detail "AbortSignal"
    **Definition:** A mechanism to cancel async operations. When an `AbortController` is aborted, its associated `AbortSignal` notifies all listeners so they can clean up gracefully.

    **Why it matters here:** Agents run for potentially long periods (executing many tool calls). If the user wants to stop mid-execution, you need a way to propagate that cancellation through async chains without leaving resources dangling. Pi uses `AbortController`/`AbortSignal` for this throughout the agent loop.

    [:fontawesome-solid-link: MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)

---

## E

### Event Stream

??? detail "Event Stream"
    **Definition:** A pattern where a producer pushes events to consumers, and consumers react independently. In TypeScript, this is often implemented as an async iterable or a callback-based system with subscription/unsubscription.

    **Why it matters here:** Your agent emits lifecycle events (`agent_start`, `turn_end`, etc.) that UIs subscribe to. The event stream is the contract between your core engine and whatever renders output — whether that's a CLI, TUI, or web interface.

    [:fontawesome-solid-link: EventEmitter pattern](https://nodejs.org/api/events.html) — Node.js's built-in implementation

---

## M

### Message (in LLM context)

??? detail "Message"
    **Definition:** A single unit of conversation: a role (`user`, `assistant`, `system`, `toolResult`) plus content (text, images, or tool calls). Messages are sent to the model as an array representing the conversation history.

    **Why it matters here:** Your agent manages messages internally as `AgentMessage[]` — a union of LLM-compatible messages and custom application-specific message types. The separation between "what the model sees" and "what the app knows" is a key design decision in pi.

    [:fontawesome-solid-link: OpenAI Messages API](https://platform.openai.com/docs/guides/text-generation/chat-completions-api)

### MCP (Model Context Protocol)

??? detail "MCP"
    **Definition:** A protocol that standardizes how AI agents interact with external tools and data sources. Instead of each agent having custom integrations, MCP provides a shared interface for tools to register themselves and be discovered by any compliant agent.

    **Why it matters here:** Pi uses MCP as its tool integration layer. When you build the tool system in Milestone 4, think about how your tool registration could map to an MCP-compatible format — that's what makes pi's architecture extensible beyond its own codebase.

    [:fontawesome-solid-link: Model Context Protocol](https://www.modelcontextprotocol.io/)

---

## S

### Streaming Response

??? detail "Streaming Response"
    **Definition:** An LLM response delivered incrementally (token by token) rather than all at once. The consumer receives partial data as it becomes available, enabling real-time display and faster perceived latency.

    **Why it matters here:** In Milestone 5 you'll implement streaming. The key challenge: a streamed message is *mutable* — each update modifies the same message object. You need to track which message is "current" during streaming and know when updates stop.

    [:fontawesome-solid-link: OpenAI Streaming Guide](https://platform.openai.com/docs/guides/text-generation/streaming)

---

## T

### Tool Call (function calling)

??? detail "Tool Call"
    **Definition:** When an LLM response includes a request to execute a specific function with arguments, rather than just returning text. The agent must parse the call, validate arguments, execute the tool, and feed the result back to the model.

    **Why it matters here:** This is what makes agents *agentic* — they don't just generate text, they take action in the world. Your Milestone 4 implementation will handle the full lifecycle: registration → detection → validation → execution → result formatting.

    [:fontawesome-solid-link: OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)

### TypeScript Generics

??? detail "TypeScript Generics"
    **Definition:** A way to write reusable code that works with multiple types while preserving type safety. In your agent harness, generics let you define tool schemas (`AgentTool<TParameters>`) where each tool has its own parameter and result types.

    **Why it matters here:** Without generics, your tool system would lose type information at registration time — you'd get `any` everywhere and lose the safety that makes TypeScript valuable. Pi uses generics extensively to maintain end-to-end type safety from schema definition through execution.

    [:fontawesome-solid-link: TypeScript Generics Deep Dive](https://www.typescriptlang.org/docs/handbook/2/generics.html)
