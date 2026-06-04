# Steps — The Agent Loop

[:material-arrow-left: Back to Milestone Overview](README.md){: .md-button }

---

## Step 3.1: Define event types and the event callback

**Goal:** Add event type definitions to `src/types.ts` that describe every lifecycle event your agent emits.

### Why This Step Exists

Events are how your agent communicates "what's happening" to other parts of the program. The CLI listens to events to print status messages. A web UI (in a real product) would listen to events to update its display. A logger would write them to a file.

By defining event types upfront, you get compile-time guarantees that every event has the right shape and every listener handles all possible event types. The discriminated union pattern (same as messages) lets TypeScript narrow event types based on the `type` field.

### The Event Lifecycle

When you call `agent.prompt("Hello")`, these events fire in order:

```
prompt() called
  │
  ├─ agent_start          ← "The agent began processing"
  │   │
  │   ├─ turn_start       ← "Starting a new LLM call cycle"
  │   │   │
  │   │   ├─ message_start    ← "LLM is generating a response"
  │   │   ├─ message_update   ← "New text arrived" (can fire multiple times)
  │   │   └─ message_end      ← "Response generation complete"
  │   │
  │   └─ turn_end         ← "LLM call cycle finished"
  │
  └─ agent_end            ← "The agent is done processing"
```

For this milestone (single-turn, no tools), you'll see: `agent_start → turn_start → message_start → message_update → message_end → turn_end → agent_end`.

When you add tool execution in Milestone 4, additional turns will appear between `turn_end` and `agent_end`.

### Requirements

Add these types to `src/types.ts`:

```typescript
// Lifecycle events — each has a unique 'type' for discriminated union narrowing
export type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AssistantMessage }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage }
  | { type: "message_end"; message: AgentMessage };

// Callback type — any function that receives events
export type OnEvent = (event: AgentEvent) => void;
```

**Why each event carries specific data:**

| Event | Data carried | Why it matters |
|-------|-------------|---------------|
| `agent_start` | Nothing | Signals "processing began" — UI can show a loading indicator |
| `agent_end` | Final messages | Gives listeners the complete conversation transcript |
| `turn_start` | Nothing | Signals "LLM is being called" — useful for timing/logging |
| `turn_end` | The assistant message | Gives listeners the response from this LLM call |
| `message_start` | Message being built | Signals "text generation began" |
| `message_update` | Partial or full message | Streaming text — UI can display tokens as they arrive |
| `message_end` | Complete message | Signals "this message is finalized" |

**Why `OnEvent` returns `void`?** The callback processes events (prints them, logs them, updates UI) but doesn't return data to the loop. If you need to communicate back from a listener, use a different mechanism (shared state, additional callbacks).

### ??? tip "Hints"

- The event types mirror pi.dev's actual event system — simplified for this course
- `OnEvent` is just a function signature type — it says "a function that takes an `AgentEvent` and returns nothing"
- You could make `OnEvent` async (`Promise<void>`) if listeners need to await operations, but keeping it sync is simpler

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `AgentEvent` union type exists with all seven event shapes
- [ ] `OnEvent` callback type exists
- [ ] Running `npm run build` compiles without errors

---

## Step 3.2: Implement the mock LLM call

**Goal:** Create a function in `src/loop.ts` that simulates an LLM response — returning a predictable `AssistantMessage` without calling any real API.

### Why This Step Exists

Before you wire up real OpenAI calls (Milestone 5), you need to verify your loop logic works correctly. A mock LLM:
- Returns a known response so you can assert expected behavior
- Has zero latency — no waiting for network calls
- Costs nothing — no API credits burned during development
- Lets you test the *structure* of the loop without the *content* of real responses

When you're ready for production, you replace `mockLLMCall()` with a real API call. The loop code doesn't change — only the function it calls.

### Requirements

Add this function to `src/loop.ts`:

```typescript
import type { AgentMessage, AssistantMessage } from "./types.js";

/**
 * Simulate an LLM response for testing.
 * Returns a predictable AssistantMessage based on the last user message.
 */
async function mockLLMCall(messages: AgentMessage[]): Promise<AssistantMessage> {
  // Find the most recent user message
  const lastUserMessage = [...messages].reverse().find((m) => m.role === "user");

  // Extract text from the user's message content blocks
  const userText = lastUserMessage
    ? lastUserMessage.content.map((c) => c.text).join("")
    : "nothing";

  // Generate a fake response
  const responseText = `I received your message: "${userText}". This is a mock response.`;

  return {
    role: "assistant",
    content: [{ type: "text", text: responseText }],
    model: "mock",
    provider: "mock",
    stopReason: "stop",   // no tool calls — the mock always gives a final answer
    timestamp: Date.now(),
  };
}
```

**Why `async` if there's no actual async work?** The function signature matches what a real LLM call would have (`Promise<AssistantMessage>`). This lets you swap `mockLLMCall()` for a real API call without changing any calling code — the `await mockLLMCall(messages)` stays the same.

**Why `[...messages].reverse().find(...)`?** Spreading creates a copy so `reverse()` doesn't mutate the original array. We reverse because the *last* user message is the most recent one, and `find()` returns the first match from the start of the array.

### ??? tip "Hints"

- The mock doesn't need to be smart — it just needs to return a valid `AssistantMessage` with all required fields
- `stopReason: "stop"` means "the model finished naturally" (no tool calls needed). In Milestone 4 you'll add `stopReason: "toolCalls"` for tool-using responses
- You can make the mock more interesting by returning different responses based on the input

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `mockLLMCall` exists in `src/loop.ts` and is `async`
- [ ] It returns an `AssistantMessage` with all required fields populated
- [ ] Running `npm run build` compiles without errors

---

## Step 3.3: Implement the core loop function

**Goal:** Create `runLoop()` in `src/loop.ts` — the main control flow that calls the LLM, emits events, and accumulates messages.

### Why This Step Exists

This is the heart of your agent. The loop function coordinates everything: it prepares the conversation, calls the LLM (mock for now), processes the response, emits events as things happen, and decides when to stop.

The key insight: **the loop doesn't care where the LLM response comes from**. It calls a function, gets an `AssistantMessage`, and processes it. Whether that function is a mock or a real API call is irrelevant to the loop logic. This separation of concerns is what makes the system maintainable.

### Requirements

Add this function to `src/loop.ts`:

```typescript
import type { AgentConfig } from "./config.js";
import type { AgentMessage, AssistantMessage, OnEvent, Tool } from "./types.js";

export async function runLoop(
  config: {
    model: string;
    apiKey: string;
    systemPrompt: string;
    tools: Tool[];
  },
  messages: AgentMessage[],
  onEvent: OnEvent,
  shouldStopAfterTurn?: (message: AssistantMessage) => boolean,
): Promise<AssistantMessage> {
  // Signal that processing has begun
  onEvent({ type: "agent_start" });

  let lastMessage: AssistantMessage | null = null;

  while (true) {
    // Start a new turn (one LLM call cycle)
    onEvent({ type: "turn_start" });

    // Call the LLM (mock for now — replaced with real API in Milestone 5)
    const assistantMessage = await mockLLMCall(messages);

    // Add the assistant's response to the conversation
    messages.push(assistantMessage);
    lastMessage = assistantMessage;

    // Emit message lifecycle events
    onEvent({ type: "message_start", message: assistantMessage });
    onEvent({ type: "message_update", message: assistantMessage });
    onEvent({ type: "message_end", message: assistantMessage });
    onEvent({ type: "turn_end", message: assistantMessage });

    // Check if we should stop (default: stop after first turn with mock)
    if (shouldStopAfterTurn?.(assistantMessage) ?? true) {
      break;
    }

    // In later milestones, tool execution happens here:
    // - detect tool calls in the assistant message
    // - execute each tool
    // - add results to messages
    // - loop back to call LLM again with new context
  }

  // Signal that processing is complete
  onEvent({ type: "agent_end", messages });

  return lastMessage!;
}
```

**Why the `config` object groups model, apiKey, systemPrompt, and tools?** It's a single parameter that carries all the settings the loop might need. In Milestone 5, you'll add the actual API key to this config and use it for real calls. Grouping keeps the function signature clean.

**Why `shouldStopAfterTurn` is optional?** For this milestone, the default is `true` (stop after one turn — enough to test with the mock). In Milestone 4, you'll provide a custom function that checks `message.stopReason` — if it's `"toolCalls"`, keep looping; if it's `"stop"`, exit.

**Why `messages.push(assistantMessage)` modifies the array directly?** The `messages` array is passed by reference from the Agent class. When the loop pushes new messages, they're added to the Agent's internal `_state.messages`. This is intentional — the loop *needs* to grow the conversation.

### ??? tip "Hints"

- The `while(true)` loop seems dangerous (infinite loop!), but the `break` inside `shouldStopAfterTurn` guarantees it exits
- Each `onEvent()` call broadcasts to all listeners — the CLI is one listener that prints events
- In Milestone 5, `message_update` will fire multiple times (once per token) for streaming responses

### ??? warning "Common Mistake"

**Forgetting to push the assistant message before checking `shouldStopAfterTurn`:** The stop condition might need to inspect the message content. If you push after the check, the message isn't in the conversation yet when the stop function runs. Always push first, then check.

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `runLoop` exists in `src/loop.ts` and is exported
- [ ] It calls `mockLLMCall`, pushes the result to messages, and emits all events
- [ ] It breaks out of the loop when `shouldStopAfterTurn` returns true (or defaults to true)
- [ ] Running `npm run build` compiles without errors

---

## Step 3.4: Add the `prompt()` method to the Agent class

**Goal:** Give the Agent a public `prompt(input, onEvent?)` method that creates a user message and starts the loop.

### Why This Step Exists

The `prompt()` method is the **public API** of your agent — it's what users (or other code) call to interact with it. It handles three responsibilities:

1. **Input validation:** Rejects calls if the agent is already processing (prevents race conditions)
2. **Message creation:** Converts a plain string into a properly typed `UserMessage`
3. **Lifecycle management:** Sets `isStreaming = true` before the loop and `false` after (even on error)

This method is the bridge between the outside world and your agent's internal state.

### Requirements

Add this method to the `Agent` class in `src/agent.ts`:

```typescript
import { runLoop } from "./loop.js";
import type { OnEvent, UserMessage, AssistantMessage } from "./types.js";

// ... inside the Agent class ...

async prompt(input: string, onEvent?: OnEvent): Promise<void> {
  // Guard: don't allow concurrent prompts (would corrupt state)
  if (this._state.isStreaming) {
    throw new Error("Agent is already processing a prompt.");
  }

  // Convert the input string into a UserMessage
  const userMessage: UserMessage = {
    role: "user",
    content: [{ type: "text", text: input }],
    timestamp: Date.now(),
  };

  // Add it to the conversation history
  this._state.messages.push(userMessage);

  // Mark the agent as busy
  this._state.isStreaming = true;

  try {
    // Start the loop — it will call the LLM, emit events, accumulate messages
    await runLoop(
      {
        model: this._state.model,
        apiKey: "",  // will be populated from config in Milestone 5
        systemPrompt: this._state.systemPrompt,
        tools: this._state.tools,
      },
      this._state.messages,
      onEvent ?? (() => {}),  // no-op callback if no listener provided
    );
  } finally {
    // Always reset isStreaming, even if an error occurred
    this._state.isStreaming = false;
    // Clear any error state from previous runs
    this._state.errorMessage = undefined;
  }
}
```

**Why the concurrency guard (`isStreaming` check)?** If two `prompt()` calls run simultaneously, they'd both push user messages and both read/modify the same messages array — creating a corrupted conversation. The guard ensures only one prompt processes at a time.

**Why `try/finally` instead of `try/catch`?** The `finally` block runs regardless of whether the loop succeeds or throws an error. This guarantees `isStreaming` is always reset to `false`. If you used `try/catch` and forgot to handle some error path, the agent would be permanently locked in "streaming" state.

**Why `onEvent ?? (() => {})`?** The callback is optional — callers might not care about events. The nullish coalescing provides a no-op function as the default, so the loop always has something to call without checking for `undefined`.

**Why `apiKey: ""` for now?** The API key isn't needed for the mock LLM. In Milestone 5, you'll wire it through from the config loaded in Milestone 1.

### ??? tip "Hints"

- Import `runLoop` from `./loop.js` (with `.js` extension for NodeNext modules)
- The `prompt` method is `async` because it `await`s the loop, which awaits the LLM call
- Don't forget to add the `AssistantMessage` import if you need it for type annotations

### ??? warning "Common Mistake"

**Setting `isStreaming = false` inside `try` instead of `finally`:** If an error is thrown between setting `true` and `false`, the agent stays in a permanent streaming state. The `finally` block guarantees cleanup:
```typescript
// ❌ Bad — if runLoop throws, isStreaming stays true
try {
  this._state.isStreaming = true;
  await runLoop(...);
  this._state.isStreaming = false;  // never reached if error above
} catch (error) { ... }

// ✅ Good — finally always runs
this._state.isStreaming = true;
try {
  await runLoop(...);
} finally {
  this._state.isStreaming = false;  // always reached
}
```

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `prompt()` method exists on the Agent class
- [ ] It throws if called while `isStreaming` is true
- [ ] It creates a `UserMessage` and adds it to the messages array
- [ ] It sets `isStreaming` to true before and false after (in finally)
- [ ] Running `npm run build` compiles without errors

---

## Step 3.5: Wire the CLI to test the loop end-to-end

**Goal:** Update `src/cli.ts` to create an agent, call `prompt()` with an event logger, and verify the full flow works.

### Why This Step Exists

You've built the loop in isolation. Now you need to verify it works when wired through the full stack: CLI → Agent → Loop → Mock LLM → Events → CLI output. This is your first end-to-end test of the agent architecture.

The event logger in the CLI is a concrete example of an "event listener" — it receives events from the loop and converts them to human-readable text. In a real application, this might be a web UI renderer, a log writer, or an analytics tracker.

### Requirements

Replace your `cli.ts` with this updated version:

```typescript
import { loadConfig } from "./config.js";
import { Agent } from "./agent.js";
import type { AgentEvent } from "./types.js";

async function main() {
  try {
    const config = loadConfig();

    const agent = new Agent({
      systemPrompt: config.systemPrompt,
      model: config.model,
    });

    // Event logger — prints each event as it arrives
    const onEvent = (event: AgentEvent) => {
      switch (event.type) {
        case "agent_start":
          console.log("[Agent started]");
          break;
        case "turn_start":
          console.log("  [Turn started]");
          break;
        case "message_start":
          console.log("    [Message generation started]");
          break;
        case "message_update":
          const text = event.message.content
            .map((c) => (c.type === "text" ? c.text : ""))
            .join("");
          console.log(`    ${text}`);
          break;
        case "message_end":
          console.log("    [Message generation ended]");
          break;
        case "turn_end":
          console.log("  [Turn ended]");
          break;
        case "agent_end":
          console.log("[Agent finished — total messages:", event.messages.length, "]");
          break;
      }
    };

    // Send a prompt and observe the events
    await agent.prompt("Hello, agent!", onEvent);

    // Verify state after completion
    console.log("\nFinal state:");
    console.log("  isStreaming:", agent.state.isStreaming);
    console.log("  messages:", agent.state.messages.length);
    process.exit(0);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    console.error(`Error: ${message}`);
    process.exit(1);
  }
}

main();
```

**Why the `switch` on `event.type`?** TypeScript's discriminated union narrowing works with switch statements. Inside each `case`, TypeScript knows the exact event shape and shows you the available fields. For example, in `case "agent_end"`, `event.messages` is typed as `AgentMessage[]`.

**Why the indentation in console output?** The nested brackets (`[ ]`, `  [ ]`, `    [ ]`) visually show the hierarchy of events — turns contain messages, agent_start contains turns. This makes it easier to follow the flow when debugging.

### ??? example "Expected output"

```
[Agent started]
  [Turn started]
    [Message generation started]
    I received your message: "Hello, agent!". This is a mock response.
    [Message generation ended]
  [Turn ended]
[Agent finished — total messages: 2 ]

Final state:
  isStreaming: false
  messages: 2
```

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `npm run build` compiles without errors
- [ ] `AGENT_API_KEY=fake npm start` prints the event sequence shown above
- [ ] The final state shows `isStreaming: false` and `messages: 2` (user + assistant)
- [ ] Events appear in the correct order (agent_start → turn_start → message events → turn_end → agent_end)

---

## All Done? Check Your Work [:material-arrow-right:](done.md)

Before moving to Milestone 4, follow the verification steps in [done.md](./done.md).
