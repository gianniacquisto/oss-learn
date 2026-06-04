# Context for Milestone 2: Messages & Agent State

## Background

The message system is the backbone of any agent harness. Every piece of information — user input, assistant responses, tool results, custom notifications — flows through this system. The key design challenge: how do you support different types of messages while keeping the code simple and type-safe?

In pi.dev, messages are represented as a union type — each message has a `role` field that tells you what kind of message it is (user, assistant, system, or tool result). The agent class manages these messages as a conversation history.

## How This Fits in the Bigger Picture

This milestone defines the data structures that Milestone 3 (the loop) will operate on. The Agent class you build here becomes the stateful wrapper around the loop — it owns the conversation transcript, manages tool registration, and tracks lifecycle events. When you get to Milestone 4, the same message types will be converted into the format each LLM provider expects.

## Key Decisions You'll Make

- **Array copying on access:** Should `state.messages` return a reference or a copy? Returning a copy prevents callers from mutating internal state accidentally, but costs memory. Pi copies on assignment (set) and returns references on read — this is a performance optimization that works because the Agent itself controls mutations.
- **Tool storage:** Should tools be stored as an array or a map? An array preserves insertion order; a map gives faster lookups. For a small number of tools, an array is fine.

## What "Good" Looks Like

By the end of this milestone:
- You have message types defined in `types.ts`
- The `Agent` class has private state with readonly public access
- Running `npm start` prints the agent's state as JSON
- You can create an agent, add tools to it, and reset its conversation
