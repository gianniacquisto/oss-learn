---
description: Build your own minimal agent harness — the core engine that powers pi.dev
---

# :material-robot: pi-mvp — Build Your Own Agent Harness

Build your own minimal agent harness — the core engine that powers [pi.dev](https://github.com/earendil-works/pi). This is not a chatbot tutorial; it's an exercise in designing the architecture of an AI agent system from scratch.

---

## What You'll Build

A working agent harness that can:

- :material-play-circle: Accept user prompts and stream responses back
- :fontawesome-solid-wand-magic-sparkles:{ } Call an LLM (OpenAI or Anthropic) to generate assistant messages
- :material-tools: Execute tool calls the model requests (read files, run commands, etc.)
- :material-refresh: Loop until the conversation is complete
- :material-signal: Emit lifecycle events for UI integration

You'll end up with something that can have a multi-turn conversation with an LLM and actually **do things** based on what it learns.

---

## Course Info

| | |
|---|---|
| **Difficulty** | :material-account-school: Intermediate |
| **Time Estimate** | :material-clock-outline: 8–12 hours |
| **Milestones** | :material-check-circle: 5 steps |
| **Language** | :simple-typescript: TypeScript |

---

## Prerequisites

Make sure you're comfortable with these before starting:

- **TypeScript basics** — types, interfaces, generics, modules
- **Node.js and npm** — running scripts, managing packages
- **Async/await and promises** — comfortable with async flows
- **Basic LLM API knowledge** — know what a chat completion request looks like

---

## What You'll Learn

??? example "Stateful Agent Design"
    How to design a stateful agent that manages conversation context across turns.

??? example "The Agent Loop Pattern"
    The core pattern: prompt → LLM call → tool execution → repeat.

??? example "Tool Registration & Execution"
    Register tools, execute them safely, and feed results back to the model — with type-safe schemas.

??? example "Event-Driven Architecture"
    Decouple the core engine from UI concerns using an event emission system.

??? example "Streaming Responses"
    Real-time token-by-token streaming for a live feel.

---

## Milestones

::: {.grid }

### :fontawesome-solid-circle-1:{ .lg .middle } Project Setup & Config

Set up your TypeScript project with a minimal CLI that prints "Hello agent."

[:octicons-arrow-right-24: Start](milestones/01-project-setup/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/01-project-setup/context.md){: .md-button }

---

### :fontawesome-solid-circle-2:{ .lg .middle } Message Types & State

Build a type-safe message system and an `Agent` class managing conversation context.

[:octicons-arrow-right-24: Start](milestones/02-message-types-and-state/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/02-message-types-and-state/context.md){: .md-button }

---

### :fontawesome-solid-circle-3:{ .lg .middle } Core Agent Loop

Implement the main loop: prompt the LLM, get a response, detect if it needs tools.

[:octicons-arrow-right-24: Start](milestones/03-core-agent-loop/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/03-core-agent-loop/context.md){: .md-button }

---

### :fontawesome-solid-circle-4:{ .lg .middle } Tool Execution System

Register tools, execute them safely, and feed results back to the model.

[:octicons-arrow-right-24: Start](milestones/04-tool-execution-system/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/04-tool-execution-system/context.md){: .md-button }

---

### :fontawesome-solid-circle-5:{ .lg .middle } LLM Integration & Streaming

Full multi-turn conversation with real API calls and event streaming.

[:octicons-arrow-right-24: Start](milestones/05-llm-integration/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/05-llm-integration/context.md){: .md-button }

:::

---

## Before You Start

1. Read the [:material-lightbulb-outline: Key Concepts](./key-concepts.md) to understand what you're learning
2. Skim the [:material-book-alphabet: Glossary](./glossary.md) for terms you might not know
3. Check the [:simple-web: Reading List](./reading-list.md) if you want deeper context

??? warning "Important"
    Each milestone builds on the last. You'll have something runnable at every step — **don't skip ahead**. The instructions give you requirements and hints, but **you write the code**. If you're stuck, re-read the context before looking for answers online.
