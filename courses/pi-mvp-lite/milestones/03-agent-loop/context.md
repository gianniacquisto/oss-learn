# Context for Milestone 3: The Agent Loop

## Background

The agent loop is the central control flow of any agentic system. At its simplest, it looks like this:

```javascript
while (hasMoreWork) {
  response = callLLM(context);
  if (response.hasToolCalls()) {
    executeTools(response.toolCalls);
    addResultsToContext();
  } else {
    break; // done
  }
}
```

But the real version has to handle many edge cases: abort signals, error states, and the distinction between "the model is thinking" vs. "the model is done."

## How This Fits in the Bigger Picture

This milestone builds the loop that Milestone 4 (real LLM calls) extends with actual API integration. The loop you write here will use a mock LLM — a fake response that lets you test the structure without needing API keys. When you get to Milestone 4, you'll replace the mock with a real API call.

## Key Decisions You'll Make

- **Single-turn vs. multi-turn:** Should your loop handle multiple LLM calls (tool calls → results → more LLM calls) or just one? For this milestone, implement the multi-turn structure but with a simple exit condition.
- **Event system:** How should the agent notify other code about what's happening? You'll use a simple callback approach — pass a function to the agent that gets called with events. This is simpler than a full EventEmitter and good enough for a CLI.
- **Error encoding:** Should errors be thrown or encoded in the message? We'll encode them in the `AssistantMessage` with `stopReason: "error"` — this lets the agent continue processing even after a failure.

## What "Good" Looks Like

By the end of this milestone:
- `runLoop()` in `loop.ts` runs a loop that calls a mock LLM
- The `Agent` class has a `prompt()` method that kicks off the loop
- Events are emitted as the loop progresses (you can log them)
- The loop correctly detects when to stop (when the mock response has no tool calls)
