# Context for Milestone 5: LLM Integration & Streaming

## Background

Connecting to an LLM API is straightforward — you send a request, get a response. But doing it *well* requires handling several complexities:

1. **Streaming:** Responses come token-by-token. You need to accumulate partial messages and emit updates as they arrive.
2. **Provider differences:** OpenAI and Anthropic have different message formats, tool call schemas, and streaming protocols. Your `convertToLlm` abstraction is what keeps the loop provider-agnostic.
3. **Error handling at the API level:** Network failures, rate limits, invalid requests — these need to be caught and converted into proper error messages within your agent's event stream.

## How This Fits in the Bigger Picture

This milestone replaces your mock LLM call with real API integration. The loop structure from Milestone 3 doesn't change — only how you get the response changes. The key insight: by keeping `convertToLlm` as an abstraction, you can swap providers without touching the loop logic. This is exactly what pi.dev does — it supports OpenAI, Anthropic, Google, and others through provider-specific message converters.

## Key Decisions You'll Make

- **Which provider to implement first?** Start with one (OpenAI or Anthropic). The other follows the same pattern. Pi.dev supports both simultaneously.
- **Streaming vs. non-streaming:** Implement streaming from the start — it's what users expect and it's not that much harder than non-streaming if you design your event system correctly.
- **How to handle tool definitions for the API?** Each provider has a different format for describing tools (OpenAI uses `function` objects, Anthropic uses `tool` objects). Your `convertToLlm` function needs to transform your internal `Tool[]` into the provider's expected format.
