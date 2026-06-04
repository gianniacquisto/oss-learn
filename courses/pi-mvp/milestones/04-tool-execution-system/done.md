# Done Checklist — Milestone 4: Tool Execution System

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: LLM Integration →](../05-llm-integration/README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors**
2. Run `npm start` with a prompt that would trigger a tool call (e.g., "Read the file at ./README.md") — verify it:
   - Detects the tool call in the assistant response
   - Validates arguments against the schema
   - Executes the tool and captures the result
   - Appends the tool result to context
   - Calls the LLM again with the results
3. Test error handling: try calling `read_file` with a non-existent path — verify it returns an error message instead of crashing

## What Should Work

- Tool calls are correctly detected in assistant messages (content blocks with `type === "toolCall"`)
- Arguments are validated against Zod schemas before execution
- Failed validation produces a clear error: `"Tool 'read_file' validation failed: path is required"`
- Successful tool execution returns content that gets formatted into a `ToolResultMessage`
- Tool result messages are appended to context and sent back to the LLM on the next turn
- The loop continues after tool execution (multi-turn flow works)
- Events include `tool_execution_start`/`message_start`/`message_end` for each tool result

## What Should NOT Work (Yet)

- Parallel tool execution — tools execute sequentially in this milestone
- Streaming partial results from tools — full streaming comes in Milestone 5
- Dynamic tool registration at runtime via MCP — you're registering tools statically in the CLI
- Tool call ID generation — for now, use a simple counter or `crypto.randomUUID()`

## If Something Is Broken

### "The loop doesn't continue after executing tools"
- Check that after processing all tool calls, you're calling back into the LLM with the updated context (messages array now includes tool results). The recursive call to `runLoop` should handle this.

### "TypeScript complains about Zod schema types"
- Make sure your `Tool.parameters` type matches what Zod produces. Use `z.infer<typeof mySchema>` for the execute function's args type, and `z.ZodType<any>` (or `z.AnyZodObject`) for the parameters field.

### "Tool execution throws an unhandled error"
- Check that your try/catch in `executeToolCall` wraps the actual tool call. If a tool throws before returning, it should be caught and converted to `{ isError: true }`.
