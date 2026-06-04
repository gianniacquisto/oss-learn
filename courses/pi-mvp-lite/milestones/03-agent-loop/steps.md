# Steps for Milestone 3: The Agent Loop

---

## Step 3.1: Define event types and the event callback

**Goal:** Create the type definitions that describe what events the agent emits.

**Requirements:**
Add these to `src/types.ts`:

- **`AgentEvent`** — a union of all lifecycle events:
  - `{ type: "agent_start" }` — emitted when a run begins
  - `{ type: "agent_end"; messages: AgentMessage[] }` — emitted when a run completes
  - `{ type: "turn_start" }` — emitted at the start of each LLM call cycle
  - `{ type: "turn_end"; message: AssistantMessage }` — emitted after each turn
  - `{ type: "message_start"; message: AgentMessage }` — emitted when a new message begins
  - `{ type: "message_update"; message: AgentMessage }` — emitted during text generation
  - `{ type: "message_end"; message: AgentMessage }` — emitted when a message is complete

- **`OnEvent`** — a callback type: `(event: AgentEvent) => void`

**Hints:**
- The event types follow pi.dev's pattern. You can use the same structure but simplified.
- `OnEvent` is a simple callback — any function that takes an `AgentEvent` and returns nothing.

??? example "Reference pattern"
    === "types.ts — add after AgentMessage"
        ```typescript
        // Lifecycle events
        export type AgentEvent =
          | { type: "agent_start" }
          | { type: "agent_end"; messages: AgentMessage[] }
          | { type: "turn_start" }
          | { type: "turn_end"; message: AssistantMessage }
          | { type: "message_start"; message: AgentMessage }
          | { type: "message_update"; message: AgentMessage }
          | { type: "message_end"; message: AgentMessage };

        // Callback for receiving events
        export type OnEvent = (event: AgentEvent) => void;
        ```

---

## Step 3.2: Implement the mock LLM call

**Goal:** Create a function that simulates an LLM response for testing.

**Requirements:**
Add to `src/loop.ts`:

- A `mockLLMCall` function that takes a message and returns a fake assistant response:
  ```typescript
  async function mockLLMCall(messages: AgentMessage[]): Promise<AssistantMessage>
  ```
- The function should:
  1. Look at the last user message
  2. Generate a fake response like `"Hello! I'm your agent. You asked me: ${lastUserMessage}"`
  3. Return an `AssistantMessage` with `stopReason: "stop"` (no tool calls)

**Hints:**
- This is a fake — it doesn't actually call any API. It just returns a predictable response.
- You'll replace this with a real API call in Milestone 4.
- The response should be interesting enough to verify the loop works end-to-end.

??? example "Reference pattern"
    === "loop.ts — add at the top"
        ```typescript
        import type { AgentMessage, AssistantMessage, TextContent } from "./types.js";

        // Mock LLM — replaces real API call during development
        async function mockLLMCall(messages: AgentMessage[]): Promise<AssistantMessage> {
          // Find the last user message
          const lastUserMessage = [...messages].reverse().find(m => m.role === "user");
          const userText = lastUserMessage
            ? lastUserMessage.content.map(c => c.text).join("")
            : "nothing";

          const responseText = `Hello! I'm your agent. You asked me: ${userText}`;

          return {
            role: "assistant",
            content: [{ type: "text", text: responseText }],
            model: "mock",
            provider: "mock",
            stopReason: "stop",
            timestamp: Date.now(),
          };
        }
        ```

---

## Step 3.3: Implement the core loop function

**Goal:** Create `runLoop` in `src/loop.ts` — the main control flow that drives the agent.

**Requirements:**
Implement a `runLoop` function:

```typescript
export async function runLoop(
  config: {
    model: string;
    apiKey: string;
    systemPrompt: string;
    tools: Tool[];
  },
  messages: AgentMessage[],       // current conversation
  onEvent: OnEvent,               // callback for events
  shouldStopAfterTurn?: (message: AssistantMessage) => boolean,  // optional exit condition
): Promise<AssistantMessage>
```

The loop should:
1. Emit `agent_start` and `turn_start` events
2. Call the mock LLM (from Step 3.2) with the current messages
3. Create an `AssistantMessage` from the response
4. Add the assistant message to the messages array
5. Emit `message_start`, `message_update`, `message_end`, and `turn_end` events
6. Check `shouldStopAfterTurn()` — if true, emit `agent_end` and return
7. If not stopping, loop back to step 2

**Hints:**
- Grouping `model`, `apiKey`, `systemPrompt`, and `tools` into a config object makes it easy to extend later (in Milestone 5, you'll add more fields like `provider`).
- The `tools` array is included here even though you haven't built tool execution yet — it's wired up in Milestone 4.

**Hints:**
- For now, the loop exits after one turn (the `shouldStopAfterTurn` returns true after the first response). You'll make it multi-turn in the next milestone.
- The `onEvent` callback is passed in as a parameter — this is how the agent notifies other code about what's happening.
- Track messages in a local array that gets modified as the loop runs.

??? example "Reference pattern"
    === "loop.ts — the runLoop function"
        ```typescript
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
          await onEvent({ type: "agent_start" });

          let lastMessage: AssistantMessage | null = null;

          while (true) {
            await onEvent({ type: "turn_start" });

            // Call the LLM (mock for now)
            const assistantMessage = await mockLLMCall(messages);

            // Add to conversation
            messages.push(assistantMessage);
            lastMessage = assistantMessage;

            // Emit message events
            await onEvent({ type: "message_start", message: assistantMessage });
            await onEvent({ type: "message_update", message: assistantMessage });
            await onEvent({ type: "message_end", message: assistantMessage });
            await onEvent({ type: "turn_end", message: assistantMessage });

            // Check if we should stop
            if (shouldStopAfterTurn?.(assistantMessage) ?? true) {
              break;
            }
          }

          await onEvent({ type: "agent_end", messages });
          return lastMessage!;
        }
        ```

---

## Step 3.4: Add the `prompt` method to the Agent class

**Goal:** Give the Agent a public `prompt()` method that wraps `runLoop`.

**Requirements:**
Add to your `Agent` class in `src/agent.ts`:

- A `prompt(input: string, onEvent?: OnEvent): Promise<void>` method that:
  1. Checks if `isStreaming` is true — if so, throw an error ("Agent is already processing")
  2. Creates a `UserMessage` from the input string with a timestamp
  3. Adds it to `this._state.messages`
  4. Sets `this._state.isStreaming = true`
  5. Calls `runLoop()` with the current messages, system prompt, and event callback
  6. On completion, sets `isStreaming = false`

**Hints:**
- The "already processing" check prevents concurrent prompts from corrupting state.
- Use a try/finally block to ensure `isStreaming` is set to false even if an error occurs.
- The `onEvent` callback defaults to a no-op function if not provided.

??? example "Reference pattern"
    === "agent.ts — add the prompt method"
        ```typescript
        import { runLoop } from "./loop.js";
        import type { OnEvent } from "./types.js";

        // ... inside the Agent class ...

        async prompt(input: string, onEvent?: OnEvent): Promise<void> {
          if (this._state.isStreaming) {
            throw new Error("Agent is already processing a prompt.");
          }

          // Create user message
          const message: UserMessage = {
            role: "user",
            content: [{ type: "text", text: input }],
            timestamp: Date.now(),
          };

          this._state.messages.push(message);
          this._state.isStreaming = true;

          try {
            // Run the loop with config
            await runLoop(
              {
                model: this._state.model,
                apiKey: "", // will be set from config in Milestone 4
                systemPrompt: this._state.systemPrompt,
                tools: this._state.tools,
              },
              this._state.messages,
              onEvent ?? (() => {}),
            );
          } finally {
            this._state.isStreaming = false;
          }
        }
        ```

---

## Step 3.5: Wire the CLI to test the loop with a mock response

**Goal:** Update `cli.ts` to create an agent, call `prompt()`, and print events as they're emitted.

**Requirements:**
- In your CLI, after creating the Agent:
  - Define an event logger function that prints each event type
  - Call `await agent.prompt("Hello, agent!")` with the event logger
  - After completion, print the final assistant message content

**Hints:**
- The event logger is a simple function that matches on `event.type` and prints accordingly.
- The mock response should include some interesting text so you can verify the flow end-to-end.

??? example "Reference pattern"
    === "cli.ts — update the main function"
        ```typescript
        import { loadConfig } from "./config.js";
        import { Agent } from "./agent.js";

        async function main() {
          try {
            const config = loadConfig();
            const agent = new Agent({
              systemPrompt: config.systemPrompt,
              model: config.model,
            });

            // Event logger
            const onEvent = (event) => {
              switch (event.type) {
                case "agent_start":
                  console.log("[Agent started]");
                  break;
                case "turn_start":
                  console.log("[Turn started]");
                  break;
                case "message_start":
                  console.log("[Message started]");
                  break;
                case "message_update":
                  console.log(`  ${event.message.content.map(c => c.text).join("")}`);
                  break;
                case "message_end":
                  console.log("[Message ended]");
                  break;
                case "turn_end":
                  console.log("[Turn ended]");
                  break;
                case "agent_end":
                  console.log("[Agent finished]");
                  break;
              }
            };

            await agent.prompt("Hello, agent!", onEvent);
            process.exit(0);
          } catch (error) {
            const message = error instanceof Error ? error.message : String(error);
            console.error(`Error: ${message}`);
            process.exit(1);
          }
        }

        main();
        ```

---

## Final Check for This Milestone

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
