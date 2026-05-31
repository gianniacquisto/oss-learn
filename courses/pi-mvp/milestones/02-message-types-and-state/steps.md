# Steps for Milestone 2: Message Types & Agent State

---

## Step 2.1: Define core message types

**Goal:** Create type definitions for all message roles your agent will handle.

**Requirements:**
Define these types in `src/types.ts`:

- **`TextContent`** — a text block with a `type: "text"` and `text: string` field
- **`ImageContent`** — an image block (you can keep this simple for now: `{ type: "image"; data: string; mimeType: string }`)
- **`UserMessage`** — role `"user"`, content is `(TextContent | ImageContent)[]`, plus a `timestamp: number`
- **`AssistantMessage`** — role `"assistant"`, content is `(TextContent | ImageContent | ToolCallBlock)[]`, plus metadata fields (`model`, `provider`, `stopReason`)
- **`ToolCallBlock`** — represents a tool call within an assistant message: `{ type: "toolCall"; id: string; name: string; arguments: Record<string, unknown> }`
- **`ToolResultMessage`** — role `"toolResult"`, with `toolCallId`, `toolName`, `content`, and `isError` fields
- **`SystemMessage`** — role `"system"`, content is a plain string

Then create a union type:
```typescript
export type AgentMessage = UserMessage | AssistantMessage | ToolResultMessage | SystemMessage;
```

**Hints:**
- Use discriminated unions (a literal `type` or `role` field) so TypeScript can narrow the type in conditionals.
- The `timestamp` field is useful for ordering and debugging — every message should have one.
- Keep `ImageContent` minimal for now. You can expand it later when you add image support.

**Reference pattern:**
```typescript
// types.ts — structural reference only

export interface TextContent {
  type: "text";
  text: string;
}

export interface ImageContent {
  type: "image";
  data: string;       // base64-encoded or URL
  mimeType: string;   // e.g., "image/png"
}

export interface ToolCallBlock {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, unknown>;
}

export interface UserMessage {
  role: "user";
  content: (TextContent | ImageContent)[];
  timestamp: number;
}

export interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ImageContent | ToolCallBlock)[];
  model: string;
  provider: string;
  stopReason: "stop" | "toolCalls" | "error" | "aborted";
  errorMessage?: string;
  timestamp: number;
}

export interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: TextContent[];
  isError: boolean;
  timestamp: number;
}

export interface SystemMessage {
  role: "system";
  content: string;
  timestamp?: number;
}

export type AgentMessage = UserMessage | AssistantMessage | ToolResultMessage | SystemMessage;
```

---

## Step 2.2: Define the tool type

**Goal:** Create a `Tool` interface that describes what tools look like from the agent's perspective.

**Requirements:**
Define in `src/types.ts`:

- **`Tool<TParameters>`** — a generic interface with:
  - `name: string` — unique identifier for the tool
  - `description: string` — human-readable description (used by the LLM to decide when to call this tool)
  - `parameters: TSchema` — JSON Schema describing the expected arguments (use TypeBox's `Type.Object()` or a generic type placeholder for now)
  - `execute(toolCallId: string, args: unknown, signal?: AbortSignal): Promise<ToolResult>` — the function that runs when this tool is called

- **`ToolResult`** — the result of executing a tool:
  - `content: TextContent[]` — text or image content returned to the model
  - `isError: boolean` — whether the execution failed

**Hints:**
- The `TSchema` type for parameters is where TypeBox comes in. For now, you can use `unknown` as a placeholder and refine it later when you integrate TypeBox properly.
- The `signal` parameter on `execute` lets callers cancel long-running tool executions. This is important for tools like file reading or command execution that might hang.

**Reference pattern:**
```typescript
// types.ts — structural reference only (add after message types)

export interface ToolResult {
  content: TextContent[];
  isError: boolean;
}

export interface Tool<TParameters = unknown> {
  name: string;
  description: string;
  parameters: TParameters;
  execute(
    toolCallId: string,
    args: TParameters,
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
  - Setter for `tools` on the public interface (assigning a new array replaces available tools)
  - A `reset()` method that clears messages and error state but keeps system prompt and model

**Hints:**
- The key design pattern here: private mutable state + readonly public getter. Callers get a snapshot of the current state but can't mutate it directly. They use `agent.state.tools = [...]` to replace tools, not push/pop on the array.
- Default values matter: what should the system prompt be if none is provided? Use your `getDefaultSystemPrompt()` from Milestone 1.
- The `model` field stores which model was last used — this lets you switch models between turns (useful for adaptive reasoning).

**Reference pattern:**
```typescript
// agent.ts — structural reference only

export interface AgentState {
  readonly systemPrompt: string;
  readonly model: string;
  readonly tools: Tool[];
  readonly messages: AgentMessage[];
  readonly isStreaming: boolean;
  readonly errorMessage?: string;
}

export class Agent {
  private _state: MutableAgentState;

  constructor(options?: { systemPrompt?: string; model?: string; tools?: Tool[] })

  get state(): AgentState

  set tools(tools: Tool[])

  reset(): void
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

**Reference pattern:**
```typescript
// cli.ts update — structural reference only

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
    // ... error handling from Milestone 1
  }
}
```

---

## Final Check for This Milestone

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
