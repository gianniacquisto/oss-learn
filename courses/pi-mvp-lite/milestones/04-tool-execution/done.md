# Done Checklist — Milestone 4: Tool Execution

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: Real LLM Calls →](../05-real-llm-calls/README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors**
2. Verify that `hasToolCalls()` correctly identifies messages with tool calls
3. Verify that `validateToolArguments()` returns `true` for valid args and an error message for missing fields
4. Verify that `executeToolCall()` catches errors and returns `{ isError: true }`
5. Register the `read_file` tool and verify it appears in `agent.state.tools`

## What Should Work

- Tool calls are correctly detected in assistant messages (content blocks with `type === "toolCall"`)
- Arguments are validated against required fields before execution
- Failed validation produces a clear error message
- Successful tool execution returns content that gets formatted into a `ToolResultMessage`
- Tool result messages are appended to context
- The loop continues after tool execution (multi-turn flow works)
- Events include `message_start`/`message_end` for each tool result

## What Should NOT Work (Yet)

- Real LLM calls — still using a mock response (that comes in Milestone 5)
- Parallel tool execution — tools execute sequentially
- Tool call ID generation — for now, use a simple counter or `crypto.randomUUID()`
- Multiple tools — you only have `read_file`, add more as you like

## If Something Is Broken

### "The loop doesn't continue after executing tools"
- Check that after processing all tool calls, you're calling back into the LLM with the updated context (messages array now includes tool results). The `continue` statement in the while loop should handle this.

### "TypeScript complains about ToolCallBlock"
- Make sure you added `ToolCallBlock` to the `AssistantMessage.content` union type. The content should be `(TextContent | ToolCallBlock)[]`.

### "Tool execution throws an unhandled error"
- Check that your try/catch in `executeToolCall` wraps the actual tool call. If a tool throws before returning, it should be caught and converted to `{ isError: true }`.

### "read_file tool can't find the file"
- Make sure you're running the CLI from the right directory. The `path` argument is relative to the current working directory.
- Try using an absolute path like `/home/youruser/README.md` for testing.
