---
description: Curated resources organized by topic area.
---

# :material-link-variant: Reading List

Curated resources organized by topic area. Each link is worth your time — no filler.

---

## Agent Architecture

::: {.grid }

### [:fontawesome-solid-book: AI Agents Survey — Lilian Weng](https://lilianweng.github.io/posts/2023-06-23-agent/)
{: .card }

The definitive overview of AI agent patterns. Covers ReAct, planning, memory, and tool use. Read this **before Milestone 3** to understand the landscape your agent fits into.

---

### [:fontawesome-solid-building: Building Effective Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
{: .card }

Practical guidance from the Anthropic team on structuring agentic systems. Focuses on what actually works in production vs. what looks good in demos.

:::

---

## LLM APIs & Integration

::: {.grid }

### [:fontawesome-solid-code: OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat/create)
{: .card }

You'll implement something similar to this. Skim the request/response shapes — you don't need to memorize them, just understand the structure.

---

### [:fontawesome-solid-code: Anthropic Messages API](https://docs.anthropic.com/en/docs/build-with-claude/message-generation)
{: .card }

Anthropic's equivalent API. Useful if you want to swap providers later — the patterns are similar.

:::

---

## TypeScript Basics (If You Need Them)

::: {.grid }

### [:simple-typescript: TypeScript in 5 Minutes](https://www.typescriptlang.org/docs/handbook/2/basic-types.html)
{: .card }

Quick crash course on TypeScript types. Skim this if you're new to TypeScript — you'll pick up the rest as you go.

---

### [:fontawesome-solid-book: TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/2/namespaces-and-modules.html)
{: .card }

Reference for modules, imports, and exports. Come back here if you're unsure about how TypeScript handles file imports.

:::

---

## Async Programming

::: {.grid }

### [:fontawesome-solid-book: JavaScript.info — Async/Await](https://javascript.info/async-await)
{: .card }

Clear explanation of async/await with examples. Essential reading before **Milestone 3** when you build the agent loop.

---

### [:octicons-stop-16: AbortController / AbortSignal](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
{: .card }

For implementing cancellation in your agent loop. You'll use this if you want to let users stop a running agent.

:::

---

## Deep Dives (Optional)

??? example "The Agent Loop — Simon Willison"
    A practical breakdown of the agent loop pattern with code examples. Complements your Milestone 3 implementation.

    [:fontawesome-solid-link: Read more](https://simonwillison.net/2024/Sep/2/agent-loop/)

??? example "Prompt Engineering Guide"
    If you want to understand how system prompts shape agent behavior, this is a great reference. Relevant when you write the default system prompt in Milestone 1.

    [:fontawesome-solid-link: Read more](https://www.promptingguide.ai/)
