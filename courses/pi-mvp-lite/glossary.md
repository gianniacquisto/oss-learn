---
description: Terms and concepts you'll encounter throughout this course.
---

# :material-book-alphabet: Glossary

Terms and concepts you'll encounter throughout this course. Click any term to expand for details and references.

---

## A

### Agent

??? detail "Agent"
    **Definition:** A program that can reason about a task, take actions (like calling tools), and learn from the results — all by coordinating with an LLM in a loop.

    **Why it matters here:** You're building one. The `Agent` class you create in Milestone 2 is the core of your harness — it manages the conversation, runs tools, and drives the loop.

    [:fontawesome-solid-link: Anthropic's Guide to Building Agents](https://www.anthropic.com/research/building-effective-agents) — practical guidance on agent design

### Agent Loop

??? detail "Agent Loop"
    **Definition:** The core control flow of an AI agent: prompt the model → receive a response → execute any tool calls → feed results back to the model → repeat until completion.

    **Why it matters here:** This is the central pattern you'll build in Milestone 3. Every agentic system, from simple chatbots to complex autonomous agents, implements some variation of this loop.

    [:fontawesome-solid-link: Lilian Weng's AI Agent Survey](https://lilianweng.github.io/posts/2023-06-23-agent/) — the definitive overview

---

## E

### Event

??? detail "Event"
    **Definition:** A notification that something happened in your program. Other parts of the code can "listen" for events and react when they occur.

    **Why it matters here:** Your agent emits events like `agent_start`, `message_update`, and `agent_end`. A listener (like your CLI) can react to these — printing text, updating a display, logging, etc.

    [:fontawesome-solid-link: Node.js EventEmitter](https://nodejs.org/api/events.html) — how Node.js implements events

---

## L

### LLM (Large Language Model)

??? detail "LLM"
    **Definition:** A type of AI model trained on massive amounts of text that can generate human-like responses. Examples include OpenAI's GPT models and Anthropic's Claude.

    **Why it matters here:** Your agent uses an LLM as its "brain." The LLM generates responses, decides when to call tools, and determines when a task is complete.

    [:fontawesome-solid-link: OpenAI API Docs](https://platform.openai.com/docs/api-reference/chat) — how to interact with LLMs via API

---

## M

### Message (in LLM context)

??? detail "Message"
    **Definition:** A single unit of conversation with a role (`user`, `assistant`, `system`, `toolResult`) and content (text, images, or tool calls). Messages form the conversation history that the LLM uses to generate responses.

    **Why it matters here:** Your agent manages messages internally. You'll define message types in Milestone 2 and learn how different message roles serve different purposes in the conversation.

    [:fontawesome-solid-link: OpenAI Messages API](https://platform.openai.com/docs/guides/text-generation/chat-completions-api) — the standard message format

---

## S

### State

??? detail "State"
    **Definition:** The current condition of a program — what data it's holding, what it's currently doing, what it knows. For an agent, state includes the conversation history, available tools, and whether it's processing a prompt.

    **Why it matters here:** Your `Agent` class stores state internally. You'll learn how to manage state safely — keeping it mutable internally but protected from external tampering.

---

## T

### Tool (in agent context)

??? detail "Tool"
    **Definition:** A function that an agent can call to take action in the world. A tool has a name, description (for the LLM to understand when to use it), parameters (what inputs it needs), and an execute function (the actual code).

    **Why it matters here:** This is what makes agents *agentic*. You'll build a tool system in Milestone 4 that lets your agent read files, run commands, and eventually interact with anything.

    [:fontawesome-solid-link: OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling) — how tools work with LLMs

### TypeScript

??? detail "TypeScript"
    **Definition:** A programming language that adds type safety to JavaScript. It lets you define the structure of your data (what fields an object has, what types of values are allowed) and catches errors at compile time instead of runtime.

    **Why it matters here:** You'll use TypeScript throughout this course. If you're new to it, the types will help you catch mistakes early — like forgetting to add a required field to a message.

    [:fontawesome-solid-link: TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/2/basic-types.html) — TypeScript basics

---

## API Key

??? detail "API Key"
    **Definition:** A secret string that authenticates your program with a service (like OpenAI). Think of it like a password — you need it to make API calls, and you should never share it publicly.

    **Why it matters here:** You'll need an OpenAI API key to make real LLM calls in Milestone 4. You'll store it in an environment variable or config file so your code can access it without hardcoding it.
