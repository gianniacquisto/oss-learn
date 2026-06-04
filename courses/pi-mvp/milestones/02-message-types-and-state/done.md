# Done Checklist — Milestone 2: Message Types & Agent State

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: Core Agent Loop →](../03-core-agent-loop/README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors**
2. Run `npm start` and verify it prints a JSON object with your agent's state (systemPrompt, model, tools array, messages array, isStreaming = false)
3. Verify that the `AgentState` interface has all required readonly fields: `systemPrompt`, `model`, `tools`, `messages`, `isStreaming`, `errorMessage?`
4. Test that assigning new tools works: modify your CLI to create an agent with no tools, then assign a tool array and print state again

## What Should Work

- All message types (`UserMessage`, `AssistantMessage`, `ToolResultMessage`, `SystemMessage`) are properly defined as discriminated unions in `types.ts`
- The `Agent` class has private mutable state and a readonly public getter
- `agent.state.isStreaming` is `false` when no run is active
- `agent.state.messages` starts as an empty array
- `agent.reset()` clears messages but preserves systemPrompt and model
- TypeScript enforces that callers can't mutate `state.messages` directly (readonly)

## What Should NOT Work (Yet)

- Sending prompts to the agent — no loop yet, so no LLM interaction
- Executing tool calls — tools are registered but not wired into any execution flow
- Streaming responses or event emission — those come in Milestones 3 and 5

## If Something Is Broken

### "TypeScript complains about readonly properties"
- You can't assign to `agent.state.messages` directly because it's readonly. Use `agent.tools = [...]` for tools (you added a setter), but messages are managed internally by the agent only. This is intentional — the agent owns its transcript.

### "JSON output shows undefined fields"
- Check that your private `_state` initializes all fields in the constructor. Missing initialization leads to `undefined` in the JSON output.

### "Tool[] type causes issues with TSchema"
- If you used `unknown` as a placeholder for `TParameters`, make sure the Tool interface still compiles. You can temporarily use `any` if TypeScript is being overly strict, but note that defeats the purpose of generics.
