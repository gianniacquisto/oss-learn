---
comments: Milestone 3 of 5
---

# ③ The Agent Loop

## What You'll Build

The core control flow that makes your agent *work*: a loop that prompts the LLM, processes its response, and repeats until the task is complete. By the end you will have:

- **Event type definitions** — a set of lifecycle events (`agent_start`, `turn_start`, `message_update`, etc.) that other code can listen to
- **A mock LLM function** — a fake response generator so you can test the loop without API keys
- **The `runLoop()` function** — the main loop in `src/loop.ts` that coordinates LLM calls, message accumulation, and event emission
- **A `prompt()` method on Agent** — the public API that users call to send messages to the agent
- **A working CLI** — type `npm start`, send a message, see events flow through, get a response

For now the loop uses a *mock* LLM (a hardcoded response). In Milestone 5 you'll replace the mock with real OpenAI API calls. The loop structure stays the same.

## Why This Matters

This is the single most important concept in agent architecture. Every AI agent — from simple chatbots to complex autonomous systems — implements some variation of this loop:

```
1. Send conversation to LLM
2. LLM responds (with text, or a tool call request)
3. If the LLM called a tool → run it, get results, go to step 1
4. If the LLM gave a final answer → return it to the user
```

Understanding this loop means understanding how agents actually work at runtime. It's not magic — it's a `while` loop with event emission.

### Key Concepts Introduced Here

??? example "The Two-Layer Loop"
    The agent loop has two nested cycles:
    - **Outer loop:** multi-turn conversation (user sends a message → agent responds → user sends another)
    - **Inner loop:** tool calling within a single response (LLM calls a tool → gets result → LLM decides what to do next → may call another tool → finally gives an answer)

    This milestone builds the outer loop. Milestone 4 adds the inner loop (tool execution).

??? example "Event-Driven Architecture"
    Instead of the loop returning one final result, it *emits events* as things happen. A CLI can print events as text, a web UI can render them as rich components — all without changing the loop code. This is the Observer pattern: the loop doesn't know who's listening, it just broadcasts events.

??? example "Mock-First Development"
    Building with a fake LLM before connecting to a real API is a deliberate strategy. It lets you test the structure (loop logic, event flow, state management) without spending API credits or dealing with network latency. When you're ready for real calls, you swap the mock for the actual API — the loop code doesn't change.

## Prerequisites for This Milestone

- **Milestone 2 complete** — you have message types and a working Agent class
- **Understanding of async/await** — the loop uses `async` functions and `await` for LLM calls
- **The files from previous milestones are in place** — `types.ts`, `agent.ts`, `config.ts`, `cli.ts`

---

## Steps

| Step | Task | What You'll Learn |
|------|------|-------------------|
| 3.1 | Define event types | The Observer pattern, discriminated unions for events |
| 3.2 | Implement a mock LLM call | Mock-first development, testing without API dependencies |
| 3.3 | Build the core loop function | The agent loop pattern, event emission, message accumulation |
| 3.4 | Add `prompt()` to the Agent class | Public API design, lifecycle guards (preventing concurrent runs) |
| 3.5 | Wire the CLI end-to-end | Event logging, verifying the full flow from input to output |

---

[:material-arrow-left: Back to Course](../../README.md){: .md-button }&nbsp;&nbsp;[:material-book-open-variant: Read Full Context](context.md){: .md-button }&nbsp;&nbsp;[:material-flag: Done Checklist](done.md){: .md-button }&nbsp;&nbsp;[:material-arrow-right: Start Building →](steps.md){: .md-button .md-button--primary }
