# :material-flag: Done Checklist — Milestone 4: Tool Execution

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: Real LLM Calls →](../05-real-llm-calls/README.md){: .md-button .md-button--primary }

---

## Verification Steps

1. **Compile:** Run `npm run build` and see **no TypeScript errors**
2. **Test tool detection:** Verify `hasToolCalls()` returns `true` for a message with `ToolCallBlock` content and `false` for a text-only message
3. **Test validation:** Call `validateToolArguments("read_file", {}, ["path"])` — it should return an error string about the missing "path" field
4. **Test execution end-to-end:** Run `AGENT_API_KEY=fake npm start` with a prompt like `"Read the package.json file"` and verify:
   - The mock LLM returns a tool call for `read_file`
   - Your code executes the tool and reads `package.json`
   - A `ToolResultMessage` is appended to the conversation
   - The loop continues (second turn) and gives a final answer
5. **Test error handling:** Register the tool, then make the mock return a call for a non-existent tool name — verify it produces an "Unknown tool" error result instead of crashing

---

## What Should Work

??? success "Expected behavior"
    - `ToolCallBlock` is part of the `AssistantMessage.content` union type
    - `agent.registerTool(readFileTool)` adds the tool to `agent.state.tools`
    - Tool calls in assistant messages are detected by `hasToolCalls()` and extracted by `getToolCalls()`
    - Arguments are validated — missing required fields produce clear error messages
    - Successful tool execution returns content formatted as a `ToolResultMessage`
    - Failed tool execution returns `{ isError: true }` with a descriptive error message
    - The loop continues after tool execution (multi-turn: tool call → result → LLM again → final answer)
    - Events include `message_start`/`message_end` for each tool result

## What Should NOT Work (Yet)

??? failure "Not expected yet"
    - Real LLM calls — still using a mock response (that comes in Milestone 5)
    - Parallel tool execution — tools run one at a time, not simultaneously
    - Multiple tools beyond `read_file` — you have the infrastructure, but only one concrete tool
    - Streaming responses — the entire response arrives at once, not token by token

## Troubleshooting

??? bug "The loop doesn't continue after executing tools"
    Check that after processing all tool calls, your code calls `continue` (not `break`). The `continue` sends execution back to the top of `while(true)` for another LLM call. If you used `break`, the loop exits after tool execution and never calls the LLM again with the results.

??? bug "TypeScript complains about ToolCallBlock in content access"
    After adding `ToolCallBlock` to `AssistantMessage.content`, any code that accesses `content[i].text` directly will error because the item might be a tool call block. Fix by checking the type first:
    ```typescript
    // ❌ Before: assumed all content is TextContent
    message.content.map(c => c.text)

    // ✅ After: filter for text blocks only
    message.content.filter(c => c.type === "text").map(c => c.text)
    ```

??? bug "Tool execution throws an unhandled error"
    Check that `executeToolCall` wraps `tool.execute()` in a try/catch. If a tool throws (file not found, permission denied), the error should be caught and returned as `{ isError: true }`, not rethrown:
    ```typescript
    async function executeToolCall(...) {
      try {
        return await tool.execute(toolCallId, args);
      } catch (error) {
        // ← must catch here
        return { content: [{ type: "text", text: `Error: ${message}` }], isError: true };
      }
    }
    ```

??? bug "read_file tool can't find the file"
    The path argument is relative to the **current working directory** (where you run `npm start`). If you're in `my-agent/` and the mock passes `"package.json"`, it reads `my-agent/package.json`. Try an absolute path for testing: `"/home/youruser/my-agent/package.json"`.
