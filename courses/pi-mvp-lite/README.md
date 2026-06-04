---
description: Build a minimal agent harness from scratch — a beginner-friendly introduction to AI agent architecture.
---

# :material-robot: pi-mvp-lite — Your First Agent Harness

Build a minimal agent harness that can talk to an LLM, use tools, and have a real conversation. This is a beginner-friendly introduction to how AI agents work — no prior agent experience needed.

---

## What You'll Build

A working command-line agent that can:

- :material-play-circle: Accept your messages and respond with text from an LLM
- :fontawesome-solid-wand-magic-sparkles:{ } Use tools to read files or run simple operations
- :material-refresh: Keep a conversation going — the agent remembers what you've discussed
- :material-signal: Show you what's happening at each step (it's like watching the agent "think")

You'll end up with something that can have a real conversation with OpenAI's GPT and actually **do things** based on what it learns.

---

## Course Info

| | |
|---|---|
| **Difficulty** | :material-account-school: Beginner |
| **Time Estimate** | 6–8 hours |
| **Milestones** | :material-check-circle: 5 steps |
| **Language** | :simple-typescript: TypeScript |

---

## Prerequisites

You should be comfortable with:

- **JavaScript/TypeScript basics** — variables, functions, objects, arrays
- **Running commands in a terminal** — `cd`, `ls`, `npm`
- **Basic file editing** — creating files, writing code in an editor

Don't worry if you haven't used TypeScript much — you'll learn what you need as you go.

---

## What You'll Learn

??? example "The Agent Loop"
    The core pattern behind every AI agent: ask the model → get a response → use tools → repeat.

??? example "Stateful Conversation"
    How agents remember what happened in a conversation, rather than treating each message as independent.

??? example "Tool Use"
    How agents can go beyond text generation to actually read files, run commands, and interact with the world.

??? example "Event-Driven Design"
    How to build systems that notify other parts of the program when things happen — without tight coupling.

---

## Milestones

::: {.grid }

### :fontawesome-solid-circle-1:{ .lg .middle } Project Setup & Hello Agent

Set up your project, install dependencies, and create a CLI that prints "Hello agent."

[:octicons-arrow-right-24: Start](milestones/01-setup-and-cli/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/01-setup-and-cli/context.md){: .md-button }

---

### :fontawesome-solid-circle-2:{ .lg .middle } Messages & Agent State

Build a message system and an `Agent` class that manages conversation context.

[:octicons-arrow-right-24: Start](milestones/02-messages-and-agent/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/02-messages-and-agent/context.md){: .md-button }

---

### :fontawesome-solid-circle-3:{ .lg .middle } The Agent Loop

Connect everything with a loop: prompt the LLM, get a response, detect tool calls, repeat.

[:octicons-arrow-right-24: Start](milestones/03-agent-loop/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/03-agent-loop/context.md){: .md-button }

---

### :fontawesome-solid-circle-4:{ .lg .middle } Tool Execution

Register tools, detect tool calls from the LLM, validate arguments, and feed results back into the conversation.

[:octicons-arrow-right-24: Start](milestones/04-tool-execution/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/04-tool-execution/context.md){: .md-button }

---

### :fontawesome-solid-circle-5:{ .lg .middle } Real LLM Calls

Replace the mock responses with real API calls to OpenAI. Your agent is now alive.

[:octicons-arrow-right-24: Start](milestones/05-real-llm-calls/README.md){: .md-button .md-button--primary }
[:material-book-open-variant: Overview](milestones/05-real-llm-calls/context.md){: .md-button }

:::

---

## Before You Start

1. Read the [:material-lightbulb-outline: Key Concepts](./key-concepts.md) to understand what you're learning
2. Skim the [:material-book-alphabet: Glossary](./glossary.md) for terms you might not know
3. Check the [:simple-web: Reading List](./reading-list.md) if you want deeper context

??? warning "Important"
    Each milestone builds on the last. You'll have something runnable at every step — **don't skip ahead**. The instructions give you requirements and hints, but **you write the code**. If you're stuck, re-read the context before looking for answers online.

??? note "Compared to the full course"
    This is a simplified version of [pi-mvp](../pi-mvp/README.md). You'll skip advanced topics like streaming, generics, Zod-based validation, and multiple provider support. The full course adds 3 more milestones on top of these foundations.
