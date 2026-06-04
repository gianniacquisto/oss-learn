# Done Checklist — Milestone 3: The Agent Loop

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: Tool Execution →](../04-tool-execution/README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors**
2. Run `npm start` and verify it prints event logs in this order:
   - `[Agent started]`
   - `[Turn started]`
   - `[Message started]`
   - The mock response text
   - `[Message ended]`
   - `[Turn ended]`
   - `[Agent finished]`
3. Verify that after `prompt()` completes, `agent.state.isStreaming` is `false` and `agent.state.messages` contains both your user message and the assistant response
4. Test the guard: call `prompt()` twice in quick succession without awaiting — verify it throws "Agent is already processing" on the second call

## What Should Work

- The loop emits all events in the correct order for a single-turn run
- Events contain the right data (message content, etc.)
- `agent.state.messages` has exactly 2 messages after one prompt: user + assistant
- The mock response is correctly converted into an `AssistantMessage` with proper fields

## What Should NOT Work (Yet)

- Multi-turn conversations with tool calls — the loop exits after one turn (tools come in Milestone 4)
- Real LLM API calls — still using a mock response
- Streaming responses — you're getting the full response at once

## If Something Is Broken

### "Events aren't emitted in order"
- Make sure you're awaiting each `onEvent()` call. If you don't await it, events might fire out of order.

### "agent.state.isStreaming stays true after prompt() completes"
- Check your try/finally block in the `prompt()` method. The finally block must set `isStreaming = false` regardless of whether runLoop succeeded or threw an error.

### "TypeScript complains about the mockLLMCall function"
- Make sure the function returns an `AssistantMessage` with all required fields: `role`, `content`, `model`, `provider`, `stopReason`, and `timestamp`.
