---
description: The core ideas behind AI agent architecture — explained for beginners.
---

# :material-lightbulb-outline: Key Concepts

These are the big ideas behind how AI agents work. You'll encounter each one as you build through the milestones.

---

## 1. The Agent Loop

??? tip "Quick Summary"
    An agent isn't a single function call — it's a **loop**. The model generates text, which may include tool calls. Those tools execute and produce results. The results go back to the model, which decides what to do next. This repeats until the model says it's done.

**What it is:** An agent loop works like this:

1. You give the agent a prompt (a question or task)
2. The agent sends the prompt to an LLM (like GPT)
3. The LLM responds with text — or it might say "I need to call a tool"
4. If there's a tool call, the agent runs the tool and sends the result back to the LLM
5. The LLM responds again, and steps 3-4 repeat until the LLM says "I'm done"

**Why it matters:** This is the fundamental pattern behind every AI agent — from simple chatbots to complex autonomous systems. Understanding this loop is understanding how agents actually work.

### How It Shows Up in This Course

You'll build this in **Milestone 3**. The loop has two layers — an outer loop that handles multi-turn conversations, and an inner loop that processes tool calls from a single response.

```text
You: "Read the README file and tell me what it says"
  ↓
Agent Loop starts
  ├─ Inner loop: LLM says "I'll use read_file"
  │   └─ Agent runs read_file("README.md") → gets file content
  │   └─ Agent sends file content back to LLM
  ├─ Inner loop: LLM says "I'm done"
  │   └─ Agent returns the final answer to you
  ↓
You see the agent's response
```

---

## 2. Stateful vs. Stateless

??? tip "Quick Summary"
    A regular API call is **stateless** — you send a request, get a response, and that's it. An agent is **stateful** — it remembers what happened in previous messages.

**What it is:** When you call an LLM API directly, you send a list of messages and get a response. The API doesn't remember anything about you between calls. An agent, on the other hand, **keeps track** of the conversation — it stores all the messages, knows which tools are available, and decides what to do next based on what's happened so far.

**Why it matters:** This is what turns a simple API call into an "agent." The agent owns the conversation history, manages the tools, and makes decisions about the flow. You're building the brain, not just the mouth.

### How It Shows Up in This Course

**Milestone 2** introduces the `Agent` class with its internal state — the list of messages, the available tools, and whether it's currently processing something. This is the "brain" of your agent.

---

## 3. Message Types

??? tip "Quick Summary"
    Conversations aren't just "user says something, assistant replies." There are different **roles** — user, assistant, system, and tool results — each carrying different information.

**What it is:** In an agent conversation, different messages serve different purposes:

- **User messages** — what you type
- **Assistant messages** — what the LLM generates (text or tool calls)
- **System messages** — instructions that shape the LLM's behavior
- **Tool result messages** — the output from running a tool

**Why it matters:** Each type of message carries different information and is handled differently. The agent needs to know which is which so it can process the conversation correctly.

### How It Shows Up in This Course

**Milestone 2** defines these as TypeScript types. You'll create a `UserMessage`, `AssistantMessage`, `SystemMessage`, and `ToolResultMessage` — each with its own structure.

---

## 4. Tool Use

??? tip "Quick Summary"
    Tools let agents **take action** — not just generate text, but read files, run commands, query databases, and interact with the world.

**What it is:** A tool is a function that the agent can call. It has:

- A **name** (like `read_file`)
- A **description** (telling the LLM when to use it)
- **Parameters** (what inputs it needs)
- An **execute function** (the actual code that runs)

When the LLM decides a tool would help, it tells the agent "call `read_file` with path='README.md'". The agent runs the tool and sends the result back to the LLM.

**Why it matters:** This is what makes agents *agentic*. Without tools, an agent is just a fancy text generator. With tools, it can actually do things.

### How It Shows Up in This Course

**Milestone 4** introduces tools. You'll define a tool interface, create a simple tool (like reading a file), and wire it into the agent loop.

---

## 5. Events

??? tip "Quick Summary"
    Instead of returning one result, the agent **sends notifications** as it progresses. Other parts of the program can listen to these notifications and react independently.

**What it is:** An event is a notification that something happened. For example:

- `agent_start` — the agent began processing
- `message_update` — the LLM generated some new text
- `agent_end` — the agent finished

Other parts of your program (like a UI) can "subscribe" to these events and update themselves when they arrive.

**Why it matters:** This decouples the agent from whatever displays its output. A CLI can print events as text, a web UI can show them as rich components — all without changing the agent code.

### How It Shows Up in This Course

**Milestone 3** introduces a simple event system. You'll define event types and a way for other code to receive them.

---

## What Makes This Interesting

Studying agent architecture teaches you about:

- **System design** — how to connect multiple moving parts (LLM API, tools, state management) into a coherent system
- **Async programming** — working with code that runs over time (API calls, streaming, concurrent operations)
- **Abstraction** — creating interfaces that let different parts of a system work together without knowing each other's details

These skills transfer to building any complex application — not just AI agents.
