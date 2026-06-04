# Done Checklist — Milestone 5: LLM Integration & Streaming

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:material-home: Back to Course Overview](../../README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors**
2. Set a valid API key (`AGENT_API_KEY=sk-...`) and run `npm start` with a simple prompt like "Hello" — verify you get streaming text output from a real LLM
3. Prompt with something that should trigger a tool call: "Read the file at ./README.md" — verify it:
   - Streams text to the terminal
   - Detects and executes the `read_file` tool call
   - Shows the file contents in the next response turn
4. Try a multi-turn conversation: after the agent reads the file, ask a follow-up question that requires understanding the file content — verify it uses the tool result from context

## What Should Work

- Real API calls to your chosen provider (OpenAI or Anthropic) with proper authentication
- Streaming text appears in real time as tokens are received (not all at once at the end)
- Tool calls are correctly detected, validated, and executed during streaming
- Tool results are appended to context and influence subsequent LLM responses
- Event stream emits all lifecycle events (`agent_start`, `turn_start`, `message_update`, etc.) in order
- Error handling works: invalid API key produces a clear error message, not a crash

## What Should NOT Work (Yet)

- Multiple provider support — you implemented one provider; the other follows the same pattern
- Parallel tool execution — tools still execute sequentially
- Context window management / token counting — messages aren't pruned when they get too long
- Session persistence — conversations don't survive process restarts
- CLI argument parsing beyond a simple prompt string

## If Something Is Broken

### "Streaming text appears all at once instead of incrementally"
- Check that you're iterating over the stream with `for await` and emitting events *during* iteration, not after. If you collect all chunks first and then emit, it's no longer streaming.

### "Tool calls aren't detected or have empty arguments"
- Tool call JSON arrives in pieces across multiple stream chunks. Make sure you're accumulating the `argumentsDelta` strings before parsing them at the end. An incomplete JSON parse will fail silently.

### "The model doesn't trigger tool calls even when prompted"
- Your system prompt might not encourage tool use. Add: "You have access to tools. Use them whenever they would help answer the user's question."
- Check that `convertToolsToOpenAIFormat` is producing valid tool definitions — invalid schemas cause the provider to silently ignore them.

### "TypeScript errors about OpenAI/Anthropic types"
- Make sure your `convertToLlm` function returns the exact type expected by the SDK (`ChatCompletionMessageParam[]` for OpenAI, `ContentBlockParam[]` for Anthropic). Type mismatches here are common and easy to fix with IDE autocomplete.
