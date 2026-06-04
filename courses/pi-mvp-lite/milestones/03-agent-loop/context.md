# Context for Milestone 3: The Agent Loop

## Background

The agent loop is the central control flow of any agentic system. It's the reason an agent can *do* something rather than just *say* something. Without a loop, you make one API call and get one response — that's a chatbot. With a loop, the agent can iterate: reason → act → observe → reason again.

### The Loop Pattern Explained

At its simplest:

```text
while (the task isn't done) {
  1. Send all messages to LLM
  2. Get response
  3. If response contains tool calls:
       a. Run each tool
       b. Add results to messages
       c. Go to step 1 (LLM sees the results and decides what to do next)
  4. If response is a final answer:
       a. Return it to the user
       b. Exit the loop
}
```

This is literally a `while(true)` loop with an exit condition. The "magic" of agents is just this loop + an LLM that can decide when it's done.

### Why Events?

Your loop needs to tell other parts of your program what's happening. A CLI wants to print `[Agent thinking...]`. A web UI wants to show a typing indicator. A logger wants to write timestamps to a file.

Instead of hardcoding `console.log()` inside the loop (which couples it to one output format), the loop *emits events*. Any code can subscribe to these events and react however it wants:

```
Loop → emits event → CLI prints text
                → Web UI updates display
                → Logger writes to file
                → Analytics tracks duration
```

This is the **Observer pattern**: the loop doesn't know who's listening. It just broadcasts events, and listeners do whatever they need.

### Why a Mock LLM?

Before connecting to OpenAI's API (Milestone 5), you need to verify your loop logic works correctly. A mock LLM:
- Returns predictable responses (you know exactly what to expect)
- Has zero latency (no network delay, no waiting)
- Costs nothing (no API credits burned)
- Lets you focus on the *structure* of the loop, not the *content* of responses

When you're ready for real calls, you swap `mockLLMCall()` for `realLLMCall()` — the loop code doesn't change. This is the **Dependency Injection** principle: the loop doesn't care *how* it gets a response, only *that* it gets one.

## How This Fits in the Bigger Picture

```text
cli.ts                        agent.ts                loop.ts
┌──────────┐              ┌──────────────┐       ┌────────────────┐
│ main()   │──create──▶  │ Agent class   │       │ runLoop()      │
│          │             │               │       │                │
│ onEvent()│◀──emits────│ prompt()      │────▶  │ mockLLMCall()  │
│ (prints) │    events   │               │       │                │
│          │             │ _state:       │       │ while(true) {  │
│          │             │  messages[]   │       │  call LLM      │
│          │             │  isStreaming  │       │  emit events   │
│          │             │  tools[]      │       │  check done    │
└──────────┘              └──────────────┘       │ }              │
                                                  └────────────────┘
```

The CLI creates an Agent and calls `agent.prompt()`. The Agent manages state and delegates to `runLoop()`. The loop calls the LLM (mock now, real later), emits events, and accumulates messages. Events flow back to the CLI for display.

## Key Decisions You'll Make

- **Where does the loop live?** In `loop.ts` as a standalone function, not inside the Agent class. This keeps the loop testable in isolation and lets you swap implementations later.
- **How should events be delivered?** A simple callback function (`OnEvent`). The full course uses Node.js `EventEmitter` for more advanced patterns, but a callback is simpler and sufficient here.
- **What's the exit condition?** For this milestone, the loop exits after one turn (the mock response always says it's done). Milestone 4 adds multi-turn logic with tool calls.

## What "Good" Looks Like

By the end of this milestone:
- ✅ `runLoop()` in `loop.ts` calls a mock LLM and emits events in the correct order
- ✅ `Agent.prompt()` starts the loop, manages `isStreaming`, and cleans up on completion
- ✅ Events flow from loop → Agent → CLI (printed as `[Agent started]`, `[Turn started]`, etc.)
- ✅ The agent's message list grows: user message + assistant response after one prompt
- ✅ Calling `prompt()` while already processing throws an error (concurrency guard)
