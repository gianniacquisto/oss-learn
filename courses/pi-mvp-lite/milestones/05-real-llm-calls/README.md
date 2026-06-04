---
comments: Milestone 5 of 5
---

# ⑤ Real LLM Calls

## What You'll Build

Replace the mock LLM with real OpenAI API calls, including **streaming responses** — text that appears token by token as the model generates it. By the end you will have:

- **A message converter** — transforms your internal `AgentMessage[]` into OpenAI's `ChatCompletionMessageParam[]` format
- **A streaming function** — connects to OpenAI's API and yields events as tokens arrive
- **An updated loop** — consumes the stream incrementally, emitting `message_update` events for each token
- **A streaming CLI** — shows text appearing in real time instead of all at once

This is the moment your agent comes alive. Instead of hardcoded mock responses, it has a real conversation with GPT — remembering context, using tools, and generating human-like text.

## Why This Matters

Everything you've built so far (message types, agent state, loop logic, tool execution) was infrastructure. This milestone connects that infrastructure to the actual AI brain. But it's not just "call an API" — you have to handle:

- **Message format conversion:** Your internal `AgentMessage` types don't match OpenAI's expected format. You need a converter function that translates between the two.
- **Streaming:** Responses arrive token by token, not all at once. Your event system from Milestone 3 was designed for this — each token fires a `message_update` event.
- **Error handling:** Network failures, rate limits, invalid API keys — these must be caught and surfaced as agent errors, not crashes.
- **Tool definitions for the API:** OpenAI needs to know what tools are available (name, description, parameter schema) so it can decide when to call them.

### Key Concepts Introduced Here

??? example "Streaming vs. Non-Streaming"
    **Non-streaming:** Send request → wait 5 seconds → get complete response all at once. Simple but bad UX — user stares at a blank screen.
    **Streaming:** Send request → tokens arrive over time → each token displayed immediately. Better UX — user sees text appearing as it's generated, like typing.

    Streaming also lets you emit `message_update` events per token (which your event system from Milestone 3 supports), enabling real-time UI updates.

??? example "Message Format Conversion"
    Your internal types (`UserMessage`, `AssistantMessage`, `ToolResultMessage`) are designed for *your* code. OpenAI expects a different format with its own field names and structure. The converter function is a **adapter** — it translates between your domain model and the API's requirements. This keeps your internal types decoupled from any specific provider.

??? example "Async Generators"
    The streaming function uses TypeScript's `async generator` syntax (`async function*`) to yield events one at a time as tokens arrive. The caller iterates with `for await (const event of stream)`, processing each token without waiting for the full response. This is the same pattern Node.js uses for reading files and HTTP responses incrementally.

## Prerequisites for This Milestone

- **Milestone 4 complete** — you have a working agent loop with tool execution
- **An OpenAI API key** — get one at [platform.openai.com](https://platform.openai.com/api-keys) (you need credits/balance)
- **Understanding of async iteration** — the `for await...of` pattern for consuming streams

---

## Steps

| Step | Task | What You'll Learn |
|------|------|-------------------|
| 5.1 | Implement OpenAI message conversion | Adapter pattern, mapping between domain and API types |
| 5.2 | Implement streaming with the OpenAI SDK | Async generators, token-by-token response handling |
| 5.3 | Update the loop to use streaming | Replacing mock with real calls, consuming async iterators |
| 5.4 | Wire the CLI for streaming display | Real-time text rendering with `process.stdout.write()` |
| 5.5 | Test with a real conversation | End-to-end verification, debugging real API interactions |

---

[:material-arrow-left: Back to Course](../../README.md){: .md-button }&nbsp;&nbsp;[:material-book-open-variant: Read Full Context](context.md){: .md-button }&nbsp;&nbsp;[:material-flag: Done Checklist](done.md){: .md-button }&nbsp;&nbsp;[:material-arrow-right: Start Building →](steps.md){: .md-button .md-button--primary }
