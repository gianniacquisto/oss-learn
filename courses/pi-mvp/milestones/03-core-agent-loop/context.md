# Context for Milestone 3: Core Agent Loop

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

But the real version has to handle many edge cases: streaming responses, abort signals, error states, tool call validation, and the distinction between "the model is thinking" vs. "the model is done."

## How This Fits in the Bigger Picture

This milestone builds the loop that Milestone 4 (tool execution) extends with actual tool handling. The loop you write here will be nearly identical to pi.dev's `runLoop` function — it just won't have the tool execution part yet. When you add tools in Milestone 4, you'll insert a tool processing step between "get response" and "check if done."

## Key Decisions You'll Make

- **Single-turn vs. multi-turn:** Should your loop handle multiple LLM calls (tool calls → results → more LLM calls) or just one? For this milestone, implement the multi-turn structure but with a simple exit condition (stop after first response without tool calls).
- **Abort handling:** When should you check the abort signal? At each iteration of the loop is safest — it means cancellation can happen between any two operations.
- **Error encoding:** Should errors be thrown or encoded in the message? Pi encodes them in the `AssistantMessage` with `stopReason: "error"` and an optional `errorMessage`. This lets the agent continue processing even after a failure rather than crashing entirely.
