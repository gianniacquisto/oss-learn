# :material-flag: Done Checklist — Milestone 3: The Agent Loop

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: Tool Execution →](../04-tool-execution/README.md){: .md-button .md-button--primary }

---

## Verification Steps

1. **Compile:** Run `npm run build` and see **no TypeScript errors**
2. **Run:** Execute `AGENT_API_KEY=fake npm start` and verify it prints events in this exact order:
   ```
   [Agent started]
     [Turn started]
       [Message generation started]
       (mock response text)
       [Message generation ended]
     [Turn ended]
   [Agent finished — total messages: 2 ]
   ```
3. **Check state:** After completion, `agent.state.isStreaming` should be `false` and `agent.state.messages` should contain exactly 2 messages (1 user + 1 assistant)
4. **Test the guard:** Temporarily modify your CLI to call `agent.prompt()` twice without awaiting the first — verify it throws `"Agent is already processing a prompt."` on the second call

---

## What Should Work

??? success "Expected behavior"
    - The loop emits all seven event types in the correct order for a single-turn run
    - Events contain the right data (message content in `message_update`, full transcript in `agent_end`)
    - `agent.state.messages` grows from 0 → 1 (user message) → 2 (assistant response) after one prompt
    - The mock response echoes back the user's input
    - `agent.state.isStreaming` is `false` after `prompt()` resolves (even if an error occurred)
    - Calling `prompt()` while the agent is busy throws a clear error

## What Should NOT Work (Yet)

??? failure "Not expected yet"
    - Multi-turn conversations — the loop exits after one turn (tools trigger multi-turn in Milestone 4)
    - Tool call detection and execution — no tool infrastructure in the loop yet
    - Real LLM API calls — still using `mockLLMCall()` (replaced in Milestone 5)
    - Streaming text — you get the full response at once, not token-by-token

## Troubleshooting

??? bug "Events aren't emitted in the expected order"
    Make sure every `onEvent()` call is awaited (or at least called in sequence). The mock LLM is fast but still async — if events fire out of order, check that you're calling `onEvent()` for each event type in the loop body in the correct sequence.

??? bug "agent.state.isStreaming stays true after prompt() completes"
    Check your `try/finally` block in the `prompt()` method. The `finally` block must set `isStreaming = false` regardless of whether `runLoop` succeeded or threw:
    ```typescript
    this._state.isStreaming = true;
    try {
      await runLoop(...);
    } finally {
      this._state.isStreaming = false;  // ← must be here, not in try or catch
    }
    ```

??? bug "TypeScript complains about the mockLLMCall function's return type"
    Make sure `mockLLMCall` returns an `AssistantMessage` with **all required fields**:
    - `role: "assistant"`
    - `content: TextContent[]` (array, not string)
    - `model: string`
    - `provider: string`
    - `stopReason: "stop" | "toolCalls" | "error" | "aborted"`
    - `timestamp: number`

??? bug "The concurrency guard doesn't fire on double prompt() calls"
    Make sure you're calling both prompts without awaiting the first:
    ```typescript
    agent.prompt("first");  // no await — starts processing
    agent.prompt("second"); // should throw because isStreaming is true
    ```
    If you `await` the first call, the second one runs after the first completes, so `isStreaming` is already `false`.
