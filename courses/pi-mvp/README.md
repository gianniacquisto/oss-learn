# pi-mvp: Build Your Own Agent Harness

Build your own minimal agent harness — the core engine that powers [pi.dev](https://github.com/earendil-works/pi). This is not a chatbot tutorial; it's an exercise in designing the architecture of an AI agent system from scratch.

## What You'll Build

A working agent harness that can:
- Accept user prompts and stream responses back
- Call an LLM (OpenAI or Anthropic) to generate assistant messages
- Execute tool calls the model requests (e.g., read files, run commands)
- Loop until the conversation is complete
- Emit lifecycle events for UI integration

You'll end up with something that can have a multi-turn conversation with an LLM and actually *do things* based on what it learns.

## Difficulty: Intermediate

## Time Estimate: 8–12 hours

## Prerequisites

- **TypeScript basics** — types, interfaces, generics, modules
- **Node.js and npm** — running scripts, managing packages
- **Async/await and promises** — you should be comfortable with async flows
- **Basic LLM API knowledge** — know what a chat completion request looks like (you'll implement one)

## What You'll Learn

- How to design a stateful agent that manages conversation context across turns
- The agent loop pattern: prompt → LLM call → tool execution → repeat
- Tool registration and execution with type-safe schemas
- Event-driven architecture for decoupling the core engine from UI concerns
- Streaming responses and real-time event emission

## Milestones

| # | Milestone | What You'll Have |
|---|-----------|------------------|
| 1 | Project setup & config | A working TypeScript project with a minimal CLI that prints "Hello agent" |
| 2 | Message types & agent state | Type-safe message system and an Agent class managing conversation context |
| 3 | Core agent loop | The main loop: prompt the LLM, get a response, detect if it needs tools |
| 4 | Tool execution system | Register tools, execute them safely, feed results back to the model |
| 5 | LLM integration & streaming | Full multi-turn conversation with real API calls and event streaming |

## Before You Start

1. Read the [key concepts](./key-concepts.md) to understand what you're learning
2. Skim the [glossary](./glossary.md) for terms you might not know
3. Check the [reading list](./reading-list.md) if you want deeper context on any topic

## How This Course Works

Each milestone builds on the last. You'll have something runnable at every step — don't skip ahead. The instructions give you requirements and hints, but **you write the code**. If you're stuck, re-read the context before looking for answers online.

Verification criteria in each `done.md` tell you exactly how to check your work.
