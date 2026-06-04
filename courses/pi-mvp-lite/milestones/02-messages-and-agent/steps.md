# Steps for Milestone 2: Messages & Agent State

---

## Step 2.1: Define core message types

**Goal:** Create type definitions for all message roles your agent will handle.

**Requirements:**
Define these types in `src/types.ts`:

- **`TextContent`** — a text block with `type: "text"` and `text: string`
- **`UserMessage`** — role `"user"`, content is an array of text blocks, plus a `timestamp: number`
- **`AssistantMessage`** — role `"assistant"`, content is an array of text blocks (or tool calls — you'll add that in Milestone 3), plus metadata fields (`model`, `provider`, `stopReason`)
- **`ToolResultMessage`** — role `"toolResult"`, with `toolCallId`, `toolName`, `content`, and `isError` fields
- **`SystemMessage`** — role `"system"`, content is a plain string

Then create a union type:
```typescript
export type AgentMessage = UserMessage | AssistantMessage | ToolResultMessage | SystemMessage;
```

**Hints:**
- Use a `role` field as a "discriminator" — TypeScript can check the role to figure out which type of message you have.
- The `timestamp` field is useful for ordering and debugging — every message should have one.
- Keep `AssistantMessage` simple for now — just text content. You'll add tool calls in Milestone 3.

??? example "Reference pattern"
    === "types.ts — add to the empty file"
        ```typescript
        // Text content block
        export interface TextContent {
          type: "text";
          text: string;
        }

        // User message — what the user types
        export interface UserMessage {
          role: "user";
          content: TextContent[];
          timestamp: number;
        }

        // Assistant message — what the LLM generates
        export interface AssistantMessage {
          role: "assistant";
          content: TextContent[];
          model: string;
          provider: string;
          stopReason: "stop" | "toolCalls" | "error" | "aborted";
          errorMessage?: string;
          timestamp: number;
        }

        // Tool result — output from running a tool
        export interface ToolResultMessage {
          role: "toolResult";
          toolCallId: string;
          toolName: string;
          content: TextContent[];
          isError: boolean;
          timestamp: number;
        }

        // System message — instructions for the LLM
        export interface SystemMessage {
          role: "system";
          content: string;
          timestamp?: number;
        }

        // Union of all message types
        export type AgentMessage =
          | UserMessage
          | AssistantMessage
          | ToolResultMessage
          | SystemMessage;
        ```

---

## Step 2.2: Define the tool type

**Goal:** Create a `Tool` interface that describes what tools look like from the agent's perspective.

**Requirements:**
Define in `src/types.ts`:

- **`ToolResult`** — the result of executing a tool:
  - `content: TextContent[]` — text content returned to the model
  - `isError: boolean` — whether the execution failed

- **`Tool`** — a tool definition:
  - `name: string` — unique identifier for the tool
  - `description: string` — human-readable description (used by the LLM to decide when to call this tool)
  - `execute(toolCallId: string, args: Record<string, unknown>, signal?: AbortSignal): Promise<ToolResult>` — the function that runs when this tool is called

**Hints:**
- The `signal` parameter on `execute` lets callers cancel long-running tool executions. This is important for tools like file reading or command execution that might hang.
- Using `Record<string, unknown>` for `args` keeps things simple for now. You'll add proper validation in Milestone 3.

??? example "Reference pattern"
    === "types.ts — add after message types"
        ```typescript
        // Result from executing a tool
        export interface ToolResult {
          content: TextContent[];
          isError: boolean;
        }

        // A tool the agent can use
        export interface Tool {
          name: string;
          description: string;
          execute(
            toolCallId: string,
            args: Record<string, unknown>,
            signal?: AbortSignal,
          ): Promise<ToolResult>;
        }
        ```

---

## Step 2.3: Implement the Agent class with state management

**Goal:** Create an `Agent` class that manages conversation context, tools, and lifecycle state.

**Requirements:**
Create in `src/agent.ts`:

- An `AgentState` interface (readonly) exposing:
  - `systemPrompt: string` — the system prompt sent to the model
  - `model: string` — current model being used
  - `tools: Tool[]` — available tools
  - `messages: AgentMessage[]` — conversation transcript
  - `isStreaming: boolean` — whether a run is in progress
  - `errorMessage?: string` — error from the most recent failed turn

- An `Agent` class with:
  - A constructor that accepts an options object: `{ systemPrompt?: string, model?: string, tools?: Tool[] }`
  - Private mutable state (`_state`) initialized from options or defaults
  - Getter for `state: AgentState` that returns the current state
  - A `reset()` method that clears messages and error state but keeps system prompt and model

**Hints:**
- The key design pattern: private mutable state + readonly public getter. Callers can read `agent.state` but can't modify it directly. They use `agent.tools = [...]` to replace tools.
- Default values matter: what should the system prompt be if none is provided? Use your `getDefaultSystemPrompt()` from Milestone 1.
- The `model` field stores which model was last used — this lets you switch models between turns.

??? example "Reference pattern"
    === "agent.ts"
        ```typescript
        import { getDefaultSystemPrompt } from "./config.js";
        import type { AgentMessage, Tool } from "./types.js";

        // Readonly interface for external consumers
        export interface AgentState {
          readonly systemPrompt: string;
          readonly model: string;
          readonly tools: Tool[];
          readonly messages: AgentMessage[];
          readonly isStreaming: boolean;
          readonly errorMessage?: string;
        }

        // Mutable internal state
        interface MutableAgentState {
          systemPrompt: string;
          model: string;
          tools: Tool[];
          messages: AgentMessage[];
          isStreaming: boolean;
          errorMessage?: string;
        }

        export class Agent {
          private _state: MutableAgentState;

          constructor(options?: {
            systemPrompt?: string;
            model?: string;
            tools?: Tool[];
          }) {
            this._state = {
              systemPrompt: options?.systemPrompt ?? getDefaultSystemPrompt(),
              model: options?.model ?? "gpt-4o",
              tools: options?.tools ?? [],
              messages: [],
              isStreaming: false,
            };
          }

          get state(): AgentState {
            return { ...this._state };
          }

          set tools(tools: Tool[]) {
            this._state.tools = tools;
          }

          reset(): void {
            this._state.messages = [];
            this._state.errorMessage = undefined;
          }
        }
        ```

---

## Step 2.4: Wire the CLI to create an Agent instance

**Goal:** Update `cli.ts` to instantiate an Agent and print its current state, verifying the class works.

**Requirements:**
- Import `Agent` from `./agent.js`
- Load config (from Milestone 1)
- Create a new `Agent` with the loaded config's system prompt and model
- Print the agent's current state to stdout as JSON (for debugging)
- Keep the "Hello agent" output for now

**Hints:**
- Use `JSON.stringify(agent.state, null, 2)` for readable output.
- You can remove the "Hello agent" print once you verify this milestone works — but keep it if you want a quick sanity check.
- Make sure you're importing from `./agent.js` (not `./agent.ts`) — TypeScript handles the extension translation.

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

            console.log("Agent state:", JSON.stringify(agent.state, null, 2));
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
