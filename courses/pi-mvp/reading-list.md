# Reading List

Curated resources organized by topic area. Each link is worth your time — no filler.

## Agent Architecture

### [AI Agents Survey — Lilian Weng](https://lilianweng.github.io/posts/2023-06-23-agent/)
The definitive overview of AI agent patterns. Covers ReAct, planning, memory, and tool use. Read this before Milestone 3 to understand the landscape your agent fits into.

### [Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
Practical guidance from the Anthropic team on structuring agentic systems. Focuses on what actually works in production vs. what looks good in demos.

## LLM APIs & Integration

### [OpenAI API Reference](https://platform.openai.com/docs/api-reference/chat)
You'll implement something similar to this. Skim the request/response shapes — you don't need to memorize them, just understand the structure.

### [Streaming Responses Best Practices](https://platform.openai.com/docs/guides/text-generation/streaming)
OpenAI's guide on streaming. Useful when you get to Milestone 5 and need to handle partial responses correctly.

## TypeScript Patterns

### [TypeScript Generics Handbook](https://www.typescriptlang.org/docs/handbook/2/generics.html)
Reference for the generic patterns you'll use in tool registration (`AgentTool<T>`). Come back here if you're unsure about type constraints.

### [TypeBox — Schema Validation for TypeScript](https://github.com/sinclairzx81/typebox)
Pi uses TypeBox (JSON Schema → TypeScript types) for tool parameter validation. If you'd prefer Zod or another validator, the concepts are the same — just swap the library.

## Async & Event Patterns

### [Async Iterators in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#the_async_iterator_and_async_iterable_protocols)
You'll use async iterables for your event stream. This MDN guide explains the protocol your implementation should follow.

### [AbortController / AbortSignal](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
For implementing cancellation in your agent loop. Essential reading before Milestone 3 when you add abort support.

## Deep Dives (Optional)

### [The Agent Loop — Simon Willison](https://simonwillison.net/2024/Sep/2/agent-loop/)
A practical breakdown of the agent loop pattern with code examples. Complements your Milestone 3 implementation.

### [Prompt Engineering Guide](https://www.promptingguide.ai/)
If you want to understand how system prompts shape agent behavior, this is a great reference. Relevant when you write the default system prompt in Milestone 2.
