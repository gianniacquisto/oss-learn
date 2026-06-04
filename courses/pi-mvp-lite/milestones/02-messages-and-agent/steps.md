# Steps — Messages & Agent State

[:material-arrow-left: Back to Milestone Overview](README.md){: .md-button }

---

## Step 2.1: Define core message types

**Goal:** Create TypeScript interfaces in `src/types.ts` that describe every kind of message your agent will handle.

### Why This Step Exists

Every piece of information flowing through your agent — what the user says, what the LLM generates, what a tool returns — is represented as a message. Defining these types upfront gives you compile-time guarantees: if a function expects a `UserMessage`, TypeScript won't let you accidentally pass an `AssistantMessage`. This eliminates entire classes of runtime bugs.

More importantly, the `role` field on each type acts as a **discriminant** — a shared field TypeScript uses to narrow union types. When you write `if (msg.role === "user")`, TypeScript knows `msg` is a `UserMessage` inside that block and shows you only the fields that exist on that type.

### The Message Lifecycle

```text
Conversation starts:
  SystemMessage ────────────┐
                            │
User: "What's in README?"   │
  UserMessage ──────────────┤──→ Sent to LLM
                            │    (as message array)
LLM responds                │
  AssistantMessage ─────────┤──→ Stored by Agent
                            │
Tool output                 │
  ToolResultMessage ────────┤──→ Appended to conversation
                            │
Next turn:                  │
  All messages above        ┤──→ Sent to LLM again (full context)
  plus new UserMessage      │
```

The LLM sees the entire conversation history on every call. The `Agent` class owns this list and manages it — you'll build that in Step 2.3.

### Requirements

Create these types in `src/types.ts`:

#### TextContent

A building block used inside messages:

```typescript
// A text content block — the smallest unit of message content
export interface TextContent {
  type: "text";   // discriminant for this block type
  text: string;   // the actual text
}
```

**Why not just use strings directly?** Because later (Milestone 4) assistant messages can contain both text blocks *and* tool call blocks in the same `content` array. Starting with `TextContent[]` now avoids refactoring later.

#### UserMessage

Represents what the user types:

```typescript
export interface UserMessage {
  role: "user";                    // discriminant — tells TypeScript this is a user message
  content: TextContent[];          // text blocks (usually just one)
  timestamp: number;               // when this message was created
}
```

**Why the `role` field?** It's how the LLM API knows this message came from the human, not the assistant. It's also your discriminating field for TypeScript type narrowing.

#### AssistantMessage

Represents what the LLM generates:

```typescript
export interface AssistantMessage {
  role: "assistant";                    // discriminant
  content: TextContent[];               // text blocks (will grow to include tool calls later)
  model: string;                        // which model generated this (e.g., "gpt-4o")
  provider: string;                     // which provider ("openai", "mock", etc.)
  stopReason: "stop" | "toolCalls" | "error" | "aborted";  // why did the model stop?
  errorMessage?: string;                // present only if stopReason is "error"
  timestamp: number;                    // when this message was created
}
```

**Why `stopReason`?** The LLM can stop generating for different reasons — it finished naturally (`"stop"`), it wants to call a tool (`"toolCalls"`), something went wrong (`"error"`), or the user cancelled (`"aborted"`). Your loop code needs to know *why* the model stopped to decide what to do next.

#### ToolResultMessage

Represents the output from running a tool:

```typescript
export interface ToolResultMessage {
  role: "toolResult";                 // discriminant
  toolCallId: string;                 // links this result back to the tool call that produced it
  toolName: string;                   // which tool was executed
  content: TextContent[];             // the output (text from the tool)
  isError: boolean;                   // did the tool succeed or fail?
  timestamp: number;                  // when this result was generated
}
```

**Why `toolCallId`?** When the LLM calls multiple tools at once, each call gets a unique ID. The result must include that ID so the LLM knows which result belongs to which call. Without it, the model can't match outputs to inputs.

#### SystemMessage

Represents the system prompt (instructions for the LLM):

```typescript
export interface SystemMessage {
  role: "system";     // discriminant
  content: string;    // plain string (not an array — system prompts are simpler)
  timestamp?: number; // optional — system messages don't always need timestamps
}
```

**Why `content: string` instead of `TextContent[]`?** System prompts are always plain text — they never contain tool calls or other structured content. Keeping this simpler avoids unnecessary complexity.

#### The Union Type

Finally, combine everything into a single union:

```typescript
export type AgentMessage =
  | UserMessage
  | AssistantMessage
  | ToolResultMessage
  | SystemMessage;
```

**Why a union type?** It lets you write functions that accept *any* message (`messages: AgentMessage[]`) while still getting precise type information when you check the `role` field. This is the backbone of your entire message system.

### ??? tip "Hints"

- Type names use **PascalCase** (e.g., `UserMessage`) because they're interfaces/types, not values.
- The `role` field uses a **string literal type** (`"user"`, not `string`). This means TypeScript knows the exact possible values and will error if you typo them.
- Every message except `SystemMessage` requires a `timestamp`. Use `Date.now()` when creating messages to get the current time in milliseconds.

### ??? example "What src/types.ts should look like after this step"

```typescript
export interface TextContent {
  type: "text";
  text: string;
}

export interface UserMessage {
  role: "user";
  content: TextContent[];
  timestamp: number;
}

export interface AssistantMessage {
  role: "assistant";
  content: TextContent[];
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

export type AgentMessage =
  | UserMessage
  | AssistantMessage
  | ToolResultMessage
  | SystemMessage;
```

### ??? warning "Common Mistake"

**Using `string` for `role` instead of string literals:** If you write `role: string`, TypeScript can't use the role field to narrow types. Always use exact string literal types (`role: "user"`) on discriminated unions.

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] All five interfaces and the union type exist in `src/types.ts`
- [ ] Running `npm run build` compiles without errors
- [ ] Each interface has a unique `role` string literal

---

## Step 2.2: Define the Tool interface

**Goal:** Add a `Tool` interface and `ToolResult` type to `src/types.ts` that describe what every tool must implement.

### Why This Step Exists

Before you can execute tools (Milestone 4), you need to agree on their shape. The `Tool` interface is the contract: any tool you create must have a name, a description (for the LLM to understand when to use it), and an `execute` function (the actual code that runs).

The LLM doesn't execute tools — *your code* does. The LLM merely requests a tool call with a name and arguments. Your harness finds the registered tool by name, validates the arguments, calls `execute()`, and feeds the result back to the LLM. The `Tool` interface defines what "registered" means.

### Requirements

Add these types to `src/types.ts` (after the message types):

#### ToolResult

The output from running any tool:

```typescript
export interface ToolResult {
  content: TextContent[];   // text output returned to the LLM
  isError: boolean;         // true if the execution failed
}
```

**Why separate `isError` from the content?** The LLM needs to know whether the tool succeeded or failed *in addition to* seeing the output. A failed file read might still return useful error text, but the LLM should treat it differently than a successful read.

#### Tool

The contract every tool must implement:

```typescript
export interface Tool {
  name: string;                                                // unique identifier (e.g., "read_file")
  description: string;                                         // tells the LLM when to use this tool
  requiredFields: string[];                                    // argument names that must be present
  execute(toolCallId: string, args: Record<string, unknown>): Promise<ToolResult>;  // the actual function
}
```

**Why each field matters:**

| Field | Used By | Why |
|-------|---------|-----|
| `name` | Your code | Finds the right tool when the LLM requests one by name |
| `description` | The LLM | Helps the model decide *which* tool to use and *when* |
| `requiredFields` | Your code | Validates arguments before execution — prevents crashes from missing inputs |
| `execute()` | Your code | Runs the actual logic when the tool is called |

**Why `toolCallId` in `execute()`?** Each tool invocation gets a unique ID so your code can match results back to their calls. The execute function receives it but doesn't need to do anything with it — it just passes it through when creating the result.

**Why `Record<string, unknown>` for args?** Tool arguments come as JSON from the LLM (or mock), so they're generic key-value pairs. Using `unknown` (not `any`) forces you to cast and validate before using them — TypeScript won't let you call `.split()` on an `unknown` without a type guard or cast.

### ??? example "What src/types.ts should look like after this step"

```typescript
// ... (message types from Step 2.1) ...

export interface ToolResult {
  content: TextContent[];
  isError: boolean;
}

export interface Tool {
  name: string;
  description: string;
  requiredFields: string[];
  execute(
    toolCallId: string,
    args: Record<string, unknown>,
  ): Promise<ToolResult>;
}
```

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `ToolResult` interface exists with `content` and `isError`
- [ ] `Tool` interface exists with `name`, `description`, `requiredFields`, and `execute`
- [ ] Running `npm run build` compiles without errors

---

## Step 2.3: Implement the Agent class with state management

**Goal:** Create an `Agent` class in `src/agent.ts` that manages conversation state, tools, and lifecycle.

### Why This Step Exists

The Agent class is the central coordinator of your entire system. It owns the conversation history, knows which tools are available, tracks whether it's currently processing a prompt, and surfaces errors to callers. Every other part of your code (the loop, the CLI, future UI components) talks to the agent through this class.

Think of it as a black box with three capabilities:
1. **State:** Read the agent's current condition (messages, tools, status)
2. **Configuration:** Set tools and model settings
3. **Control:** Reset the conversation, prompt the agent (Milestone 3)

### Requirements

#### Part A: Define `AgentState` (the public, readonly interface)

```typescript
export interface AgentState {
  readonly systemPrompt: string;   // instructions sent to the LLM
  readonly model: string;          // current model name
  readonly tools: Tool[];          // registered tools
  readonly messages: AgentMessage[];  // full conversation history
  readonly isStreaming: boolean;   // is a prompt currently being processed?
  readonly errorMessage?: string;  // error from the last failed turn
}
```

**Why `readonly` on every field?** This is TypeScript's way of saying "you can read these values but you can't assign to them." It enforces encapsulation — only the Agent class itself modifies its own state through private methods.

#### Part B: Define mutable internal state (private)

```typescript
interface MutableAgentState {
  systemPrompt: string;
  model: string;
  tools: Tool[];
  messages: AgentMessage[];
  isStreaming: boolean;
  errorMessage?: string;
}
```

Same fields, but without `readonly`. This type is used *only* inside the class — it's not exported.

**Why two types instead of one?** The public API says "these fields can't be changed from outside" (`AgentState`). The private implementation needs to actually modify them (`MutableAgentState`). Using two types makes this contract explicit and lets TypeScript enforce it.

#### Part C: Create the Agent class

```typescript
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

  // Readonly access to current state
  get state(): AgentState {
    return { ...this._state };
  }

  // Setter for tools — the one field callers *can* change
  set tools(tools: Tool[]) {
    this._state.tools = tools;
  }

  // Clear conversation history but keep config
  reset(): void {
    this._state.messages = [];
    this._state.errorMessage = undefined;
  }
}
```

**Why the spread in `get state()`?** `{ ...this._state }` creates a shallow copy of the internal state. Without it, callers could do `agent.state.messages.push(...)` to mutate the private state directly. The copy prevents this — callers get a snapshot, not a reference.

**Why optional constructor params?** You want the Agent to work with minimal configuration: `new Agent()` creates one with defaults, while `new Agent({ model: "gpt-4o", tools: [...] })` lets you customize. The `??` operator provides defaults for any missing values.

**Why `reset()` doesn't clear tools or model?** When the user starts a new conversation, they typically want to keep the same agent configuration (same tools, same model) but start fresh with an empty message history. Resetting everything would require reconfiguring the entire agent for each conversation.

### ??? tip "Hints"

- Import `getDefaultSystemPrompt` from `./config.js` — you created this in Milestone 1
- Import types from `./types.js` using the `.js` extension (required by NodeNext module resolution)
- The `private _state` field uses the underscore prefix — a convention indicating "this is internal, don't access it directly"

### ??? warning "Common Mistake"

**Returning `this._state` directly from `get state()`:** Without the spread (`{ ... }`), callers get a reference to the actual internal state object. They can then do `agent.state.messages = []` and silently corrupt the agent's memory. Always return a copy or use `Object.freeze()`.

### ??? example "Complete agent.ts"

```typescript
import { getDefaultSystemPrompt } from "./config.js";
import type { AgentMessage, Tool } from "./types.js";

export interface AgentState {
  readonly systemPrompt: string;
  readonly model: string;
  readonly tools: Tool[];
  readonly messages: AgentMessage[];
  readonly isStreaming: boolean;
  readonly errorMessage?: string;
}

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

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `Agent` class exports from `src/agent.ts`
- [ ] `new Agent()` works (creates an agent with defaults)
- [ ] `agent.state` returns an object with all expected fields
- [ ] `agent.reset()` clears messages but preserves systemPrompt and model
- [ ] `agent.state.messages.push(...)` fails at compile time (readonly enforcement)

---

## Step 2.4: Wire the CLI to create and inspect an Agent

**Goal:** Update `src/cli.ts` to instantiate an `Agent`, configure it from loaded config, and print its state as JSON.

### Why This Step Exists

You've built types and a class in isolation. Now you need to verify they work together through the same entry point (`cli.ts`) that a user would run. Printing the agent's state as JSON is your first "debug view" — it lets you inspect the internal structure of your agent at a glance.

This also establishes the pattern for how the CLI will interact with the Agent in later milestones: load config → create agent → register tools (later) → call prompt (later).

### Requirements

Update `src/cli.ts`:

1. **Import the Agent class:**
   ```typescript
   import { Agent } from "./agent.js";
   ```

2. **Create an agent from config and print its state:**
   ```typescript
   async function main() {
     try {
       const config = loadConfig();

       const agent = new Agent({
         systemPrompt: config.systemPrompt,
         model: config.model,
       });

       console.log("Agent created with state:");
       console.log(JSON.stringify(agent.state, null, 2));
       process.exit(0);
     } catch (error) {
       const message = error instanceof Error ? error.message : String(error);
       console.error(`Error: ${message}`);
       process.exit(1);
     }
   }

   main();
   ```

**Why `JSON.stringify(state, null, 2)`?** The second argument (`null`) means no custom replacer function. The third argument (`2`) means "indent with 2 spaces" — it makes the output readable instead of one massive line.

### ??? tip "Hints"

- Make sure your config is loaded before creating the Agent — the agent uses `config.model` and `config.systemPrompt`
- The `.js` extension in imports is still required even though the source files are `.ts`
- If you want to test with tools, create a mock tool object that satisfies the `Tool` interface

### ??? example "Expected output when running npm start"

```
Agent created with state:
{
  "systemPrompt": "You are a helpful coding assistant...",
  "model": "gpt-4o",
  "tools": [],
  "messages": [],
  "isStreaming": false
}
```

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `npm run build` compiles without errors
- [ ] `AGENT_API_KEY=fake npm start` prints agent state as formatted JSON
- [ ] The JSON shows `messages: []`, `tools: []`, and `isStreaming: false`
- [ ] The systemPrompt matches your default or config value

---

## All Done? Check Your Work [:material-arrow-right:](done.md)

Before moving to Milestone 3, follow the verification steps in [done.md](./done.md).
