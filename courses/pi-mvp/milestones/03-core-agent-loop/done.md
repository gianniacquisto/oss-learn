# Done Checklist — Milestone 3: Core Agent Loop

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors**
2. Run `npm start` and verify it prints event logs in this order:
   - `[EVENT] agent_start`
   - `[EVENT] turn_start`
   - `[EVENT] message_start` (with the assistant message)
   - `[EVENT] message_end`
   - `[EVENT] turn_end`
   - `[EVENT] agent_end`
3. Verify that after `prompt()` completes, `agent.state.isStreaming` is `false` and `agent.state.messages` contains both your user message and the assistant response
4. Test the guard: call `prompt()` twice in quick succession without awaiting — verify it throws "Agent is already processing" on the second call

## What Should Work

- The loop emits all events in the correct order for a single-turn run
- Events contain the right data (message content, toolResults array, etc.)
- `agent.state.messages` has exactly 2 messages after one prompt: user + assistant
- The mock response is correctly converted into an `AssistantMessage` with proper fields (`model`, `provider`, `stopReason`)
- TypeScript types are enforced throughout — no `any` leaking through

## What Should NOT Work (Yet)

- Multi-turn conversations — the loop exits after one turn (tool call detection comes in Milestone 4)
- Real LLM API calls — still using a mock response
- Tool execution — tools are registered but not wired into the loop yet
- Streaming events — only `message_start` and `message_end` for now, no `message_update`

## If Something Is Broken

### "Events aren't emitted in order"
- Make sure you're awaiting each `emit()` call. If emit is async and you don't await it, events can fire out of order.

### "TypeScript complains about the convertToLlm function"
- Your mock implementation needs to handle all message roles. At minimum: filter out system messages if your provider doesn't support them, and flatten text content into a single string for providers that expect `content: string`.

### "agent.state.isStreaming stays true after prompt() completes"
- Check your try/finally block in the `prompt()` method. The finally block must set `isStreaming = false` regardless of whether runLoop succeeded or threw an error.
