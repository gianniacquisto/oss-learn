# Context for Milestone 2: Message Types & Agent State

## Background

The message system is the backbone of any agent harness. Every piece of information — user input, assistant responses, tool results, custom notifications — flows through this system. The key design challenge: how do you support both LLM-standard messages and application-specific messages while maintaining type safety?

Pi.dev solves this with a union type (`AgentMessage = StandardMessage | CustomMessage`) and uses TypeScript's declaration merging to let applications extend the custom message types without modifying the base library.

## How This Fits in the Bigger Picture

This milestone defines the data structures that Milestone 3 (the loop) will operate on. The Agent class you build here becomes the stateful wrapper around the loop — it owns the conversation transcript, manages tool registration, and tracks lifecycle events. When you get to Milestone 5, the same message types will be converted into the format each LLM provider expects.

## Key Decisions You'll Make

- **Array copying on access:** Should `state.messages` return a reference or a copy? Returning a copy prevents callers from mutating internal state accidentally, but costs memory. Pi copies on assignment (set) and returns references on read — this is a performance optimization that works because the Agent itself controls mutations.
- **Tool storage:** Should tools be stored as an array or a map? A map gives O(1) lookup by name; an array preserves insertion order. Pi uses an array but does linear search (the number of tools is typically small).
