# Context for Milestone 4: Tool Execution System

## Background

Tools are the bridge between the agent's reasoning and the real world. When an LLM decides it needs to read a file, run a command, or query a database, it does so by emitting a tool call — a structured request that says "execute function X with these arguments." The agent harness is responsible for:

1. **Registering** tools (making them discoverable)
2. **Detecting** tool calls in LLM responses
3. **Validating** arguments against the tool's schema
4. **Executing** the tool and capturing results
5. **Formatting** results back into a message the LLM can understand

## How This Fits in the Bigger Picture

This milestone extends your loop from Milestone 3. The loop structure doesn't change much — you just add a step between "get response" and "check if done": "if there are tool calls, execute them and feed results back." The key insight: after executing tools, you go back to the LLM with the tool results appended to context. This is what makes agents iterative — they reason → act → observe → reason again.

## Key Decisions You'll Make

- **Sequential vs. parallel execution:** Should multiple tool calls from a single response execute one at a time or concurrently? Pi supports both modes (configurable per-tool). Sequential is simpler and safer; parallel is faster but requires careful error handling.
- **Error handling in tools:** Should tools throw exceptions or return `{ isError: true, content: [...] }`? Pi uses the latter — it lets the loop continue processing even if one tool fails, rather than aborting the entire batch.
- **Schema validation:** Do you validate eagerly (before calling execute) or lazily (inside execute)? Eager validation gives better error messages ("argument X is required" vs. "undefined is not a function") but adds overhead. Pi validates eagerly.
