# Milestone 5: LLM Integration & Streaming

## What You'll Build

Real LLM API integration (OpenAI or Anthropic) with streaming support. The agent can now have a genuine multi-turn conversation — it sends messages to a real model, receives streamed responses token-by-token, and handles tool calls from actual AI-generated content. This is where your agent harness becomes functional.

## Why This Matters

Up until now, you've built the architecture but tested with mocks. Milestone 5 connects everything to reality: real API calls, real streaming, real tool calls from an actual model. It also introduces the `convertToLlm` abstraction — converting your internal message format into whatever the LLM provider expects. This is where you see whether your type system and message design hold up under real-world usage.

## Prerequisites for This Milestone

- Complete Milestones 1–4
- An API key for OpenAI or Anthropic (set `AGENT_API_KEY` environment variable)
- Understanding of how streaming works with LLM APIs (token-by-token delivery)
