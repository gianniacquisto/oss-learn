# :material-flag: Done Checklist — Milestone 2: Messages & Agent State

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:octicons-arrow-right-24: Next: The Agent Loop →](../03-agent-loop/README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. **Compile:** Run `npm run build` and see **no TypeScript errors** (warnings about unused variables are fine)
2. **Run:** Set an API key and run `AGENT_API_KEY=fake npm start`. You should see a JSON object with your agent's state printed to stdout:
   - `systemPrompt` — your default prompt string
   - `model` — `"gpt-4o"` (or whatever you set in config)
   - `tools` — an empty array `[]`
   - `messages` — an empty array `[]`
   - `isStreaming` — `false`
3. **Test reset:** Modify your CLI temporarily to call `agent.reset()` and print state again — `messages` should still be `[]`, but `systemPrompt` and `model` should be unchanged
4. **Test tool setter:** Modify your CLI to do `agent.tools = [...]` with a mock tool object, then print state — the tools array should contain your tool

---

## What Should Work

??? success "Expected behavior"
    - All message types (`UserMessage`, `AssistantMessage`, `ToolResultMessage`, `SystemMessage`) compile without errors
    - The `AgentMessage` union type lets TypeScript narrow based on `role`
    - `agent.state` returns an object with all six fields populated
    - `agent.state.messages` is readonly — trying to assign to it (`agent.state.messages = []`) produces a TypeScript error
    - `agent.reset()` clears `messages` and `errorMessage` but keeps `systemPrompt`, `model`, and `tools`
    - `agent.tools = [...]` works (setter is defined)

## What Should NOT Work (Yet)

??? failure "Not expected yet"
    - Sending prompts to the agent — there's no `prompt()` method yet (that's Milestone 3)
    - Executing tool calls — tools are registered but the loop doesn't run them (Milestone 4)
    - The CLI reading input from stdin — it just prints state and exits

## Troubleshooting

??? bug "TypeScript complains about readonly properties when I try `agent.state.messages.push(...)`"
    **This is correct behavior!** The `AgentState` interface marks all fields as `readonly`. You can't mutate the agent's state through the public getter — that's the whole point of encapsulation. Only the Agent's internal methods can modify `_state`.

??? bug "JSON output shows `undefined` for some fields"
    Check your constructor in `agent.ts`. Every field in `_state` must be initialized:
    ```typescript
    this._state = {
      systemPrompt: options?.systemPrompt ?? getDefaultSystemPrompt(),  // ← not undefined
      model: options?.model ?? "gpt-4o",                                // ← not undefined
      tools: options?.tools ?? [],                                      // ← not undefined
      messages: [],                                                     // ← not undefined
      isStreaming: false,                                               // ← not undefined
    };
    ```
    If any line is missing, that field will be `undefined` in the output.

??? bug "Cannot find module './agent.js'"
    - Make sure you ran `npm run build` first — the CLI runs from `dist/`, which contains compiled `.js` files
    - Check that `agent.ts` exists in `src/` and compiles without errors
    - Verify the import uses `.js` extension: `import { Agent } from "./agent.js"`

??? bug "TypeScript says 'role' can't be used to narrow the type"
    Make sure each message interface uses a **string literal type** for role, not plain `string`:
    ```typescript
    // ✅ Correct — TypeScript knows the exact value
    role: "user";

    // ❌ Wrong — TypeScript treats this as any string, can't narrow
    role: string;
    ```
