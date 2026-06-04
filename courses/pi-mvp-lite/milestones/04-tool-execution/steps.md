# Steps for Milestone 4: Tool Execution

---

## Step 4.1: Define the tool call type

**Goal:** Add a type for tool calls that appear in assistant messages.

**Requirements:**
Add to `src/types.ts`:

- **`ToolCallBlock`** — represents a tool call within an assistant message:
  - `type: "toolCall"`
  - `id: string` — unique identifier for this tool call
  - `name: string` — the tool's name
  - `arguments: Record<string, unknown>` — the arguments passed to the tool

- Update `AssistantMessage.content` to include `ToolCallBlock`:
  ```typescript
  content: (TextContent | ToolCallBlock)[];
  ```

**Hints:**
- `ToolCallBlock` is a discriminated union member — the `type: "toolCall"` field lets TypeScript narrow the type.
- The `id` field is used to match tool results back to their calls. For now, you can generate simple IDs with a counter.

??? example "Reference pattern"
    === "types.ts — add ToolCallBlock"
        ```typescript
        // A tool call within an assistant message
        export interface ToolCallBlock {
          type: "toolCall";
          id: string;
          name: string;
          arguments: Record<string, unknown>;
        }

        // Update AssistantMessage to include tool calls
        export interface AssistantMessage {
          role: "assistant";
          content: (TextContent | ToolCallBlock)[];  // ← added ToolCallBlock
          model: string;
          provider: string;
          stopReason: "stop" | "toolCalls" | "error" | "aborted";
          errorMessage?: string;
          timestamp: number;
        }
        ```

---

## Step 4.2: Update the Tool interface with required fields

**Goal:** Add a `requiredFields` property to the Tool interface so validation knows what to check.

**Requirements:**
Update the `Tool` interface in `src/types.ts`:

- Add `requiredFields: string[]` — the list of argument field names that must be present

**Hints:**
- This is a simple addition — each tool defines which fields it needs. The `read_file` tool needs `"path"`.
- The full course uses Zod schemas for this, but for now, a simple array of field names is enough.

??? example "Reference pattern"
    === "types.ts — update Tool interface"
        ```typescript
        export interface Tool {
          name: string;
          description: string;
          requiredFields: string[];  // ← add this
          execute(
            toolCallId: string,
            args: Record<string, unknown>,
          ): Promise<ToolResult>;
        }
        ```

---

## Step 4.3: Add tool registration to the Agent class

**Goal:** Give the Agent a `registerTool()` method.

**Requirements:**
Update `src/agent.ts`:

- Add a `registerTool(tool: Tool): void` method that pushes the tool onto `_state.tools`
- Update the constructor to accept tools and pass them through

**Hints:**
- `registerTool` is simpler than the full course's approach — it just appends to the array. The full course uses a map for O(1) lookups, but for a small number of tools, an array is fine.
- Make sure the tools are included when you pass them to `runLoop` in the `prompt()` method.

??? example "Reference pattern"
    === "agent.ts — add registerTool"
        ```typescript
        // ... inside the Agent class ...

        registerTool(tool: Tool): void {
          this._state.tools.push(tool);
        }
        ```

---

## Step 4.4: Implement tool call detection

**Goal:** Add helpers to detect and extract tool calls from assistant messages.

**Requirements:**
Add to `src/loop.ts`:

- `hasToolCalls(message: AssistantMessage): boolean` — checks if any content block has `type === "toolCall"`
- `getToolCalls(message: AssistantMessage): ToolCallBlock[]` — filters and returns all tool call blocks
- `createToolResultMessage(result: ToolResult, toolCall: ToolCallBlock): ToolResultMessage` — formats the result into a message

**Hints:**
- These are simple array operations — `message.content.some()` and `.filter()`. Don't overthink this step.
- The real work is what you do *after* detecting the calls.

??? example "Reference pattern"
    === "loop.ts — add tool detection helpers"
        ```typescript
        export function hasToolCalls(message: AssistantMessage): boolean {
          return message.content.some(c => c.type === "toolCall");
        }

        export function getToolCalls(message: AssistantMessage): ToolCallBlock[] {
          return message.content.filter(c => c.type === "toolCall") as ToolCallBlock[];
        }

        export function createToolResultMessage(
          result: ToolResult,
          toolCall: ToolCallBlock,
        ): ToolResultMessage {
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

## Step 4.5: Implement simple argument validation

**Goal:** Validate tool call arguments before execution.

**Requirements:**
Add to `src/loop.ts`:

- A `validateToolArguments` function that checks if required fields are present:
  ```typescript
  function validateToolArguments(
    toolName: string,
    args: Record<string, unknown>,
    requiredFields: string[],
  ): true | string
  ```
  - Returns `true` if validation passes
  - Returns an error message string if validation fails (e.g., `"Missing required field 'path'"`)

**Hints:**
- This is much simpler than the full course's Zod-based validation. You're just checking if required fields are present and are the right type.
- Each tool can define its own required fields. The `read_file` tool needs `path` (a string).
- You don't need a schema library — simple `if/else` checks are enough for this milestone.

??? example "Reference pattern"
    === "loop.ts — add validation"
        ```typescript
        export function validateToolArguments(
          toolName: string,
          args: Record<string, unknown>,
          requiredFields: string[],
        ): true | string {
          for (const field of requiredFields) {
            if (!(field in args)) {
              return `Missing required field '${field}'`;
            }
          }
          return true;
        }
        ```

---

## Step 4.6: Implement tool execution

**Goal:** Create a function that executes a single tool call.

**Requirements:**
Add to `src/loop.ts`:

- An `executeToolCall` function:
  ```typescript
  async function executeToolCall(
    toolCall: ToolCallBlock,
    tool: Tool,
    validatedArgs: Record<string, unknown>,
  ): Promise<ToolResult>;
  ```
  - Calls `tool.execute(toolCall.id, validatedArgs)`
  - Catches any exceptions and returns `{ content: [...], isError: true }`
  - Returns the tool's result on success

**Hints:**
- Always wrap in try/catch. A single failing tool shouldn't crash the entire loop.
- The `signal` parameter is omitted here for simplicity — you'll add it in the full course.

??? example "Reference pattern"
    === "loop.ts — add execution"
        ```typescript
        async function executeToolCall(
          toolCall: ToolCallBlock,
          tool: Tool,
          validatedArgs: Record<string, unknown>,
        ): Promise<ToolResult> {
          try {
            return await tool.execute(toolCall.id, validatedArgs);
          } catch (error) {
            const message = error instanceof Error ? error.message : String(error);
            return {
              content: [{ type: "text", text: `Error executing ${tool.name}: ${message}` }],
              isError: true,
            };
          }
        }
        ```

---

## Step 4.7: Wire tools into the loop

**Goal:** Update `runLoop` to execute tool calls when present.

**Requirements:**
Update `runLoop` in `src/loop.ts`:

- After getting an assistant response, check `hasToolCalls()`
- If yes:
  - For each tool call, find the matching registered tool by name
  - Validate arguments (you'll define required fields per tool in the next step)
  - Execute with `executeToolCall`
  - Create tool result messages and append them to context
  - Emit events for each tool result
  - Loop back — call the LLM again with results appended
- If no tool calls: check `shouldStopAfterTurn()` and exit if true

**Hints:**
- The key structural change: your loop now has an inner loop (process tool calls from one response) inside the outer loop (keep calling the LLM until done). This is the "agent loop" pattern.
- For this milestone, execute tools sequentially (one at a time).
- After executing tools and appending results to context, you need to call the LLM again — the message array has changed.

??? example "Reference pattern"
    === "loop.ts — update runLoop (tool execution branch)"
        ```typescript
        // ... inside the while(true) loop, after getting the assistant message ...

        // Add to conversation
        messages.push(assistantMessage);
        lastMessage = assistantMessage;

        // Emit message events
        await onEvent({ type: "message_start", message: assistantMessage });
        await onEvent({ type: "message_update", message: assistantMessage });
        await onEvent({ type: "message_end", message: assistantMessage });

        // Check for tool calls
        if (hasToolCalls(assistantMessage)) {
          const toolCalls = getToolCalls(assistantMessage);

          for (const toolCall of toolCalls) {
            // Find the registered tool
            const tool = config.tools.find(t => t.name === toolCall.name);
            if (!tool) {
              // Unknown tool — create an error result
              const errorResult: ToolResult = {
                content: [{ type: "text", text: `Unknown tool: ${toolCall.name}` }],
                isError: true,
              };
              const resultMsg = createToolResultMessage(errorResult, toolCall);
              messages.push(resultMsg);
              await onEvent({ type: "message_start", message: resultMsg });
              await onEvent({ type: "message_end", message: resultMsg });
              continue;
            }

            // For now, skip validation — we'll add it in the next step
            // TODO: validateToolArguments(tool.name, toolCall.arguments, tool.requiredFields)

            // Execute
            const result = await executeToolCall(toolCall, tool, toolCall.arguments);
            const resultMsg = createToolResultMessage(result, toolCall);
            messages.push(resultMsg);
            await onEvent({ type: "message_start", message: resultMsg });
            await onEvent({ type: "message_end", message: resultMsg });
          }

          // Continue the loop — call LLM again with tool results
          await onEvent({ type: "turn_end", message: assistantMessage });
          continue;
        }

        // No tool calls — we're done with this turn
        await onEvent({ type: "turn_end", message: assistantMessage });

        // Check if we should stop
        if (shouldStopAfterTurn?.(assistantMessage) ?? true) {
          break;
        }
        ```

---

## Step 4.8: Create a simple read_file tool

**Goal:** Add a concrete tool so you can test the full flow end-to-end.

**Requirements:**
Create in `src/tools/read-file.ts`:

- A `readFileTool` that reads a file from disk:
  - name: `"read_file"`
  - description: `"Read the contents of a file at the given path."`
  - requiredFields: `["path"]`
  - execute: uses `fs.readFile()` to read the file and returns the content

- Register this tool in your CLI before calling `prompt()`:
  ```typescript
  agent.registerTool(readFileTool);
  ```

**Hints:**
- The execute function should return `{ content: [{ type: "text", text: fileContent }], isError: false }`.
- Handle errors gracefully — if the file doesn't exist, return `{ isError: true, content: [...] }` rather than throwing.
- Use `fs/promises` for async file operations.

??? example "Reference pattern"
    === "tools/read-file.ts"
        ```typescript
        import fs from "fs/promises";
        import type { Tool, ToolResult, TextContent } from "../types.js";

        export const readFileTool: Tool = {
          name: "read_file",
          description: "Read the contents of a file at the given path.",
          requiredFields: ["path"],
          execute: async (toolCallId, args) => {
            try {
              const path = args.path as string;
              const content = await fs.readFile(path, "utf-8");
              return {
                content: [{ type: "text", text: content }],
                isError: false,
              };
            } catch (error) {
              const message = error instanceof Error ? error.message : String(error);
              return {
                content: [{ type: "text", text: `Failed to read file: ${message}` }],
                isError: true,
              };
            }
          },
        };
        ```

---

## Step 4.9: Wire the CLI to use tools

**Goal:** Update `cli.ts` to register the read_file tool and test the full flow.

**Requirements:**
- Import `readFileTool` from `./tools/read-file.js`
- Register it with the agent before calling `prompt()`
- Update the event logger to show tool calls and results:
  - When a tool result appears, print `[TOOL RESULT: toolName - isError]`
  - Print the result content

**Hints:**
- The LLM mock response in Milestone 3 won't generate tool calls naturally. You have two options:
  1. Update the mock to occasionally return a tool call (for testing)
  2. Wait until Milestone 5 (real LLM) where the model will naturally trigger tool calls
- For now, you can test tool detection by manually creating a tool call in a test.

??? example "Reference pattern"
    === "cli.ts — register tools"
        ```typescript
        import { readFileTool } from "./tools/read-file.js";

        async function main() {
          try {
            const config = loadConfig();
            const agent = new Agent({
              systemPrompt: config.systemPrompt,
              model: config.model,
            });

            // Register tools
            agent.registerTool(readFileTool);

            // ... event logger and prompt call ...
            await agent.prompt("Read the README.md file", onEvent);
            process.exit(0);
          } catch (error) {
            // ... error handling ...
          }
        }
        ```

---

## Final Check for This Milestone

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
