# Context for Milestone 4: Real LLM Calls

## Background

Connecting to an LLM API is straightforward — you send a request, get a response. But doing it *well* requires handling several complexities:

1. **Streaming:** Responses come token-by-token. You need to accumulate partial messages and emit updates as they arrive.
2. **Message format conversion:** Your internal `AgentMessage[]` format is different from what OpenAI expects. You need a `convertToOpenAIMessages` function that transforms your messages into the OpenAI format.
3. **Error handling at the API level:** Network failures, rate limits, invalid requests — these need to be caught and converted into proper error messages within your agent's event stream.

## How This Fits in the Bigger Picture

This milestone replaces your mock LLM call with a real API integration. The loop structure from Milestone 3 doesn't change — only how you get the response changes. The key insight: by keeping `convertToOpenAIMessages` as a separate function, you can swap providers later without touching the loop logic.

## Key Decisions You'll Make

- **Streaming vs. non-streaming:** You'll implement streaming — it's what users expect and it's not that much harder than non-streaming if you design your event system correctly.
- **How to handle tool definitions for the API:** OpenAI uses `function` objects for tools. Your Zod schemas need to be convertible to JSON Schema.

## What "Good" Looks Like

By the end of this milestone:
- `convertToOpenAIMessages` transforms your messages into OpenAI's format
- `streamChat` makes a real streaming API call to OpenAI
- The loop consumes the stream and emits events as tokens arrive
- Your CLI renders streaming text — you see text appearing in real time
- You can have a real conversation with GPT through your agent
