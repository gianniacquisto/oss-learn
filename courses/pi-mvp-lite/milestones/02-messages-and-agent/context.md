# Context for Milestone 2: Messages & Agent State

## Background

Every conversation with an LLM consists of a series of messages, each with a specific role. But those messages aren't just "who said what" — they carry different structures and serve different purposes in the agent's reasoning process.

This milestone defines the data shapes that everything else depends on. The types you create here are referenced by the agent class (this milestone), the loop (Milestone 3), tool execution (Milestone 4), and LLM integration (Milestone 5). Getting them right now prevents a cascade of refactoring later.

### Why Different Message Roles?

When you talk to ChatGPT in a browser, it feels like one conversation. But behind the scenes, the API sees a structured list of messages where each one has a specific role:

| Role | Who Creates It | What It Contains | Why It Exists |
|------|---------------|-----------------|---------------|
| `system` | You (the developer) | Instructions for the LLM's behavior | Sets the agent's "personality" and rules — always sent first |
| `user` | The end user | Text input from the person using the agent | The actual question or command the user is asking |
| `assistant` | The LLM | Generated text (or tool calls) | What the model decided to say or do |
| `toolResult` | Your code | Output from running a tool | Feeds real-world data back to the LLM so it can reason about it |

The LLM uses these roles to understand context: system messages set rules, user messages are what to respond to, assistant messages are its own prior responses, and tool results are observations from the world. Without these distinctions, the LLM wouldn't know whether a message is an instruction, a question, or data.

### Why an Agent Class (and Not Just Functions)?

You *could* manage conversation state with plain functions that pass arrays around. But as soon as you add tools, lifecycle tracking ("is the agent busy?"), error state, and reset capability, the pattern becomes messy. The `Agent` class bundles all of this into one object:

- **State ownership:** Only the Agent modifies its own messages — callers can't accidentally delete half the conversation
- **Lifecycle tracking:** `isStreaming` prevents two prompts from running simultaneously (which would corrupt state)
- **Extensibility:** New capabilities (tool registration, event emission) are added as methods on the same class

### The Encapsulation Pattern in Detail

```typescript
class Agent {
  private _state: MutableAgentState;  // only the class can modify this

  // Readonly — callers can read but not write
  get state(): AgentState {
    return { ...this._state };
  }

  // Explicit setters for fields that *should* be changeable from outside
  set tools(tools: Tool[]) {
    this._state.tools = tools;
  }

  // Methods that modify internal state in controlled ways
  reset(): void {
    this._state.messages = [];
    this._state.errorMessage = undefined;
  }
}
```

The spread operator (`{ ...this._state }`) creates a shallow copy, so callers can't mutate the original by assigning to properties of the returned object. This is a deliberate design choice that prevents subtle bugs where one part of the code modifies state without the Agent's knowledge.

## How This Fits in the Bigger Picture

```text
                    types.ts (this milestone)
                   ┌───────────────────┐
                   │ UserMessage       │
                   │ AssistantMessage  │
                   │ ToolResultMessage │◀── loop.ts uses these
                   │ SystemMessage     │    to build conversation
                   │ AgentMessage (∪)  │    context
                   │ Tool              │◀── runLoop needs these
                   └───────────────────┘
                           ▲
                           │ references
                   ┌───────┴───────┐
                   │   agent.ts    │◀── owns the list of messages,
                   │  (Agent class)│    provides prompt()/reset()
                   └───────┬───────┘
                           │ used by
                   ┌───────┴───────┐
                   │   cli.ts      │◀── creates Agent, calls prompt()
                   └───────────────┘
```

Every file that interacts with conversation data imports from `types.ts`. Every file that manages the agent's lifecycle imports from `agent.ts`. This is the dependency chain — and it flows one direction, which keeps things maintainable.

## Key Decisions You'll Make

- **Content as an array of blocks vs. a string:** We use `content: TextContent[]` (an array) instead of `content: string` because a single message can contain both text and tool calls in later milestones. Starting with the array shape avoids refactoring later.
- **Timestamps on every message:** Every message gets a `timestamp: number` (from `Date.now()`). This isn't required by the LLM API, but it's invaluable for debugging ("which message came first?") and sorting.
- **Copy-on-read vs. reference-on-read:** Should `state` return a copy or the actual reference? We return a spread copy (`{ ...this._state }`) — it's slightly less performant but much safer. Callers can't accidentally mutate internal state.

## What "Good" Looks Like

By the end of this milestone:
- ✅ All message types compile without errors and have the correct `role` discriminants
- ✅ The `Agent` class stores state privately and exposes it read-only
- ✅ You can create an agent, set its model, and print its state as JSON
- ✅ `agent.reset()` clears messages but preserves system prompt and model settings
- ✅ TypeScript prevents callers from doing `agent.state.messages = []` (readonly enforcement)
