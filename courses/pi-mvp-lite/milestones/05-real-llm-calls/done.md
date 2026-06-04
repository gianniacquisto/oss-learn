# Done Checklist — Milestone 4: Real LLM Calls

[:material-arrow-left: Back to Steps](steps.md){: .md-button }&nbsp;&nbsp;[:material-home: Back to Course Overview](../../README.md){: .md-button .md-button--primary }

---

If everything works, you should be able to do the following:

## Verification Steps

1. Run `npm run build` and see **no TypeScript errors**
2. Set a valid API key (`AGENT_API_KEY=sk-...`) and run `npm start` with a simple prompt like "Hello" — verify you get streaming text output from a real LLM
3. Try a follow-up: ask "What did I just say?" — verify the agent remembers your previous message
4. Verify the event stream emits all lifecycle events in order

## What Should Work

- Real API calls to OpenAI with proper authentication
- Streaming text appears in real time as tokens are received (not all at once at the end)
- Event stream emits all lifecycle events (`agent_start`, `turn_start`, `message_update`, etc.) in order
- Error handling works: invalid API key produces a clear error message, not a crash

## What Should NOT Work (Yet)

- Multiple provider support — you only implemented OpenAI
- Tool execution — tools come in the full course (pi-mvp)
- Context window management — messages aren't pruned when they get too long
- Session persistence — conversations don't survive process restarts

## If Something Is Broken

### "Streaming text appears all at once instead of incrementally"
- Check that you're iterating over the stream with `for await` and emitting events *during* iteration, not after. If you collect all chunks first and then emit, it's no longer streaming.

### "I get an authentication error"
- Double-check your API key. It should start with `sk-` and be valid for your OpenAI account.
- Make sure you have credits available in your OpenAI account.

### "The response seems wrong or unhelpful"
- Check your system prompt in `.agentrc.json` or `config.ts`. A basic prompt like "You are a helpful assistant" is a good start.
- Try a more specific prompt: "You are a helpful coding assistant. Be concise and accurate."

### "TypeScript errors about OpenAI types"
- Make sure your `convertToOpenAIMessages` function returns the exact type expected by the SDK (`ChatCompletionMessageParam[]`).
