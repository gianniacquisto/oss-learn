# :material-flag: Done Checklist — Milestone 5: Real LLM Calls

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:material-home: Back to Course Overview](../../README.md){: .md-button .md-button--primary }

---

**Congratulations!** If you made it here, you've built a working AI agent from scratch. Let's verify everything works.

## Verification Steps

1. **Compile:** Run `npm run build` and see **no TypeScript errors**
2. **Basic streaming:** Set a valid API key and run with a simple prompt:
   ```bash
   AGENT_API_KEY=sk-your-key-here npm start
   ```
   Verify text appears **incrementally** (token by token, not all at once after a delay)
3. **Multi-turn memory:** Send two prompts in a row and verify the second response references the first:
   ```typescript
   await agent.prompt("What is TypeScript?", onEvent);
   await agent.prompt("How does it compare to JavaScript?", onEvent);
   ```
4. **Tool usage:** Prompt with something that triggers the `read_file` tool and verify:
   - The LLM decides to call the tool
   - Your code executes it and returns the file contents
   - The LLM generates a response based on the file content
5. **Error handling:** Set an invalid API key and verify you get a clear error message (not a crash with a stack trace)

---

## What Should Work

??? success "Expected behavior"
    - Real API calls to OpenAI with proper authentication via your API key
    - Streaming text appears in real time as tokens are generated (not all at once)
    - Event stream emits `message_update` per token during streaming, enabling real-time display
    - Multi-turn conversations work — the agent remembers previous messages and references them
    - Tool calls from the real LLM trigger your tool execution pipeline (detect → validate → execute → feed back)
    - Invalid API key produces a clear error: `"Error: Error code: 401 — Invalid API key"` (not a raw stack trace)
    - The agent handles missing files gracefully (tool returns `{ isError: true }` instead of crashing)

## What Should NOT Work (Yet)

??? failure "Not expected yet"
    - Multiple provider support — you only implemented OpenAI (Anthropic would need a new converter and stream function)
    - Context window management — messages aren't pruned when the conversation gets too long for the model's context limit
    - Session persistence — conversations don't survive process restarts (no saving/loading state)
    - Abort/cancel — you can't stop a streaming response mid-generation
    - Rate limiting — if you exceed OpenAI's API limits, you'll get rate limit errors

## Troubleshooting

??? bug "Streaming text appears all at once instead of incrementally"
    This usually means the stream isn't being consumed properly. Check:
    1. You're using `for await (const event of stream)` — not collecting all events first then processing them
    2. The `streamChat` function uses `yield` for each token — not buffering them in an array
    3. The CLI uses `process.stdout.write()` for `message_update` events — not `console.log()` (which adds newlines)
    4. You're awaiting the stream iteration — if you don't await, events fire but the process might exit before they're printed

??? bug "I get an authentication error (401)"
    - Double-check your API key starts with `sk-` and is valid for your OpenAI account
    - Make sure you have credits/balance available in your [OpenAI dashboard](https://platform.openai.com/account/usage)
    - Verify the key is being passed correctly: `new OpenAI({ apiKey })` should receive the full key string

??? bug "The LLM doesn't call tools even when I ask it to"
    - Make sure tools are registered **before** calling `prompt()`: `agent.registerTool(readFileTool);`
    - Check that tool definitions are being sent to OpenAI (the `openaiTools` array in `streamChat` must not be empty)
    - Try a more explicit prompt: `"Use the read_file tool to read package.json"`
    - The system prompt might need to mention tools: `"You have access to tools. Use them when helpful."`

??? bug "TypeScript errors about OpenAI types"
    - Make sure `openai` is installed: `npm install openai`
    - Check that your imports use the correct paths:
      ```typescript
      import OpenAI from "openai";
      import type { ChatCompletionMessageParam } from "openai/resources/chat/completions";
      ```
    - If types changed in a newer SDK version, you may need to update your import paths — check the [OpenAI SDK docs](https://github.com/openai/openai-node)

??? bug "Tool arguments are malformed or empty"
    - OpenAI sends tool call arguments as a JSON string. Make sure you're parsing it: `JSON.parse(tc.arguments)`
    - If the argument string is incomplete (streaming might split it across chunks), make sure you've accumulated all chunks before parsing
    - Check that your `requiredFields` array matches what OpenAI expects — if a field is missing, validation should catch it

---

## What You've Built

You now have a complete AI agent harness that:

- ✅ Loads configuration from files and environment variables
- ✅ Manages conversation state with type-safe message types
- ✅ Runs an agent loop that calls the LLM, processes responses, and repeats
- ✅ Executes tools (like file reading) when the LLM requests them
- ✅ Streams responses token by token for real-time display
- ✅ Emits lifecycle events for logging, UI updates, or analytics
- ✅ Handles errors gracefully at every layer

This is the foundation of every production agent system. From here, you could add:
- More tools (web search, code execution, database queries)
- Additional providers (Anthropic, local models via Ollama)
- A web UI instead of a CLI
- Conversation persistence (save/load sessions)
- Context window management (summarize old messages)

**You built this. You understand how it works. That's the point.**
