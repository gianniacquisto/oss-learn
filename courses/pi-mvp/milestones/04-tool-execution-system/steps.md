# Steps for Milestone 4: Tool Execution System

---

## Step 4.1: Implement tool call detection

**Goal:** Add logic to detect whether an assistant message contains tool calls and extract them.

**Requirements:**
Add a helper function in `src/loop.ts`:

```typescript
function hasToolCalls(message: AssistantMessage): boolean;
function getToolCalls(message: AssistantMessage): ToolCallBlock[];
```

- `hasToolCalls` checks if any content block in the message has `type === "toolCall"`
- `getToolCalls` filters and returns all tool call blocks from a message

**Hints:**
- These are simple array operations — `message.content.some()` and `.filter()`. Don't overthink this step.
- The real work is what you do *after* detecting the calls.

---

## Step 4.2: Implement argument validation with Zod

**Goal:** Create a function that validates tool call arguments against the registered tool's schema before execution.

**Requirements:**
Add to `src/loop.ts`:

- A `validateToolArguments(tool: Tool, toolCall: ToolCallBlock): unknown` function that:
  - Uses Zod to validate `toolCall.arguments` against `tool.parameters`
  - Returns the validated arguments if successful
  - Throws a descriptive error if validation fails (e.g., "Missing required field 'path'")

- Update your `Tool` interface in `types.ts` so `parameters` is a Zod schema:
  ```typescript
  parameters: z.ZodType<any>;  // or z.AnyZodObject for object schemas
  ```

**Hints:**
- You'll need to import Zod and use `tool.parameters.parse(args)` for validation. If it throws, catch the error and rethrow with a clearer message.
- For tool parameter schemas, you can start simple: `{ path: z.string(), content: z.string().optional() }` for a file-write tool.

**Reference pattern:**
```typescript
// loop.ts — structural reference only

import { z } from "zod";

export function validateToolArguments(tool: Tool, toolCall: ToolCallBlock): unknown {
  try {
    return tool.parameters.parse(toolCall.arguments);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    throw new Error(`Tool "${tool.name}" validation failed: ${message}`);
  }
}
```

---

## Step 4.3: Implement tool execution

**Goal:** Create a function that executes a single tool call and returns the result.

**Requirements:**
Add to `src/loop.ts`:

- An `executeToolCall` function with this signature:
  ```typescript
  async function executeToolCall(
    toolCall: ToolCallBlock,
    tool: Tool,
    validatedArgs: unknown,
    signal?: AbortSignal,
  ): Promise<ToolResult>;
  ```

- The function should:
  1. Call `tool.execute(toolCall.id, validatedArgs, signal)`
  2. Catch any exceptions and return `{ content: [{ type: "text", text: error.message }], isError: true }`
  3. Return the tool's result on success

- Add a `createToolResultMessage(result: ToolResult, toolCall: ToolCallBlock): ToolResultMessage` helper that formats the result into a message

**Hints:**
- The `signal` parameter is important — pass it through so long-running tools can be cancelled.
- Always wrap in try/catch. A single failing tool shouldn't crash the entire loop.
- `createToolResultMessage` creates a `ToolResultMessage` with role `"toolResult"` that will be appended to context and sent back to the LLM.

**Reference pattern:**
```typescript
// loop.ts — structural reference only

async function executeToolCall(
  toolCall: ToolCallBlock,
  tool: Tool,
  validatedArgs: unknown,
  signal?: AbortSignal,
): Promise<ToolResult> {
  try {
    return await tool.execute(toolCall.id, validatedArgs, signal);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    return { content: [{ type: "text", text: `Error executing ${tool.name}: ${message}` }], isError: true };
  }
}

function createToolResultMessage(result: ToolResult, toolCall: ToolCallBlock): ToolResultMessage {
  return {
    role: "toolResult",
    toolCallId: toolCall.id,
    toolName: toolCall.name,
    content: result.content,
    isError: result.isError,
    timestamp: Date.now(),
  };
}
```

---

## Step 4.4: Add tools to the Agent class and wire them into the loop

**Goal:** Give the Agent a `registerTool()` method and update the loop to execute tool calls when present.

**Requirements:**
Update your `Agent` class in `src/agent.ts`:

- Add a `registerTool(tool: Tool): void` method that adds the tool to `_state.tools`
- Update the `createConfig()` helper to include tools in the context snapshot passed to `runLoop`

Update `runLoop` in `src/loop.ts`:

- After getting an assistant response, check if it has tool calls using `hasToolCalls()`
- If yes:
  - For each tool call, find the matching registered tool by name
  - Validate arguments with `validateToolArguments()`
  - Execute with `executeToolCall()`
  - Create tool result messages and append them to context
  - Emit `tool_execution_start`, `message_start`/`message_end` for each result
- After executing all tools, go back to the top of the loop (call the LLM again with results appended)
- If no tool calls: check `shouldStopAfterTurn()` and exit if true

**Hints:**
- The key structural change: your loop now has an inner loop (process tool calls from one response) inside the outer loop (keep calling the LLM until done). This is the "agent loop" pattern.
- For this milestone, execute tools sequentially (one at a time). Parallel execution is a Milestone 5+ enhancement.
- After executing tools and appending results to context, you need to call `convertToLlm()` again before the next LLM call — the message array has changed.

**Reference pattern:**
```typescript
// loop.ts runLoop update — structural reference only (showing the tool execution branch)

async function runLoop(context: AgentContext, config: AgentLoopConfig, emit, signal?) {
  // ... setup events and initial LLM call as before ...

  const assistantMessage = /* result from LLM call */;
  context.messages.push(assistantMessage);

  if (hasToolCalls(assistantMessage)) {
    await emit({ type: "turn_start" });

    for (const toolCall of getToolCalls(assistantMessage)) {
      // Find the registered tool
      const tool = context.tools?.find(t => t.name === toolCall.name);
      if (!tool) {
        // Error: unknown tool
        continue;
      }

      // Validate and execute
      const validatedArgs = validateToolArguments(tool, toolCall);
      const result = await executeToolCall(toolCall, tool, validatedArgs, signal);
      const toolResultMessage = createToolResultMessage(result, toolCall);

      context.messages.push(toolResultMessage);
      await emit({ type: "message_start", message: toolResultMessage });
      await emit({ type: "message_end", message: toolResultMessage });
    }

    // Loop back — call LLM again with results appended
    return runLoop(context, config, emit, signal);  // recursive call
  }

  // No tool calls — we're done
  await emit({ type: "turn_end", message: assistantMessage, toolResults: [] });
  await emit({ type: "agent_end", messages: context.messages });
  return assistantMessage;
}
```

---

## Step 4.5: Register a simple built-in tool for testing

**Goal:** Add at least one concrete tool so you can test the full flow end-to-end.

**Requirements:**
Create in `src/tools/` (new directory):

- A `readFileTool` that reads a file from disk:
  - name: `"read_file"`
  - description: `"Read the contents of a file at the given path."`
  - parameters: `{ path: z.string() }`
  - execute: uses `fs.readFile()` to read the file and returns the content

- Register this tool in your CLI before calling `prompt()`:
  ```typescript
  agent.registerTool(readFileTool);
  ```

**Hints:**
- Use `z.object({ path: z.string() })` for the parameter schema.
- The execute function should return `{ content: [{ type: "text", text: fileContent }], isError: false }`.
- Handle errors gracefully — if the file doesn't exist, return `{ isError: true, content: [...] }` rather than throwing.

**Reference pattern:**
```typescript
// tools/read-file.ts — structural reference only

import { z } from "zod";
import fs from "fs/promises";
import type { Tool } from "../types.js";

export const readFileTool: Tool<z.infer<typeof readFileSchema>> = {
  name: "read_file",
  description: "Read the contents of a file at the given path.",
  parameters: z.object({ path: z.string() }),
  execute: async (toolCallId, args) => {
    try {
      const content = await fs.readFile(args.path, "utf-8");
      return { content: [{ type: "text", text: content }], isError: false };
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      return { content: [{ type: "text", text: `Failed to read file: ${message}` }], isError: true };
    }
  },
};
```

---

## Final Check for This Milestone

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
