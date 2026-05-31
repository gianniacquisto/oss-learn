# Steps for Milestone 3: Core Agent Loop

---

## Step 3.1: Define the event types and config interface

**Goal:** Create the type definitions that describe what events the agent emits and how the loop is configured.

**Requirements:**
Add these to `src/types.ts`:

- **`AgentEvent`** — a discriminated union of all lifecycle events:
  - `{ type: "agent_start" }` — emitted when a run begins
  - `{ type: "agent_end"; messages: AgentMessage[] }` — emitted when a run completes (last event)
  - `{ type: "turn_start" }` — emitted at the start of each LLM call cycle
  - `{ type: "turn_end"; message: AssistantMessage; toolResults: ToolResultMessage[] }` — emitted after each turn
  - `{ type: "message_start"; message: AgentMessage }` — emitted when a new message begins streaming
  - `{ type: "message_update"; message: AgentMessage }` — emitted during streaming (for assistant messages)
  - `{ type: "message_end"; message: AgentMessage }` — emitted when a message is complete

- **`AgentLoopConfig`** — the configuration object passed to the loop function. Include these fields:
  - `model: string` — which model to use
  - `apiKey: string` — API key for the provider
  - `systemPrompt: string` — system prompt for this run
  - `convertToLlm(messages: AgentMessage[]): Message[]` — function that converts internal messages to LLM-provider format (you'll implement this per-provider later)
  - `shouldStopAfterTurn?(message: AssistantMessage, toolResults: ToolResultMessage[]): boolean` — optional hook to decide if the loop should exit after a turn

**Hints:**
- The event types follow pi.dev's pattern closely. You can use the same structure but simplify where needed (e.g., no streaming events for this milestone since you're doing non-streaming first).
- `convertToLlm` is a critical abstraction — it lets the loop stay provider-agnostic. The loop doesn't care *which* provider you use; it just needs to know how to convert messages into that provider's format.

**Reference pattern:**
```typescript
// types.ts additions — structural reference only

export type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AssistantMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage }
  | { type: "message_end"; message: AgentMessage };

export interface AgentLoopConfig {
  model: string;
  apiKey: string;
  systemPrompt: string;
  convertToLlm(messages: AgentMessage[]): Message[];
  shouldStopAfterTurn?: (context: {
    message: AssistantMessage;
    toolResults: ToolResultMessage[];
  }) => boolean;
}

// A simple Message type for LLM providers
export interface Message {
  role: "system" | "user" | "assistant" | "toolResult";
  content: string | (TextContent | ImageContent)[];
}
```

---

## Step 3.2: Implement the core loop function

**Goal:** Create `runLoop` in `src/loop.ts` — the main control flow that drives the agent.

**Requirements:**
Implement a `runLoop` function with this signature:

```typescript
export async function runLoop(
  context: AgentContext,      // { systemPrompt, messages, tools }
  config: AgentLoopConfig,
  emit: (event: AgentEvent) => Promise<void>,
  signal?: AbortSignal,
): Promise<AssistantMessage>
```

The loop should:
1. Emit `agent_start` and `turn_start` events
2. Convert the context messages to LLM format using `config.convertToLlm`
3. Build a request payload with system prompt + converted messages
4. Call the LLM (you'll implement this in Milestone 5 — for now, use a mock/stub that returns a fake response)
5. Create an `AssistantMessage` from the response and add it to context.messages
6. Emit `message_start`, `message_end`, and `turn_end` events
7. Check `config.shouldStopAfterTurn()` — if true, emit `agent_end` and return
8. If not stopping, loop back to step 2 (but only once for this milestone — you'll add the real multi-turn logic in Milestone 4)

**Hints:**
- For now, use a mock LLM call that returns a fake assistant message with `stopReason: "stop"` and some text content. This lets you test the loop structure without needing API keys.
- The `emit` function is passed in as a parameter (dependency injection). This makes the loop testable and decoupled from any specific event consumer.
- Track the accumulated messages in a local array (`newMessages`) that gets returned with `agent_end`.

**Reference pattern:**
```typescript
// loop.ts — structural reference only

export interface AgentContext {
  systemPrompt: string;
  messages: AgentMessage[];
  tools?: Tool[];
}

export async function runLoop(
  context: AgentContext,
  config: AgentLoopConfig,
  emit: (event: AgentEvent) => Promise<void>,
  signal?: AbortSignal,
): Promise<AssistantMessage> {
  await emit({ type: "agent_start" });
  await emit({ type: "turn_start" });

  // Convert messages to LLM format
  const llmMessages = config.convertToLlm(context.messages);

  // Build request with system prompt + messages
  // Call the mock LLM (replace with real call in Milestone 5)
  const response = await mockLLMCall({
    model: config.model,
    apiKey: config.apiKey,
    systemPrompt: config.systemPrompt,
    messages: llmMessages,
  });

  // Create assistant message from response
  const assistantMessage: AssistantMessage = {
    role: "assistant",
    content: [{ type: "text", text: response.text }],
    model: config.model,
    provider: "mock",
    stopReason: "stop",
    timestamp: Date.now(),
  };

  context.messages.push(assistantMessage);
  await emit({ type: "message_start", message: assistantMessage });
  await emit({ type: "message_end", message: assistantMessage });
  await emit({ type: "turn_end", message: assistantMessage, toolResults: [] });

  // Check if we should stop
  const shouldStop = config.shouldStopAfterTurn?.({
    message: assistantMessage,
    toolResults: [],
  });

  if (shouldStop) {
    await emit({ type: "agent_end", messages: context.messages });
    return assistantMessage;
  }

  // For now, just return — multi-turn logic comes in Milestone 4
  await emit({ type: "agent_end", messages: context.messages });
  return assistantMessage;
}
```

---

## Step 3.3: Add the `prompt` method to the Agent class

**Goal:** Give the Agent a public `prompt()` method that wraps `runLoop`.

**Requirements:**
Add to your `Agent` class in `src/agent.ts`:

- A `prompt(input: string): Promise<void>` method that:
  1. Checks if `isStreaming` is true — if so, throw an error ("Agent is already processing")
  2. Creates a `UserMessage` from the input string with a timestamp
  3. Adds it to `this._state.messages`
  4. Sets `this._state.isStreaming = true`
  5. Calls `runLoop()` with the current context snapshot and config
  6. On completion, sets `isStreaming = false`

**Hints:**
- The "already processing" check is important — it prevents concurrent prompts from corrupting state. This is a common pattern in async systems: guard against reentrancy.
- You need to create an `AbortController` and pass its signal to the loop so callers can cancel mid-execution via `agent.abort()`.

**Reference pattern:**
```typescript
// agent.ts — add to Agent class (structural reference only)

async prompt(input: string): Promise<void> {
  if (this._state.isStreaming) {
    throw new Error("Agent is already processing a prompt.");
  }

  const message: UserMessage = {
    role: "user",
    content: [{ type: "text", text: input }],
    timestamp: Date.now(),
  };

  this._state.messages.push(message);
  this._state.isStreaming = true;

  try {
    const abortController = new AbortController();
    // Store for later access via agent.signal / agent.abort()

    await runLoop(
      { systemPrompt: this._state.systemPrompt, messages: [...this._state.messages] },
      this.createConfig(),  // helper to build AgentLoopConfig from state
      async (event) => this.emitEvent(event),  // helper to emit events
      abortController.signal,
    );
  } finally {
    this._state.isStreaming = false;
  }
}
```

---

## Step 3.4: Wire the CLI to test the loop with a mock response

**Goal:** Update `cli.ts` to create an agent, call `prompt()`, and print events as they're emitted.

**Requirements:**
- In your CLI, after creating the Agent:
  - Subscribe to events (use `agent.subscribe()` or a simple callback)
  - Call `await agent.prompt("Hello, agent!")`
  - Print each event type as it's received
  - After completion, print the final assistant message content

**Hints:**
- For event subscription, you can start with a simple approach: pass an emit function directly to the loop rather than implementing a full subscribe/unsubscribe system. You'll add that in Milestone 5 when streaming events become important.
- The mock response should include some interesting text (e.g., "Hello! I'm your agent. How can I help you today?") so you can verify the flow end-to-end.

**Reference pattern:**
```typescript
// cli.ts update — structural reference only

import { Agent } from "./agent.js";

async function main() {
  const config = loadConfig();
  const agent = new Agent({ systemPrompt: config.systemPrompt, model: config.model });

  // Simple event logger
  agent.subscribe((event) => {
    console.log(`[EVENT] ${event.type}`, event);
  });

  await agent.prompt("Hello, agent!");
}
```

---

## Final Check for This Milestone

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
